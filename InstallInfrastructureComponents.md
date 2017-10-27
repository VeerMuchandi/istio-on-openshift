# Install Infrastructure Components

Switch over to `istio-system` project

```
$ oc project istio-system
```

Install [Zipkin](http://zipkin.io/) for tracing and expose the service as route so that you can access Zipkin UI.

```
$ oc apply -f install/kubernetes/addons/zipkin.yaml
$ oc expose svc zipkin
```

Install [Prometheus](https://prometheus.io/) for metrics and expose the service as route to access Prometheus.

```
$ oc apply -f install/kubernetes/addons/prometheus.yaml
$ oc expose svc prometheus
```

Prometheus uses [Grafana](https://grafana.com/) for visualization of metrics. Expose the service to get the UI.

```
$ oc apply -f install/kubernetes/addons/grafana.yaml
$ oc expose svc grafana
```

Service graph allows you to visualize in microservices call tree. Let's install servicegraph and expose the service to get the UI.

```
$ oc apply -f install/kubernetes/addons/servicegraph.yaml
$ oc expose svc servicegraph
```

If you list the routes in the `istio-system` project you will get all the URLs that you can access.

```
$ oc get route
NAME            HOST/PORT                                     PATH      SERVICES        PORT         TERMINATION   WILDCARD
grafana         grafana-istio-system.127.0.0.1.nip.io                   grafana         http                       None
istio-ingress   istio-ingress-istio-system.127.0.0.1.nip.io             istio-ingress   http                       None
prometheus      prometheus-istio-system.127.0.0.1.nip.io                prometheus      prometheus                 None
servicegraph    servicegraph-istio-system.127.0.0.1.nip.io              servicegraph    http                       None
zipkin          zipkin-istio-system.127.0.0.1.nip.io                    zipkin          http                       None
```

Open up the following URLs in the browser. As of now that data will all be empty as we don't have any application running yet.


To access service graph as a graph use `/dotviz` suffix

[http://servicegraph-istio-system.127.0.0.1.nip.io/dotviz](http://servicegraph-istio-system.127.0.0.1.nip.io/dotviz)


To access Istio dashboard on Grafana use:

[http://grafana-istio-system.127.0.0.1.nip.io/dashboard/db/istio-dashboard](http://grafana-istio-system.127.0.0.1.nip.io/dashboard/db/istio-dashboard)


For tracing using zipkin:
[http://zipkin-istio-system.127.0.0.1.nip.io/zipkin/](http://zipkin-istio-system.127.0.0.1.nip.io/zipkin/)

At this point we are ready to deploy an application and test.
