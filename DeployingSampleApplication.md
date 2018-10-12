# Deploying sample application

### Prerequisites

* Istio is installed and running 
* Istio samples and tools are also available on your desktop

### Install Bookinfo sample application

We will deploy the sample bookinfo application explained in the [istiodocs](https://istio.io/docs/guides/bookinfo.html). The instructions are more or less the same as kubernetes with some slight variations. Hence I have documented the openshift deployment process here.

Create a new project for the application

```
$ oc new-project bookinfo
Now using project "bookinfo" on server "https://127.0.0.1:8443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app centos/ruby-22-centos7~https://github.com/openshift/ruby-ex.git

to build a new example application in Ruby.
```

When we install the bookinfo application, the application pods have init containers whose `proxy_init` runs in privileged mode and adds `NET_ADMIN`
as shown here. You will find this in the individual `deployment` artifacts.

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

These application pods current run as `default` service account in a project. Hence we have to provide `privileged` access to the `default` account. Eventually, these examples should change to not require privileged access. But in the meanwhile, here is how you can set `default` service account to `privileged` security context constraint (scc) in the `bookinfo` project/namespace.

```
$ oc adm policy add-scc-to-user privileged -z default -n bookinfo
```

Let's now deploy the `bookinfo` application. We are using `istioctl kube-inject` to add `Envoy` sidecar proxies to each of the kubernetes deployment yamls and using the resultant deployment yamls to create an application.

```
kubectl apply -f <(istioctl kube-inject -f samples/bookinfo/platform/kube/bookinfo.yaml)
```
and watch the output as shown below
```
service/details created
deployment.extensions/details-v1 created
service/ratings created
deployment.extensions/ratings-v1 created
service/reviews created
deployment.extensions/reviews-v1 created
deployment.extensions/reviews-v2 created
deployment.extensions/reviews-v3 created
service/productpage created
deployment.extensions/productpage-v1 created
```

Give a few mins for the container images to be pulled and for the pods to come up. Note all the components that are running.

```
$ kubectl get po
NAME                              READY     STATUS            RESTARTS   AGE
details-v1-5c6798dd8d-dbbbj       0/2       PodInitializing   0          33s
productpage-v1-6645b8588d-ztxpb   0/2       PodInitializing   0          29s
ratings-v1-86d8c8dfcb-rqxsx       0/2       PodInitializing   0          33s
reviews-v1-94b6b6b9d-4qhsx        0/2       PodInitializing   0          33s
reviews-v2-5d6498bd56-v5cxv       0/2       PodInitializing   0          32s
reviews-v3-76db6df594-927q6       0/2       PodInitializing   0          31s

$ kubectl get services
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
details       ClusterIP   172.30.4.163     <none>        9080/TCP   30s
productpage   ClusterIP   172.30.247.217   <none>        9080/TCP   28s
ratings       ClusterIP   172.30.119.188   <none>        9080/TCP   30s
reviews       ClusterIP   172.30.142.212   <none>        9080/TCP   29s
```

In order to make your application accessible from outside the cluster, an [Istio Gateway](https://istio.io/docs/concepts/traffic-management/#gateways) is required.

```
$ kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
gateway.networking.istio.io/bookinfo-gateway created
virtualservice.networking.istio.io/bookinfo created
```
This creates a gateway and a virtual service.

The gateway will direct all the `HTTP` traffic coming on port `80` at istio-ingressgateway to the bookinfo sample application

```
$ kubectl get gateway
NAME               CREATED AT
bookinfo-gateway   11s

$ kubectl get gateway bookinfo-gateway -o yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"networking.istio.io/v1alpha3","kind":"Gateway","metadata":{"annotations":{},"name":"bookinfo-gateway","namespace":"bookinfo"},"spec":{"selector":{"istio":"ingressgateway"},"servers":[{"hosts":["*"],"port":{"name":"http","number":80,"protocol":"HTTP"}}]}}
  clusterName: ""
  creationTimestamp: 2018-10-10T16:46:45Z
  generation: 1
  name: bookinfo-gateway
  namespace: bookinfo
  resourceVersion: "16011"
  selfLink: /apis/networking.istio.io/v1alpha3/namespaces/bookinfo/gateways/bookinfo-gateway
  uid: 12d4fddb-ccac-11e8-b16d-66d8cef31907
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - '*'
    port:
      name: http
      number: 80
      protocol: HTTP
```

And the virtualservice would match the traffic at specific endpoints `/productpage`, ` /login`, `/logout` etc.

```
$ kubectl get virtualservice
NAME       CREATED AT
bookinfo   6h

$ kubectl get virtualservice bookinfo -o yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"networking.istio.io/v1alpha3","kind":"VirtualService","metadata":{"annotations":{},"name":"bookinfo","namespace":"bookinfo"},"spec":{"gateways":["bookinfo-gateway"],"hosts":["*"],"http":[{"match":[{"uri":{"exact":"/productpage"}},{"uri":{"exact":"/login"}},{"uri":{"exact":"/logout"}},{"uri":{"prefix":"/api/v1/products"}}],"route":[{"destination":{"host":"productpage","port":{"number":9080}}}]}]}}
  clusterName: ""
  creationTimestamp: 2018-10-10T16:46:45Z
  generation: 1
  name: bookinfo
  namespace: bookinfo
  resourceVersion: "16012"
  selfLink: /apis/networking.istio.io/v1alpha3/namespaces/bookinfo/virtualservices/bookinfo
  uid: 12e74346-ccac-11e8-b16d-66d8cef31907
spec:
  gateways:
  - bookinfo-gateway
  hosts:
  - '*'
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
      
 
```



The virtual service above shows that we can access product page at `/productpage` endpoint. This endpoint is for the Istio ingress i.e.,

```
$ kubectl get route -n istio-system istio-ingressgateway
NAME                   HOST/PORT                                                PATH      SERVICES               PORT      TERMINATION   WILDCARD
istio-ingressgateway   istio-ingressgateway-istio-system.192.168.64.72.nip.io             istio-ingressgateway   http2                   None
```

> **Note** Your URLs would be different from mine. So use your values. If you want  to know your URLs run `kubectl get route -n istio-system`

So I can access the product page at the URL [http://istio-ingressgateway-istio-system.192.168.64.72.nip.io/productpage](http://istio-ingressgateway-istio-system.192.168.64.72.nip.io/productpage). *Use your own URL*

Familiarize with this application a little bit. Use it a few times. Go back and check the service graph on kiali at [https://kiali-istio-system.192.168.64.72.nip.io](https://kiali-istio-system.192.168.64.72.nip.io)
You will see that the graph as below: *Use your own URL*
![servicegraph](./images/kialiServiceGraph.jpeg)

Also notice the data collected by Prometheus and displayed on Grafana at [http://grafana-istio-system.192.168.64.72.nip.io/d/LJ_uJAvmk/istio-service-dashboard](http://grafana-istio-system.192.168.64.72.nip.io/d/LJ_uJAvmk/istio-service-dashboard) *Use your own URL*


We have application running now. It is now time to test the awesomeness of Istio.



