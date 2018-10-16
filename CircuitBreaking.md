# Circuit Breaking

Circuit breaking allows you to write applications that limit the impact of failures, latency spikes, and other undesirable effects of network peculiarities. Here we will configure circuit breaking rules and then test the configuration by intentionally “tripping” the circuit breaker


### Prerequisites
* A running Istio Cluster


## Applying Circuit Breaker

Create a new project to deploy our sample.

```
oc new-project httpbin
Now using project "httpbin" on server "https://192.168.64.79:8443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app centos/ruby-22-centos7~https://github.com/openshift/ruby-ex.git

to build a new example application in Ruby.
```

Allow priveleged SCC to the default service account in the project created above

```
$ oc adm policy add-scc-to-user privileged -z default -n httpbin
scc "privileged" added to: ["system:serviceaccount:httpbin:default"]
```

Deploy the `httpbin` application

```
$ kubectl apply -f samples/httpbin/httpbin.yaml
service/httpbin created
deployment.extensions/httpbin created
```

Create a destination rule to apply circuit breaking policy on the httpbin service

```
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutiveErrors: 1
      interval: 1s
      baseEjectionTime: 3m
      maxEjectionPercent: 100
EOF
```

This configures circuit breaking for connections, requests, and outlier detection

Create a load testing client `fortio`, to send traffic to the httpbin service. Fortio lets us control the number of connections, concurrency, and delays for outgoing HTTP calls. We will use this client to “trip” the circuit breaker policies set in the DestinationRule.

```
kubectl apply -f <(istioctl kube-inject -f samples/httpbin/sample-client/fortio-deploy.yaml)
```

Now let us login to the client pod and use fortio tool to call the httpd bin service.

Finding the pod name
```
FORTIO_POD=$(kubectl get pod | grep fortio | awk '{ print $1 }')
```

```
$ echo $FORTIO_POD
fortio-deploy-6fb59f9d6d-qmwh4
```

Check the name of the `httpdbin` service. 

```
$ kubectl get services
NAME      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
httpbin   ClusterIP   172.30.110.136   <none>        8000/TCP   4m
```

Now call the `httpdbin` service from inside the client pod to observe that the request is successful.

```
$ kubectl exec -it $FORTIO_POD  -c fortio /usr/local/bin/fortio -- load -curl  http://httpbin:8000/get
HTTP/1.1 200 OK
server: envoy
date: Tue, 16 Oct 2018 18:07:24 GMT
content-type: application/json
access-control-allow-origin: *
access-control-allow-credentials: true
content-length: 934
x-envoy-upstream-service-time: 17

{
  "args": {}, 
  "headers": {
    "Content-Length": "0", 
    "Host": "httpbin:8000", 
    "User-Agent": "istio/fortio-1.0.1", 
    "X-B3-Sampled": "1", 
    "X-B3-Spanid": "702e0b5984cd62be", 
    "X-B3-Traceid": "702e0b5984cd62be", 
    "X-Envoy-Decorator-Operation": "httpbin.httpbin.svc.cluster.local:8000/*", 
    "X-Istio-Attributes": "CjoKE2Rlc3RpbmF0aW9uLnNlcnZpY2USIxIhaHR0cGJpbi5odHRwYmluLnN2Yy5jbHVzdGVyLmxvY2FsCkMKCnNvdXJjZS51aWQSNRIza3ViZXJuZXRlczovL2ZvcnRpby1kZXBsb3ktNmZiNTlmOWQ2ZC1xbXdoNC5odHRwYmluCj8KGGRlc3RpbmF0aW9uLnNlcnZpY2UuaG9zdBIjEiFodHRwYmluLmh0dHBiaW4uc3ZjLmNsdXN0ZXIubG9jYWwKPQoXZGVzdGluYXRpb24uc2VydmljZS51aWQSIhIgaXN0aW86Ly9odHRwYmluL3NlcnZpY2VzL2h0dHBiaW4KJQoYZGVzdGluYXRpb24uc2VydmljZS5uYW1lEgkSB2h0dHBiaW4KKgodZGVzdGluYXRpb24uc2VydmljZS5uYW1lc3BhY2USCRIHaHR0cGJpbg==", 
    "X-Request-Id": "51e3b07e-daf1-9544-a946-5a9a7de3f05d"
  }, 
  "origin": "172.17.0.29", 
  "url": "http://httpbin:8000/get"
}

```


Now let's call the service with two concurrent connections (-c 2) and send 20 requests (-n 20)

```
$ kubectl exec -it $FORTIO_POD  -c fortio /usr/local/bin/fortio -- load -c 2 -qps 0 -n 20 -loglevel Warning http://httpbin:8000/get
18:09:14 I logger.go:97> Log level is now 3 Warning (was 2 Info)
Fortio 1.0.1 running at 0 queries per second, 4->4 procs, for 20 calls: http://httpbin:8000/get
Starting at max qps with 2 thread(s) [gomax 4] for exactly 20 calls (10 per thread + 0)
18:09:14 W http_client.go:604> Parsed non ok code 503 (HTTP/1.1 503)
18:09:14 W http_client.go:604> Parsed non ok code 503 (HTTP/1.1 503)
Ended after 34.963125ms : 20 calls. qps=572.03
Aggregated Function Time : count 20 avg 0.0033139377 +/- 0.001149 min 0.001911933 max 0.005379765 sum 0.066278755
# range, mid point, percentile, count
>= 0.00191193 <= 0.002 , 0.00195597 , 5.00, 1
> 0.002 <= 0.003 , 0.0025 , 50.00, 9
> 0.003 <= 0.004 , 0.0035 , 70.00, 4
> 0.004 <= 0.005 , 0.0045 , 90.00, 4
> 0.005 <= 0.00537976 , 0.00518988 , 100.00, 2
# target 50% 0.003
# target 75% 0.00425
# target 90% 0.005
# target 99% 0.00534179
# target 99.9% 0.00537597
Sockets used: 4 (for perfect keepalive, would be 2)
Code 200 : 18 (90.0 %)
Code 503 : 2 (10.0 %)
Response Header Sizes : count 20 avg 207 +/- 69 min 0 max 230 sum 4140
Response Body/Total Sizes : count 20 avg 1069.3 +/- 284.1 min 217 max 1164 sum 21386
All done 20 calls (plus 0 warmup) 3.314 ms avg, 572.0 qps
```

