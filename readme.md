# Istio on Openshift

This write up covers my notes on running Istio on OpenShift. As I test with different implementations of OpenShift (such as minishift or a real openshift cluster), I'll add additional links here.

## Deploying Istio

* [Deploying Istio with `oc cluster up`](./DeployingIstioWithOcclusterup.md)
* Deploying Istio on OpenShift Cluster - Under Construction

## Infrastructure components on Istio
* [Deploy infrastructure components for metrics and tracing](./InstallInfrastructureComponents.md)

## Sample Application and Testing
* [Deploy Sample BookInfo Application](./DeployingSampleApplication.md)

In the following tests, you will run the same steps in [Istio Documentation](https://istio.io/docs/guides/intelligent-routing.html) except that you will replace `istioctl` with `oc`. 
As an example instead of running 	
`istioctl create -f samples/bookinfo/kube/route-rule-all-v1.yaml`, 	
you will run 	
`oc create -f samples/bookinfo/kube/route-rule-all-v1.yaml`

* [Canary using Content based routing](./CanaryContentBasedRouting.md)  [Istio Docs](https://istio.io/docs/tasks/traffic-management/request-routing.html#content-based-routing)	
* [Network Latency Fault Injection](./FaultInjection.md)     [Istio Docs](https://istio.io/docs/tasks/traffic-management/fault-injection.html)
* [Traffic Shaping using Routing Rules](./ABTesting.md)     [Istio Docs](https://istio.io/docs/tasks/traffic-management/traffic-shifting.html)
* [Request Timeouts](./RequestTimeOut.md)      [Istio Docs](https://istio.io/docs/tasks/traffic-management/request-timeouts.html)
* [Rules Precedence](./RulesPrecedence.md)     [Istio Docs](https://istio.io/docs/concepts/traffic-management/rules-configuration.html#rules-have-precedence)
* [Applying Rate Limits](./ApplyingRateLimits.md)   [Istio Docs](https://istio.io/docs/tasks/policy-enforcement/rate-limiting.html)
* More tests to be added