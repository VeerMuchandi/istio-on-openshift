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
* [Request Routing and Identity Based Routing](./RequestRouting.md)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;   [Istio_Documentation](https://istio.io/docs/tasks/traffic-management/request-routing/)	
* [Fault Injections - Latency and Abort](./FaultInjection.md)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;     [Istio_Documentation](https://istio.io/docs/tasks/traffic-management/fault-injection.html)
* [Traffic Shifting - AB Testing](./ABTesting.md)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;     [Istio_Documentation](https://istio.io/docs/tasks/traffic-management/traffic-shifting/)
* [Request Timeouts](./RequestTimeOut.md)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;      [Istio_Documentation](https://istio.io/docs/tasks/traffic-management/request-timeouts)
* [Rules Precedence](./RulesPrecedence.md)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;     
* [Applying Rate Limits](./ApplyingRateLimits.md)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;   [Istio_Documentation](https://istio.io/docs/tasks/policy-enforcement/rate-limiting.html)
* More tests to be added