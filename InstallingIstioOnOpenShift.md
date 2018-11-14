# Deploying Istio on OpenShift

Here we will learn the administrative steps to deploy Istio on OpenShift and set up the Istio cluster for running the examples to follow in a multi user environment. The instructions listed here are based on the official documentation [here](https://docs.openshift.com/container-platform/3.10/servicemesh-install/servicemesh-install.html).

>**Note** Istio on OpenShift is still Tech Preview. So, these labs are to learn how things are shaping up. Technology Preview releases are not supported with Red Hat production service-level agreements (SLAs) and might not be functionally complete, and Red Hat does NOT recommend using them for production. 

These steps have been tested for `OCP 3.11.16`.

## Prerequisites
* A Running OpenShift Cluster that is deployed either using `ovs-subnet` or `ovs-networkpolicy` plugins
* You need administrative access to the cluster to perform the tasks listed in this chapter


## Preparing the Cluster

### Apply Patch to Master(s)

Let us apply the following patch to the master(s) on the master configuration file `/etc/origin/master/master-config.yaml` to enable `MutatingAdmissionWebhook` and `ValidatingAdmissionWebhook`.

```
admissionConfig:
  pluginConfig:
    MutatingAdmissionWebhook:
      configuration:
        apiVersion: apiserver.config.k8s.io/v1alpha1
        kubeConfigFile: /dev/null
        kind: WebhookAdmission
    ValidatingAdmissionWebhook:
      configuration:
        apiVersion: apiserver.config.k8s.io/v1alpha1
        kubeConfigFile: /dev/null
        kind: WebhookAdmission
```

You can download the patch first, take a back up and use `oc ex config patch` using the patch and the backup files to apply the patch as shown below:

```
mkdir istio-install
cd istio-install
wget https://raw.githubusercontent.com/Maistra/openshift-ansible/maistra-0.4/istio/master-config.patch
cp -p /etc/origin/master/master-config.yaml /etc/origin/master/master-config.yaml.prepatch
oc ex config patch /etc/origin/master/master-config.yaml.prepatch -p "$(cat ./master-config.patch)" > /etc/origin/master/master-config.yaml
```

>**Note** If you have multiple masters, do this on all masters.

Ensure your master-config file shows the webhooks as shown below

```
# cat /etc/origin/master/master-config.yaml
admissionConfig:
  pluginConfig:
    ....
    ....
    MutatingAdmissionWebhook:
      configuration:
        apiVersion: apiserver.config.k8s.io/v1alpha1
        kind: WebhookAdmission
        kubeConfigFile: /dev/null
    ValidatingAdmissionWebhook:
      configuration:
        apiVersion: apiserver.config.k8s.io/v1alpha1
        kind: WebhookAdmission
        kubeConfigFile: /dev/null
    ....
    ....
    
```

Now restart the master(s) by running:

```
/usr/local/bin/master-restart api
/usr/local/bin/master-restart controllers
```

### Updating Node Configuration

To run the Elasticsearch application, you must make a change to the kernel configuration on each node. On every node

* Create a file named `/etc/sysctl.d/99-elasticsearch.conf` with `vm.max_map_count = 262144`
* Run the command `sysctl vm.max_map_count=262144`

Let us accomplish these tasks using ansible. Create a list of all nodes in a file as shown below. You can as well use `/etc/ansible/hosts` file if you already have this list there.

```
# cat hostnames.txt 
[nodes]
master.us-west-2.compute.internal
node1.us-west-2.compute.internal
node2.us-west-2.compute.internal
node3.us-west-2.compute.internal
node4.us-west-2.compute.internal
```

Run the commands across all nodes to update configurations

```
ansible nodes -i hostnames.txt -m shell -a "echo 'vm.max_map_count = 262144' > /etc/sysctl.d/99-elasticsearch.conf"
ansible nodes -i hostnames.txt -m shell -a "sysctl vm.max_map_count=262144"
```

## Installing Service Mesh

Installing Istio on OpenShift involves creating a custom resource definition file, then installing mesh an using operator to create and manage the custom resource. 

> **Note**  As of the Tech Preview, there is a cyclic dependency between the CRD and Operator, wherein CRD depends on the Operator to create a custom resource definition for `installations.istio.openshift.com` and Operator depends on `istio-installation` CRD. The following the steps have been tried successfully in spite of such dependencies.

### Create the CRD file

Create a file with name `istio_installation.yaml` with the following contents. This should be sufficient for the purpose of our workshop. [OpenShift Istio documentation](https://docs.openshift.com/container-platform/3.10/servicemesh-install/servicemesh-install.html#creating-custom-resource) has some additional features such as Fabric8 launcher with Istio. 

This configuration instructs the operator 
* to install istio on an OCP cluster (`deployment_type: openshift`)
* the version of istio is tech preview
* install jaeger 1.7.0 for tracing and to use elasticsearch with `1Gi` memory
* install kiali 0.7.0 for observability and the credentials to log onto kiali are `admin/admin`. You may want to change these to values of your choice.

```
# cat istio-installation.yaml 
apiVersion: "istio.openshift.com/v1alpha1"
kind: "Installation"
metadata:
  name: "istio-installation"
spec:
  deployment_type: openshift
  istio:
    authentication: true 
    community: false
    prefix: openshift-istio-tech-preview/
    version: 0.3.0
  jaeger:
    prefix: distributed-tracing-tech-preview/
    version: 1.7.0
    elasticsearch_memory: 1Gi
  kiali:
    username: admin    
    password: admin 
    prefix: openshift-istio-tech-preview/
    version: 0.8.1
```
This file is also available [here](./istio_installation.yaml), if you want to directly use it.


### Start Installation

Create a new project with name `istio-operator` and invoke the operator template by passing your own value for `OPENSHIFT_ISTIO_MASTER_PUBLIC_URL`, which should be the master's public URL. 

> **Remember** to edit the Master URL value before you run these commands

```
oc new-project istio-operator
oc new-app -f https://raw.githubusercontent.com/Maistra/openshift-ansible/maistra-0.3/istio/istio_product_operator_template.yaml --param OPENSHIFT_ISTIO_MASTER_PUBLIC_URL=master.devday.ocpcloud.com --param OPENSHIFT_RELEASE=v3.11.16
```

This will start an istio-operator pod that starts the installation. 

```
# oc get po -n istio-operator
NAME                              READY     STATUS    RESTARTS   AGE
istio-operator-7b4985d56b-d4r5l   1/1       Running   0          22h
```

If you check the logs by running `oc logs istio-operator-7b4985d56b-d4r5l` it will be waiting for the CRD.

Now create the CRD by running. 

```
oc create -f istio-installation.yaml 
```

Watch the logs for istio-operator pod and you will notice the installation proceeding as follows

```
# oc logs -f istio-operator-7b4985d56b-d4r5l -n istio-operator
time="2018-10-17T15:29:37Z" level=info msg="Go Version: go1.9.4"
time="2018-10-17T15:29:37Z" level=info msg="Go OS/Arch: linux/amd64"
time="2018-10-17T15:29:37Z" level=info msg="operator-sdk Version: 0.0.5+git"
time="2018-10-17T15:29:37Z" level=info msg="Metrics service istio-operator created"
time="2018-10-17T15:29:37Z" level=info msg="Watching resource istio.openshift.com/v1alpha1, kind Installation, namespace istio-operator, resyncPeriod 0"
time="2018-10-17T15:29:38Z" level=info msg="Installing istio for Installation istio-installation"
```

Watch the pods in the `istio-system` project and you will notice the following. The number of pods may vary based on the number of nodes you have. Make sure the pods are in `Running` status and the istio-installer pod in `Completed` status.

```
NAME                                          READY     STATUS      RESTARTS   AGE
elasticsearch-0                               1/1       Running     0          2m
grafana-6d5c5477-k7wrh                        1/1       Running     0          2m
istio-citadel-6f9c778bb6-q9tg9                1/1       Running     0          3m
istio-egressgateway-957857444-2g84h           1/1       Running     0          3m
istio-galley-c47f5dffc-dm27s                  1/1       Running     0          3m
istio-ingressgateway-7db86747b7-s2dv9         1/1       Running     0          3m
istio-pilot-5646d7786b-rh54p                  2/2       Running     0          3m
istio-policy-7d694596c6-pfdzt                 2/2       Running     0          3m
istio-sidecar-injector-57466d9bb-4cjrs        1/1       Running     0          3m
istio-statsd-prom-bridge-7f44bb5ddb-6vx7n     1/1       Running     0          3m
istio-telemetry-7cf7b4b77c-p8m2k              2/2       Running     0          3m
jaeger-agent-5mswn                            1/1       Running     0          2m
jaeger-collector-9c9f8bc66-j7kjv              1/1       Running     0          2m
jaeger-query-fdc6dcd74-99pnx                  1/1       Running     0          2m
openshift-ansible-istio-installer-job-f8n9g   0/1       Completed   0          7m
prometheus-84bd4b9796-2vcpc                   1/1       Running     0          3m
kiali-76b6f689f6-ghj74                        1/1       Running     0          3m
```


### Cleanup

**If you need to uninstall Istio** from the OpenShift cluster, the following can be done:

* delete the istio-operator tempalte
* delete the CRDs `installation` and `istio-installation` from the project
* delete the two project `istio-operator` and `istio-system`

```
oc process -n istio-operator -f https://raw.githubusercontent.com/Maistra/openshift-ansible/maistra-0.3/istio/istio_product_operator_template.yaml | oc delete -f -

oc delete -n istio-operator installation istio-installation
oc delete project istio-operator
oc delete project istio-system 
```

## Preparing Istio Cluster for a Multi-user Workshop

### Change OpenShift Router to handle Wildcard routes

We'll configure OpenShift router to use Wildcard routes following the [documentation here](https://docs.openshift.com/container-platform/3.10/install_config/router/default_haproxy_router.html#using-wildcard-routes). This way we can expose a wildcard route for the `istio-ingressgateway` service. Once you have wildcard route for istio-ingressgateway, we can extend that route for each application to have a unique route.

Run the following command to enable wildcard routes on the router in the `default` project

```
oc set env dc/router ROUTER_ALLOW_WILDCARD_ROUTES=true
```
The above command will edit the deployment configuration for the router and redeploy the router pods. Once the router pods are up and running, move forward.

### Expose a Wildcard Route for Istio Ingress Gateway

If you check the routes in the `istio-system` project, istio-installer should have already created an openshift route for the service `istio-ingressgateway`.

```
# oc get route istio-ingressgateway -n istio-system
NAME                   HOST/PORT                                                    PATH      SERVICES               PORT      TERMINATION   WILDCARD
istio-ingressgateway   istio-ingressgateway-istio-system.apps.devday.ocpcloud.com             istio-ingressgateway   http2                   None
```
This is a single entry point for all your application traffic running on Istio. If we want to run multiple applications we want to access them with different URLs.

One way to accomplish that currently is to create a wildcard route for `istio-ingressgateway`. Since we enabled wildcard routes on our OpenShift cluster, let us expose a wildcard route for `istio-ingressgateway` service.

>**Note** substitute your own domain name for the `hostname` parameter. In my case `apps.devday.ocpcloud.com` is my openshift default domain. I have added `istio` to qualify all the istio applications. 

```
oc expose svc istio-ingressgateway --hostname="www.istio.apps.devday.ocpcloud.com" --port=http2 --name=istio-wildcard-ingress --wildcard-policy=Subdomain
```


This creates a route with the following configuration:

```
# oc get route istio-wildcard-ingress -o yaml -n istio-system
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  creationTimestamp: 2018-10-18T00:29:43Z
  labels:
    app: istio-ingressgateway
    chart: gateways-1.0.1
    heritage: Tiller
    istio: ingressgateway
    maistra-version: 0.2.0
    release: istio-1.0.2
  name: istio-wildcard-ingress
  namespace: istio-system
  resourceVersion: "4494685"
  selfLink: /apis/route.openshift.io/v1/namespaces/istio-system/routes/istio-wildcard-ingress
  uid: e89fc12a-d26c-11e8-900a-069c4762f7ea
spec:
  host: www.istio.apps.devday.ocpcloud.com
  port:
    targetPort: http2
  to:
    kind: Service
    name: istio-ingressgateway
    weight: 100
  wildcardPolicy: Subdomain
status:
  ingress:
  - conditions:
    - lastTransitionTime: 2018-10-18T00:29:43Z
      status: "True"
      type: Admitted
    host: www.istio.apps.devday.ocpcloud.com
    routerName: router
    wildcardPolicy: Subdomain
```

The `wildcardPolicy: Subdomain` in this configuration allows us to create specific hostnames for application such as `bookinfo1.istio.apps.devday.ocpcloud.com`, `bookinfo2.istio.apps.devday.ocpcloud.com` etc. **Of course, you will have your own domain names substituted in this example.**

When we deploy an application later, we will learn how to configure application specific hostnames into the application gateway and virtualservice.

### Additional access to the users 

As of now, each user that needs to run Istio examples need `view` access to the `istio-system` project. You can provide such access by running the following command for each user:

```
oc adm policy add-role-to-user view user1 -n istio-system
```
> **Note** If you are enabling this cluster for a workshop with a bunch of users, run the above command for all the user-ids created for the workshop.

### Additional access to the project

Applications are deployed in to projects/namespaces. On OpenShift the applications running in a namespace run with a `default` service account. This `default` service account runs with `restricted` SCC, which prevents it from running containers as specific user-ids or root, and also has restrictions on the linux capabilities. 

Istio requires specific kinds of access at the project level:

* Project needs to be labeled as `istio-injection=enabled` to let Istio know that this project can be used to enable automatic injection of side cars
* As of now, the `default` service account need to be elevated to `privileged` SCC, so that it can allow the application pods to have init containers whose `proxy_init` runs in privileged mode and adds `NET_ADMIN`
as shown here. You will find this configuration in the individual `deployment` artifacts when you deploy the application.

```
      initContainers:
      - args:
        - -p
        - "15001"
        - -u
        - "1337"
        - -m
        - REDIRECT
        - -i
        - '*'
        - -x
        - ""
        - -b
        - 9080,
        - -d
        - ""
        image: docker.io/istio/proxy_init:1.0.2
        imagePullPolicy: IfNotPresent
        name: istio-init
        resources: {}
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
          privileged: true
```

Let's create a project named `bookinfo1` for a workshop user named `user1`, label this project for istio-injection, and make `user1` the project administrator. If there are multiple workshop users, you would repeat these for every user. **Example:** Project `bookinfo2` for `user2` and so on. Make a note of the projects created for each user and let them know.

```
oc new-project bookinfo1
oc adm policy add-scc-to-user privileged -z default -n bookinfo1
oc adm policy add-role-to-user admin user1 -n bookinfo1
oc label namespace bookinfo1 istio-injection=enabled
```

Now, if a user logs in as `user1`, they will see both `bookinfo1` and `istio-system` in their list.


## Summary

In this chapter we learnt to perform the following administrative tasks on an OpenShift cluster:

* Preparing an OpenShift cluster to install Istio
* Installed Istio
* Set up Wildcard Router and a Wildcard route for istio-ingressgateway
* Enabled user(s) to run applications on Istio
* Created a project(s) with necessary privileges for end users to use 

