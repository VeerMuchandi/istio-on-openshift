# Deploying Istio with `oc cluster up`

This chapter explains deploying Istio with `oc cluster up`. We will need at least OpenShift 3.7. As of the time of writing this, OpenShift 3.7 is alpha.

The steps discussed here are adopted from the Istio setup on Kubernetes explained in the following link
[https://istio.io/docs/setup/kubernetes/quick-start.html](https://istio.io/docs/setup/kubernetes/quick-start.html)

**Prerequisites:**
* Docker should be running on your workstation

### Download OpenShift Client
Refer documentation here [https://docs.openshift.org/latest/cli_reference/get_started_cli.html](https://docs.openshift.org/latest/cli_reference/get_started_cli.html) to download and set up OpenShift CLI on your workstation.

I have tested this with the following version of OpenShift CLI running on my Mac.

```
$ oc version
oc v3.7.0-alpha.1+fdbd3dc
kubernetes v1.7.0+695f48a16f
features: Basic-Auth
```

### Start OpenShift Cluster

To start an OpenShift cluster on your box run the following command.

```
$ oc cluster up
```

This will download and start an OpenShift all-in-one image and start this image to run OpenShift on your workstation. It will also provide you a URL for your master (on Mac it gives https://127.0.0.1:8443 by default) and creates a user `developer` which you can use with any password of your choice. It will also log you in as `developer` by default. Also you will have a project named `myproject` created to use.

We have an OpenShift Kubernetes environment to test Istio on.

### Download Istio

You can download the installation file corresponding to your OS from here [https://github.com/istio/istio/releases](https://github.com/istio/istio/releases)

If you are using Mac or Linux you can run the following command that will extract the latest release automatically

```
$ curl -L https://git.io/getLatestIstio | sh -
```

I am testing with Istio version `0.2.7`. 

```
$ cd istio-0.2.7
```

Set the path to `istioctl` binary or copy to a location where it can run. As an example on Mac, I am copying istioctl to `/usr/local/bin` so that I can run this command. Verify running `istioctl version`.

```
$ cp bin/istioctl /usr/local/bin

$ istioctl version
Version: 0.2.7
GitRevision: 6b145c189aad8306b13af1725123bebfbc7eefd4
GitBranch: master
User: root@f1eeb85f62ab
GolangVersion: go1.8
```

You are all good to setup Istio now.

### Setup Istio

When you started the cluster. You were logged in as the user `developer`. In order to install Istio, you will need to log in as cluster administrator on OpenShift. Run the following command to log in as the cluster admin.

```
$ oc login -u system:admin
```

The components that make up Istio run as specific service accounts on Kubernetes and OpenShift. When we install Istio in the next few steps, a new project with name `istio-system` will be created and a few service accounts are added to this project. Any new service accounts created on OpenShift are restricted by default. `Ingress` and `Egress` pods run as root. Hence we have to allow these service accounts to run as `anyuid`. 


```
$ oc adm policy add-scc-to-user anyuid -z istio-ingress-service-account -n istio-system
$ oc adm policy add-scc-to-user anyuid -z istio-egress-service-account -n istio-system
```

In the same way, when we run supporting components `prometheus` and `grafana` for monitoring, they use `default` service account in the `istio-system` project. We are also allowing this `default` service account to containers as `anyuid`.

```
#prometheus and grafana run as root and they use default sa
$ oc adm policy add-scc-to-user anyuid -z default -n istio-system
```

**Note:** This above workaround may change in the future. 

Istio installer is available in two variants. 
* `istio.yaml` without mutual TLS authentication between sidecars
* `istio-auth.yaml` with mutual TLS authentication between sidecars

Run the following command to setup Istio. This will create a project named `istio-system`, creates clusterroles, service accounts, and deploys Istio on your OpenShift cluster.

```
$ oc apply -f install/kubernetes/istio-auth.yaml
namespace "istio-system" created
clusterrole "istio-pilot-istio-system" created
clusterrole "istio-initializer-istio-system" created
clusterrole "istio-mixer-istio-system" created
clusterrole "istio-ca-istio-system" created
clusterrole "istio-sidecar-istio-system" created
clusterrolebinding "istio-pilot-admin-role-binding-istio-system" created
clusterrolebinding "istio-initializer-admin-role-binding-istio-system" created
clusterrolebinding "istio-ca-role-binding-istio-system" created
clusterrolebinding "istio-ingress-admin-role-binding-istio-system" created
clusterrolebinding "istio-egress-admin-role-binding-istio-system" created
clusterrolebinding "istio-sidecar-role-binding-istio-system" created
clusterrolebinding "istio-mixer-admin-role-binding-istio-system" created
configmap "istio-mixer" created
service "istio-mixer" created
serviceaccount "istio-mixer-service-account" created
deployment "istio-mixer" created
customresourcedefinition "rules.config.istio.io" created
customresourcedefinition "attributemanifests.config.istio.io" created
customresourcedefinition "deniers.config.istio.io" created
customresourcedefinition "listcheckers.config.istio.io" created
customresourcedefinition "memquotas.config.istio.io" created
customresourcedefinition "noops.config.istio.io" created
customresourcedefinition "prometheuses.config.istio.io" created
customresourcedefinition "stackdrivers.config.istio.io" created
customresourcedefinition "statsds.config.istio.io" created
customresourcedefinition "stdios.config.istio.io" created
customresourcedefinition "svcctrls.config.istio.io" created
customresourcedefinition "checknothings.config.istio.io" created
customresourcedefinition "listentries.config.istio.io" created
customresourcedefinition "logentries.config.istio.io" created
customresourcedefinition "metrics.config.istio.io" created
customresourcedefinition "quotas.config.istio.io" created
customresourcedefinition "reportnothings.config.istio.io" created
attributemanifest "istioproxy" created
attributemanifest "kubernetes" created
stdio "handler" created
logentry "accesslog" created
rule "stdio" created
metric "requestcount" created
metric "requestduration" created
metric "requestsize" created
metric "responsesize" created
metric "tcpbytesent" created
metric "tcpbytereceived" created
prometheus "handler" created
rule "promhttp" created
rule "promtcp" created
configmap "istio" created
customresourcedefinition "destinationpolicies.config.istio.io" created
customresourcedefinition "egressrules.config.istio.io" created
customresourcedefinition "routerules.config.istio.io" created
service "istio-pilot" created
serviceaccount "istio-pilot-service-account" created
deployment "istio-pilot" created
service "istio-ingress" created
serviceaccount "istio-ingress-service-account" created
deployment "istio-ingress" created
service "istio-egress" created
serviceaccount "istio-egress-service-account" created
deployment "istio-egress" created
serviceaccount "istio-ca-service-account" created
deployment "istio-ca" created
```

It will take a few minutes for these images to be downloaded and the pods to come up. 

### Verify Istio

Switch over to `istio-system` project and understand all the components that are deployed. Look at the service accounts, pods, and different types of custom resource definitions added by the previous step. You will find the 5 core components of Istio running as pods and their correspoding deployments i.e, ca, pilot, mixer, ingress and egress.


```
$ oc project istio-system

$ oc get sa
NAME                            SECRETS   AGE
builder                         2         23s
default                         2         23s
deployer                        2         23s
istio-ca-service-account        2         19s
istio-egress-service-account    2         20s
istio-ingress-service-account   2         20s
istio-mixer-service-account     2         22s
istio-pilot-service-account     2         20s

$ oc get pods
NAME                            READY     STATUS    RESTARTS   AGE
istio-ca-2617747623-0ch0b       1/1       Running   0          15s
istio-egress-2389443630-l8706   1/1       Running   0          16s
istio-ingress-355016184-nd4gp   1/1       Running   0          16s
istio-mixer-3229407178-v3q3m    2/2       Running   0          19s
istio-pilot-589912157-7x7p7     1/1       Running   0          17s

$ oc get crd
NAME                                  KIND
attributemanifests.config.istio.io    CustomResourceDefinition.v1beta1.apiextensions.k8s.io
checknothings.config.istio.io         CustomResourceDefinition.v1beta1.apiextensions.k8s.io
deniers.config.istio.io               CustomResourceDefinition.v1beta1.apiextensions.k8s.io
destinationpolicies.config.istio.io   CustomResourceDefinition.v1beta1.apiextensions.k8s.io
egressrules.config.istio.io           CustomResourceDefinition.v1beta1.apiextensions.k8s.io
listcheckers.config.istio.io          CustomResourceDefinition.v1beta1.apiextensions.k8s.io
listentries.config.istio.io           CustomResourceDefinition.v1beta1.apiextensions.k8s.io
logentries.config.istio.io            CustomResourceDefinition.v1beta1.apiextensions.k8s.io
memquotas.config.istio.io             CustomResourceDefinition.v1beta1.apiextensions.k8s.io
metrics.config.istio.io               CustomResourceDefinition.v1beta1.apiextensions.k8s.io
noops.config.istio.io                 CustomResourceDefinition.v1beta1.apiextensions.k8s.io
prometheuses.config.istio.io          CustomResourceDefinition.v1beta1.apiextensions.k8s.io
quotas.config.istio.io                CustomResourceDefinition.v1beta1.apiextensions.k8s.io
reportnothings.config.istio.io        CustomResourceDefinition.v1beta1.apiextensions.k8s.io
routerules.config.istio.io            CustomResourceDefinition.v1beta1.apiextensions.k8s.io
rules.config.istio.io                 CustomResourceDefinition.v1beta1.apiextensions.k8s.io
stackdrivers.config.istio.io          CustomResourceDefinition.v1beta1.apiextensions.k8s.io
statsds.config.istio.io               CustomResourceDefinition.v1beta1.apiextensions.k8s.io
stdios.config.istio.io                CustomResourceDefinition.v1beta1.apiextensions.k8s.io
svcctrls.config.istio.io              CustomResourceDefinition.v1beta1.apiextensions.k8s.io

$ oc get attributemanifests
NAME         KIND
istioproxy   attributemanifest.v1alpha2.config.istio.io
kubernetes   attributemanifest.v1alpha2.config.istio.io

$ oc get metrics
NAME              KIND
requestcount      metric.v1alpha2.config.istio.io
requestduration   metric.v1alpha2.config.istio.io
requestsize       metric.v1alpha2.config.istio.io
responsesize      metric.v1alpha2.config.istio.io
tcpbytereceived   metric.v1alpha2.config.istio.io
tcpbytesent       metric.v1alpha2.config.istio.io

$ oc get prometheuses
NAME      KIND
handler   prometheus.v1alpha2.config.istio.io

$ oc get rules
NAME       KIND
promhttp   rule.v1alpha2.config.istio.io
promtcp    rule.v1alpha2.config.istio.io
stdio      rule.v1alpha2.config.istio.io

$ oc get logentries
NAME        KIND
accesslog   logentry.v1alpha2.config.istio.io

$ oc get stdios
NAME      KIND
handler   stdio.v1alpha2.config.istio.io


$ oc get deployments
NAME            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
istio-ca        1         1         1            1           1h
istio-egress    1         1         1            1           1h
istio-ingress   1         1         1            1           1h
istio-mixer     1         1         1            1           1h
istio-pilot     1         1         1            1           1h
```

Note the services running here. 

```
$ oc get svc
NAME            CLUSTER-IP      EXTERNAL-IP                     PORT(S)                                                  AGE
istio-egress    172.30.98.105   <none>                          80/TCP                                                   3m
istio-ingress   172.30.45.112   172.29.101.193,172.29.101.193   80:31388/TCP,443:30520/TCP                               3m
istio-mixer     172.30.82.151   <none>                          9091/TCP,9093/TCP,9094/TCP,9102/TCP,9125/UDP,42422/TCP   3m
istio-pilot     172.30.82.242   <none>                          8080/TCP,443/TCP
```

`istio-ingress` is the entrypoint for all your traffic through Istio. The simplest way to get our routing to work on OpenShift is to expose this service as an openshift route so that the openshift router that captures traffic on ports 80/443 will send the traffic to this `istio-ingress` service and rest of the control is with `istio-ingress`. So create a route by running:

```
$ oc expose svc istio-ingress

$ oc get route -n istio-system
NAME            HOST/PORT                                     PATH      SERVICES        PORT      TERMINATION   WILDCARD
istio-ingress   istio-ingress-istio-system.127.0.0.1.nip.io             istio-ingress   http  
```
Now the url [http://istio-ingress-istio-system.127.0.0.1.nip.io](istio-ingress-istio-system.127.0.0.1.nip.io) is my ingress point.

You are now good to start running Istio examples.