We observe that most of the requests have gone thru `http code 200` and a couple of them have failed due to circuit breaking.

```
Code 200 : 18 (90.0 %)
Code 503 : 2 (10.0 %)
```

Let's increase the load by increasing the number of concurrent connections up to 3:

```
$ kubectl exec -it $FORTIO_POD  -c fortio /usr/local/bin/fortio -- load -c 3 -qps 0 -n 20 -loglevel Warning http://httpbin:8000/get
18:10:39 I logger.go:97> Log level is now 3 Warning (was 2 Info)
Fortio 1.0.1 running at 0 queries per second, 4->4 procs, for 20 calls: http://httpbin:8000/get
Starting at max qps with 3 thread(s) [gomax 4] for exactly 20 calls (6 per thread + 2)
18:10:39 W http_client.go:604> Parsed non ok code 503 (HTTP/1.1 503)
18:10:39 W http_client.go:604> Parsed non ok code 503 (HTTP/1.1 503)
18:10:39 W http_client.go:604> Parsed non ok code 503 (HTTP/1.1 503)
18:10:39 W http_client.go:604> Parsed non ok code 503 (HTTP/1.1 503)
18:10:39 W http_client.go:604> Parsed non ok code 503 (HTTP/1.1 503)
18:10:39 W http_client.go:604> Parsed non ok code 503 (HTTP/1.1 503)
18:10:39 W http_client.go:604> Parsed non ok code 503 (HTTP/1.1 503)
18:10:39 W http_client.go:604> Parsed non ok code 503 (HTTP/1.1 503)
18:10:39 W http_client.go:604> Parsed non ok code 503 (HTTP/1.1 503)
18:10:39 W http_client.go:604> Parsed non ok code 503 (HTTP/1.1 503)
Ended after 37.302157ms : 20 calls. qps=536.16
Aggregated Function Time : count 20 avg 0.0042114846 +/- 0.004256 min 0.000481987 max 0.016395203 sum 0.084229692
# range, mid point, percentile, count
>= 0.000481987 <= 0.001 , 0.000740994 , 25.00, 5
> 0.001 <= 0.002 , 0.0015 , 40.00, 3
> 0.002 <= 0.003 , 0.0025 , 50.00, 2
> 0.003 <= 0.004 , 0.0035 , 55.00, 1
> 0.004 <= 0.005 , 0.0045 , 70.00, 3
> 0.005 <= 0.006 , 0.0055 , 80.00, 2
> 0.006 <= 0.007 , 0.0065 , 85.00, 1
> 0.007 <= 0.008 , 0.0075 , 90.00, 1
> 0.012 <= 0.014 , 0.013 , 95.00, 1
> 0.016 <= 0.0163952 , 0.0161976 , 100.00, 1
# target 50% 0.003
# target 75% 0.0055
# target 90% 0.008
# target 99% 0.0163162
# target 99.9% 0.0163873
Sockets used: 12 (for perfect keepalive, would be 3)
Code 200 : 10 (50.0 %)
Code 503 : 10 (50.0 %)
Response Header Sizes : count 20 avg 115.1 +/- 115.1 min 0 max 231 sum 2302
Response Body/Total Sizes : count 20 avg 690.6 +/- 473.6 min 217 max 1165 sum 13812
All done 20 calls (plus 0 warmup) 4.211 ms avg, 536.2 qps
```

This time the failures are about 50%

```
Code 200 : 10 (50.0 %)
Code 503 : 10 (50.0 %)
```

Let us query the istio-proxy stats:

```
$ kubectl exec -it $FORTIO_POD  -c istio-proxy  -- sh -c 'curl localhost:15000/stats' | grep httpbin | grep pending
cluster.outbound|8000||httpbin.httpbin.svc.cluster.local.upstream_rq_pending_active: 0
cluster.outbound|8000||httpbin.httpbin.svc.cluster.local.upstream_rq_pending_failure_eject: 0
cluster.outbound|8000||httpbin.httpbin.svc.cluster.local.upstream_rq_pending_overflow: 12
cluster.outbound|8000||httpbin.httpbin.svc.cluster.local.upstream_rq_pending_total: 29
```

We see that 12 requests are flagged for circuit breaking.

```
httpbin.httpbin.svc.cluster.local.upstream_rq_pending_overflow: 12
```

## Clean up

Remove the destination rule

```
kubectl delete destinationrule httpbin
```
Remove the deployments for httpbin and the client

```
kubectl delete deploy --all  
kubectl get rs --all
kubectl delete po --all
kubectl delete svc httpbin
oc delete project httpbin
```



