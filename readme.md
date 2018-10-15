# Istio on Openshift

This write up covers my notes on running Istio on OpenShift. As I test with different implementations of OpenShift (such as minishift or a real openshift cluster), I'll add additional links here.

## Deploying Istio

* [Deploying Istio with minishift](./DeployingIstioWithMinishift.md)
* Deploying Istio on OpenShift Cluster - Under Construction

## Supporting Services
* [Understanding Supporting Services](./usingIstioSupportingServices.md)


## Sample Application 
* [Deploy Sample BookInfo Application](./DeployingSampleApplication.md)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; [Istio_Documentation](https://istio.io/docs/examples/bookinfo/)

## Traffic Management
* [Request Routing and Identity Based Routing](./RequestRouting.md)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;   [Istio_Documentation](https://istio.io/docs/tasks/traffic-management/request-routing.html#content-based-routing)	
* [Network Latency Fault Injection](./FaultInjection.md)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;     [Istio_Documentation](https://istio.io/docs/tasks/traffic-management/fault-injection.html)
* [Traffic Shaping using Routing Rules](./ABTesting.md)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;     [Istio_Documentation](https://istio.io/docs/tasks/traffic-management/traffic-shifting.html)
* [Request Timeouts](./RequestTimeOut.md)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;      [Istio_Documentation](https://istio.io/docs/tasks/traffic-management/request-timeouts.html)
* [Rules Precedence](./RulesPrecedence.md)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;     [Istio_Documentation](https://istio.io/docs/concepts/traffic-management/rules-configuration.html#rules-have-precedence)
* [Applying Rate Limits](./ApplyingRateLimits.md)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;   [Istio_Documentation](https://istio.io/docs/tasks/policy-enforcement/rate-limiting.html)
* More tests to be added