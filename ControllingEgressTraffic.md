# Egress Traffic

### Prerequisites

* We need a running Istio Cluster


Sidecar proxy by default handles only intercluster communications i.e, access is not allowed to any external URLs. Here we will learn how to make external services reachable from the services running on your cluster by defining [ServiceEntry](https://istio.io/docs/reference/config/istio.networking.v1alpha3/#ServiceEntry) configurations.

### Deploy an application to test Connectivity

Let us deploy a simple sample application in the project that is istio-enabled. This application can be used as a terminal to `curl` to make external calls.

```
kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml)
```

Once the application starts up, you'll see the corresponding pod

```
$ kubectl get po
NAME                              READY     STATUS    RESTARTS   AGE
sleep-74fcbff5d9-sn52m            2/2       Running   0          3h
```

Once the pod is in `Running` status, note the pod name and you can log into the pod and get access to the terminal by running the following command:

> **Note** Substitute your pod name

```
$ oc rsh -c sleep sleep-74fcbff5d9-sn52m
/ # 
```
This command will exec into the pod, and specifically into the container named sleep. We choose a particular container with `-c sleep` because our pod has multiple containers i.e, istio-sidecar container runs alongside your application container. If you want the same effect with kubectl, you can run `kubectl exec -it sleep-74fcbff5d9-sn52m -c sleep bash`  as an alternative.

You can exit this pod by typing in `exit`

### Configure External Services

Allowing access to an external service over `HTTP` such as [http://httpbin.org](http://httpbin.org) requires us to define a ServiceEntry. Run the following command to create a ServiceEntry. 

```
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: httpbin-ext
spec:
  hosts:
  - httpbin.org
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS
  location: MESH_EXTERNAL
EOF
```

For HTTPS services, we also need a VirtualService in addition to ServiceEntry. So create both ServiceEntry and VirtualService to be able to connect to [https://www.google.com](https://www.google.com) as shown below.

```
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: google
spec:
  hosts:
  - www.google.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
  location: MESH_EXTERNAL
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: google
spec:
  hosts:
  - www.google.com
  tls:
  - match:
    - port: 443
      sni_hosts:
      - www.google.com
    route:
    - destination:
        host: www.google.com
        port:
          number: 443
      weight: 100
EOF
```

Now login to the pod as explained before using `oc rsh ` or `kubectl exec -it` and test the following from inside the container.

Connect to http service running `curl http://httpbin.org/headers` as shown below:

```
/ # curl http://httpbin.org/headers
{
  "headers": {
    "Accept": "*/*", 
    "Connection": "close", 
    "Host": "httpbin.org", 
    "User-Agent": "curl/7.60.0", 
    "X-B3-Sampled": "1", 
    "X-B3-Spanid": "6dcbc1baa3f7d083", 
    "X-B3-Traceid": "6dcbc1baa3f7d083", 
    "X-Envoy-Decorator-Operation": "httpbin.org:80/*", 
    "X-Envoy-Expected-Rq-Timeout-Ms": "3000", 
    "X-Istio-Attributes": "Cj4KCnNvdXJjZS51aWQSMBIua3ViZXJuZXRlczovL3NsZWVwLTc0ZmNiZmY1ZDktc241Mm0uYm9va2luZm8yMA=="
  }
}
```

Connect to the https service running `curl https://www.google.com` as shown below:

```
# curl https://www.google.com
```

### Applying Routing Rules to Egress

Similar to requests flowing between services running within the service mesh, routing rules can also be applied to external services. Let us set a timeout rule and test here.

First let us test without a timeout, but with a 5 second delay:

> **Note** Substitute pod name

```
$ oc rsh -c sleep sleep-74fcbff5d9-sn52m time curl -o /dev/null -s -w "%{http_code}\n" http://httpbin.org/delay/5
200
real	0m 5.25s
user	0m 0.00s
sys	0m 0.00s
```

You can also run a more wordy kubectl command `kubectl exec -it $SOURCE_POD -c sleep bash time curl -o /dev/null -s -w "%{http_code}\n" http://httpbin.org/delay/5` instead. At this point you'll notice a return code of `200` which indicates a success and the time taken to service this request is a little over 5s due to the delay we set.

Now let us set a 3s timeout by creating a VirtualService for httpbin.org as shown below:

```
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin-ext
spec:
  hosts:
    - httpbin.org
  http:
  - timeout: 3s
    route:
      - destination:
          host: httpbin.org
        weight: 100
EOF
```

Now test calling with 5s delay again, and this time it should timeout after 3s and you should see a `504` response.

```
$ oc rsh -c sleep sleep-74fcbff5d9-sn52m time curl -o /dev/null -s -w "%{http_code}\n" http://httpbin.org/delay/5
504
real	0m 3.01s
user	0m 0.00s
sys	0m 0.00s
```

### Summary

In this lab we have learnt to allow egress traffic to external services and set routing rules to control the traffic

### Cleanup

Remove the rules

```
$ kubectl delete serviceentry httpbin-ext google
$ kubectl delete virtualservice httpbin-ext google
```

Remove the sleep service

```
kubectl delete -f samples/sleep/sleep.yaml
```








