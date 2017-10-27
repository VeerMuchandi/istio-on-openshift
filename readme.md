# Istio on Openshift

This write up covers my notes on running Istio on OpenShift. As I test with different implementations of OpenShift (such as minishift or a real openshift cluster), I'll add additional links here.

## Deploying Istio

* [Deploying Istio with `oc cluster up`](./DeployingIstioWithOcclusterup.md)
* Deploying Istio on OpenShift Cluster

## Infrastructure components on Istio
* [Deploy infrastructure components for metrics and tracing](./InstallInfrastructureComponents.md)

## Sample Application and Testing
* [Deploy Sample BookInfo Application](./DeployingSampleApplication.md)

For testing, you will run the same steps in [Istio Documentation](https://istio.io/docs/guides/intelligent-routing.html) except that you will replace `istioctl` with `oc`. As an example 
instead of running `istioctl create -f samples/bookinfo/kube/route-rule-all-v1.yaml`, you will run `oc create -f samples/bookinfo/kube/route-rule-all-v1.yaml`

* Testing Canary, Content based routing [Istio Docs](https://istio.io/docs/tasks/traffic-management/request-routing.html#content-based-routing) [openshift commands](./CanaryContentBasedRouting.md)
* More tests to be added