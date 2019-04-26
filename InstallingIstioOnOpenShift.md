# Deploying Istio on OpenShift

Here we will learn the administrative steps to deploy Istio on OpenShift and set up the Istio cluster for running the examples to follow in a multi user environment. The instructions listed here are based on the official documentation [here](https://docs.openshift.com/container-platform/3.11/servicemesh-install/servicemesh-install.html).

>**Note** Istio on OpenShift is still Tech Preview. So, these labs are to learn how things are shaping up. Technology Preview releases are not supported with Red Hat production service-level agreements (SLAs) and might not be functionally complete, and Red Hat does NOT recommend using them for production. 

These steps have been tested for `openshift v3.11.88`.

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

**Note:** If you restart the nodes, the setting `sysctl vm.max_map_count=262144` goes away. You may want to check and run the command again. 

## Installing Service Mesh

Istio installation is handled by an operator on OpenShift. Istio operator runs as a pod and it watches custom resource (CR) of kind `controlplane`. If a custom resource of kind `controlplane` is created, the operator follows the configurated provided in that CR and creates a control plane accordingly.

### Installing Istio Operator

As a cluster administrator run the following

* Create two projects to run operator and the other one for istio control plane

```
oc new-project istio-operator
oc new-project istio-system
```

Review the template [https://raw.githubusercontent.com/Maistra/istio-operator/maistra-0.10/deploy/servicemesh-operator.yaml](https://raw.githubusercontent.com/Maistra/istio-operator/maistra-0.10/deploy/servicemesh-operator.yaml)

This template adds
* Custom Resource Definitions for `installation` and `controlplane`
* Adds a cluster role, and service account for `istio-operator` and creates a role-binding
* Creates a deployment for `istio-operator`. This deployment results in running an `istio-operator` pod that watches the CRs for the above CRDs.

Apply this template to create custom resource definitions and to run the operator.

```
# oc apply -n istio-operator -f https://raw.githubusercontent.com/Maistra/istio-operator/maistra-0.10/deploy/servicemesh-operator.yaml


customresourcedefinition.apiextensions.k8s.io/installations.istio.openshift.com created
customresourcedefinition.apiextensions.k8s.io/controlplanes.istio.openshift.com created
clusterrole.rbac.authorization.k8s.io/istio-operator created
serviceaccount/istio-operator created
clusterrolebinding.rbac.authorization.k8s.io/istio-operator-account-istio-operator-cluster-role-binding created
deployment.apps/istio-operator created
```

Verify that the operator is running

```
# oc get po -n istio-operator
NAME                              READY     STATUS    RESTARTS   AGE
istio-operator-79c5b4d68f-gvglw   1/1       Running   0          1h
```

Also verify the CRDS created

```
# oc get crd installations.istio.openshift.com
NAME                                CREATED AT
installations.istio.openshift.com   2019-04-25T23:25:51Z

# oc get crd controlplanes.istio.openshift.com
NAME                                CREATED AT
controlplanes.istio.openshift.com   2019-04-25T23:25:51Z
```

So if you create a custom resource of kind controlplane, the operator is now ready to act on it.


### Create Custom Resource for Istio Installation

A basic istio control plane installation is provided by the custom resource. Review this custom resource [https://raw.githubusercontent.com/Maistra/istio-operator/maistra-0.10/deploy/examples/istio_v1alpha3_controlplane_cr_basic.yaml](https://raw.githubusercontent.com/Maistra/istio-operator/maistra-0.10/deploy/examples/istio_v1alpha3_controlplane_cr_basic.yaml)

Note the kind is `ControlPlane` (the CRD that was created earlier) and the CR is named `basic-install`

```
kind: ControlPlane
metadata:
  name: basic-install
```
Notice that the CR provides configurations for various istio control plane components installed by the operator.

We will make a small change to this CR so let us first download this CR 

```
wget  -O istio-installation.yaml  https://raw.githubusercontent.com/Maistra/istio-operator/maistra-0.10/deploy/examples/istio_v1alpha3_controlplane_cr_basic.yaml
```

and make a change to enable automatic Istio openshift route creation by changing `ior_enabled` to `true`. This change will start an ior operator that automatically creates a new openshift route for each application you deploy with a specific host so that we have an ingress for our application. If you don't understand this, it's OK. We will discuss more on this later. Run the following command to make the above change to the CR. 

```
sed -ibak -e "s/ior_enabled: false/ior_enabled: true/" istio-installation.yaml
```

You can `cat istio-installation.yaml` to verify the change is effective.


Create the CR by running

```
# oc create -f istio-installation.yaml 

controlplane.istio.openshift.com/basic-install created
```
Once the CR is created, operator starts installing the istio control plane components. If you watch `istio-system` project running `watch oc get po -n istio-system` eventually you will see the following pods running

```
watch oc get po -n istio-system

NAME                                      READY     STATUS    RESTARTS   AGE
elasticsearch-0                           1/1       Running   0          2h
grafana-6c5dfdf5bd-pphkp                  1/1       Running   0          2h
ior-69cbb8b7f5-ch6v6                      1/1       Running   0          8m
istio-citadel-66cf447cbd-9vzlw            1/1       Running   0          2h
istio-egressgateway-69b65dddf5-mpkbd      1/1       Running   0          2h
istio-galley-5dbd58568d-g7bkl       	  1/1       Running   0          2h
istio-ingressgateway-b688c9d9b-st44c      1/1       Running   0          2h
istio-pilot-79668d4bf6-ppp5n        	  2/2       Running   0          2h
istio-policy-5f45fcf95f-zn72c             2/2       Running   0          2h
istio-sidecar-injector-7c44bcbbcd-rshvg   1/1       Running   0          2h
istio-telemetry-7fcd854d6b-2q7wd          2/2       Running   0          2h
jaeger-agent-cvgk9                        1/1       Running   0          2h
jaeger-agent-kjhpb                        1/1       Running   0          2h
jaeger-agent-kwfxh                        1/1       Running   0          2h
jaeger-agent-vdv92                        1/1       Running   0          2h
jaeger-agent-zw7hm                        1/1       Running   0          2h
jaeger-collector-576b66f88c-qrzrt         1/1       Running   2          2h
jaeger-query-7549b87c55-cfvfv             1/1       Running   2          2h
kiali-7475849854-cfxsq                    1/1       Running   0          2h
prometheus-5dfcf8dcf9-l9xgp               1/1       Running   0          2h
```

### Cleanup

To **uninstall Istio Control Plane**, we just need to delete the custom resource and the operator will handle the clean up.

```
oc delete controlplane basic-install 
```

Verify that the controlplane custom resource is deleted and the operator has deleted the pods related to control plane in the `istio-system` project.

```
# oc get controlplane -n istio-system
No resources found.

# oc get po -n istio-system
No resources found.
```

Note that istio-operator is still running. If you are interested in just removing control plane to reinstall, you can stop here.

```
# oc get po -n istio-operator
NAME                              READY     STATUS    RESTARTS   AGE
istio-operator-79c5b4d68f-b6gj2   1/1       Running   0          19h
```

If you want to remove istio-operator, run `oc delete` using the same template that we used to create the operator. This removes the CRDs and the deployment for the operator

```
# oc delete -n istio-operator -f https://raw.githubusercontent.com/Maistra/istio-operator/maistra-0.10/deploy/servicemesh-operator.yaml


customresourcedefinition.apiextensions.k8s.io "installations.istio.openshift.com" deleted
customresourcedefinition.apiextensions.k8s.io "controlplanes.istio.openshift.com" deleted
clusterrole.rbac.authorization.k8s.io "istio-operator" deleted
serviceaccount "istio-operator" deleted
clusterrolebinding.rbac.authorization.k8s.io "istio-operator-account-istio-operator-cluster-role-binding" deleted
deployment.apps "istio-operator" deleted
```
and delete respective projects

```
oc delete project istio-system
```
and

```
oc delete project istio-operator
```

Now your cluster is completely free of istio and istio-operator.


## Preparing Istio Cluster for a Multi-user Workshop

### Additional access to the users 

As of now, each user that needs to run Istio examples need `view` access to the `istio-system` project. You can provide such access by running the following command for each user:

```
oc adm policy add-role-to-user view user1 -n istio-system
```
> **Note** If you are enabling this cluster for a workshop with a bunch of users, run the above command for all the user-ids created for the workshop.


### Additional access to the project

Applications are deployed in to projects/namespaces. On OpenShift the applications running in a namespace run with a `default` service account. This `default` service account runs with `restricted` SCC, which prevents it from running containers as specific user-ids or root, and also has restrictions on the linux capabilities. 

Istio requires specific kinds of access at the project level:

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
oc adm policy add-scc-to-user anyuid -z default -n bookinfo1
oc adm policy add-role-to-user admin user1 -n bookinfo1
```

Now, if a user logs in as `user1`, they will see both `bookinfo1` and `istio-system` in their list.


## Summary

In this chapter we learnt to perform the following administrative tasks on an OpenShift cluster:

* Preparing an OpenShift cluster to install Istio
* Installed Istio
* Enabled user(s) to run applications on Istio
* Created a project(s) with necessary privileges for end users to use 

