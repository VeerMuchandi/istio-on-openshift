# Applying Rate Limits

In this exercise we will test rate limiting a service by defining a rate limit handler. Assume that you want to allow a certain service to be used only 'n' number of times in a given period. 

### Pre-requisites
* A running Istio Cluster
* Sample BookInfo Application deployed
* Destination rules created
* Create virtual services that would default to v1 i.e, `kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml`

### Exercise


```
$ cat samples/bookinfo/policy/mixer-rule-productpage-ratelimit.yaml
apiVersion: "config.istio.io/v1alpha2"
kind: memquota
metadata:
  name: handler
  namespace: istio-system
spec:
  quotas:
  - name: requestcount.quota.istio-system
    maxAmount: 500
    validDuration: 1s
    # The first matching override is applied.
    # A requestcount instance is checked against override dimensions.
    overrides:
    # The following override applies to 'reviews' regardless
    # of the source.
    - dimensions:
        destination: reviews
      maxAmount: 1
      validDuration: 5s
    # The following override applies to 'productpage' when
    # the source is a specific ip address.
    - dimensions:
        destination: productpage
        source: "10.28.11.20"
      maxAmount: 500
      validDuration: 1s
    # The following override applies to 'productpage' regardless
    # of the source.
    - dimensions:
        destination: productpage
      maxAmount: 2
      validDuration: 5s
---
apiVersion: "config.istio.io/v1alpha2"
kind: quota
metadata:
  name: requestcount
  namespace: istio-system
spec:
  dimensions:
    source: request.headers["x-forwarded-for"] | "unknown"
    destination: destination.labels["app"] | destination.service | "unknown"
    destinationVersion: destination.labels["version"] | "unknown"
---
apiVersion: config.istio.io/v1alpha2
kind: QuotaSpec
metadata:
  name: request-count
  namespace: istio-system
spec:
  rules:
  - quotas:
    - charge: 1
      quota: requestcount
---
apiVersion: config.istio.io/v1alpha2
kind: QuotaSpecBinding
metadata:
  name: request-count
  namespace: istio-system
spec:
  quotaSpecs:
  - name: request-count
    namespace: istio-system
  services:
  - name: productpage
    namespace: default
    #  - service: '*'  # Uncomment this to bind *all* services to request-count
---
apiVersion: config.istio.io/v1alpha2
kind: rule
metadata:
  name: quota
spec:
  # quota only applies if you are not logged in.
  # match: match(request.headers["cookie"], "user=*") == false
  actions:
  - handler: handler.memquota
    instances:
    - requestcount.quota
```


```
$ kubectl apply -f samples/bookinfo/policy/mixer-rule-productpage-ratelimit.yaml
memquota.config.istio.io/handler created
quota.config.istio.io/requestcount created
quotaspec.config.istio.io/request-count created
quotaspecbinding.config.istio.io/request-count created
rule.config.istio.io/quota created
```


```
$ kubectl -n istio-system get memquota handler -o yaml
apiVersion: config.istio.io/v1alpha2
kind: memquota
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"config.istio.io/v1alpha2","kind":"memquota","metadata":{"annotations":{},"name":"handler","namespace":"istio-system"},"spec":{"quotas":[{"maxAmount":500,"name":"requestcount.quota.istio-system","overrides":[{"dimensions":{"destination":"reviews"},"maxAmount":1,"validDuration":"5s"},{"dimensions":{"destination":"productpage","source":"10.28.11.20"},"maxAmount":500,"validDuration":"1s"},{"dimensions":{"destination":"productpage"},"maxAmount":2,"validDuration":"5s"}],"validDuration":"1s"}]}}
  clusterName: ""
  creationTimestamp: 2018-10-16T17:46:13Z
  generation: 1
  name: handler
  namespace: istio-system
  resourceVersion: "15902"
  selfLink: /apis/config.istio.io/v1alpha2/namespaces/istio-system/memquotas/handler
  uid: 5fd5f613-d16b-11e8-bfea-625578ce8fcd
spec:
  quotas:
  - maxAmount: 500
    name: requestcount.quota.istio-system
    overrides:
    - dimensions:
        destination: reviews
      maxAmount: 1
      validDuration: 5s
    - dimensions:
        destination: productpage
        source: 10.28.11.20
      maxAmount: 500
      validDuration: 1s
    - dimensions:
        destination: productpage
      maxAmount: 2
      validDuration: 5s
    validDuration: 1s
```



Direct user Jason to reviews version v2 by running

```
$ oc create -f samples/bookinfo/kube/route-rule-reviews-test-v2.yaml
routerule "reviews-test-v2" created
```

Direct all other users to reviews version v3

```
$ oc replace -f samples/bookinfo/kube/route-rule-reviews-v3.yaml
routerule "reviews-default" replaced		
```

Test the application in browser. All the requests from Jason should show black stars for ratings. For any other user, it should show red stars.

Now let's apply rate-limits

We will first define a memquota adapter with following features:	
* Apply default rate limit of 5000 queries per second (qps) for the ratings service 	
* Apply just 1 queries for every 5s for ratings v2 svc. This should give us enough time to test between the queries.	
So when the user "Jason" uses the system, the requests are directed to v2 and it should be limited to 1 query per 5s

The memquota adapter named handler is applied as shown below:

```
$ cat <<EOF | oc create -f -
apiVersion: config.istio.io/v1alpha2
kind: memquota
metadata:
  name: handler
  namespace: istio-system
spec:
  quotas:
  - name: requestcount.quota.istio-system
    # default rate limit is 5000qps
    maxAmount: 5000
    validDuration: 1s
    # The first matching override is applied.
    # A requestcount instance is checked against override dimensions.
    overrides:
    # The following override applies to traffic from 'rewiews' version v2,
    # destined for the ratings service. The destinationVersion dimension is ignored.
    - dimensions:
        destination: ratings
        source: reviews
        sourceVersion: v2
      maxAmount: 1
      validDuration: 5s
EOF

memquota "handler" created
```

Note that this handler is created in the `istio-system` project

```
$ oc get memquota -n istio-system
NAME      KIND
handler   memquota.v1alpha2.config.istio.io
```

We have one more step. We have to create a `quota` with name `requestcount`. The above handler is checking against `requestcount.quota.istio-system`.
Also a `rule` that ties the `handler` to `requestcount` as shown below:


```
$ cat <<EOF | oc create -f -
apiVersion: config.istio.io/v1alpha2
kind: quota
metadata:
  name: requestcount
  namespace: istio-system
spec:
  dimensions:
    source: source.labels["app"] | source.service | "unknown"
    sourceVersion: source.labels["version"] | "unknown"
    destination: destination.labels["app"] | destination.service | "unknown"
    destinationVersion: destination.labels["version"] | "unknown"
---
apiVersion: config.istio.io/v1alpha2
kind: rule
metadata:
  name: quota
  namespace: istio-system
spec:
  actions:
  - handler: handler.memquota
    instances:
    - requestcount.quota
EOF
```

Now we are ready to test.

If you test as a regular user (other than Jason), your requests will go through with no issues as you get 5000 qps. 

Now login as Jason and try to hit the refresh button twice within 5 seconds. The first request will show you **black** stars. But the next request will show you no stars as the call to ratings service is prevented. You will get back **black** stars only after 5 seconds.

### Summary

In this exercise we have seen how to apply rate limits to a service using Istio handler.



