# Request timeouts

In this exercise we will learn to to introduce timeouts using Routing rules.


### Pre-requisites
You want all the traffic being directed to v1 services. If the system state is inconsistent delete all the route rules and create them again 

```
$ oc delete routerule --all
routerule "details-default" deleted
routerule "productpage-default" deleted
routerule "ratings-default" deleted
routerule "reviews-default" deleted

$ oc create -f samples/bookinfo/kube/route-rule-all-v1.yaml
routerule "productpage-default" created
routerule "reviews-default" created
routerule "ratings-default" created
routerule "details-default" created
```

### Exercise

We'll change the default reviews routing rule to redirect the traffic to reviews version v2 as below:

```
$ cat <<EOF | oc replace -f -
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: reviews-default
spec:
  destination:
    name: reviews
  route:
  - labels:
      version: v2
EOF

routerule "reviews-default" replaced
```

If you test it now you will see all requests going to reviews v2 (black stars)
Note the response time for ratings service and reviews service in Zipkin it will be a few milliseconds.

Let's introduce 2 sec delay for the ratings service as in our [Fault Injection](./FaultInjection.md) exercise but for all the users (not just Jason). Note that the delay in this case is 2s which is still less than the timeout of 3s between product page and reviews page.


```
$ cat <<EOF | oc replace -f -
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: ratings-default
spec:
  destination:
    name: ratings
  route:
  - labels:
      version: v1
  httpFault:
    delay:
      percent: 100
      fixedDelay: 2s
EOF
```

Now if you test you will see that the there is a 2 sec wait for the response.
Note the response time for ratings service in Zipkin. It would have gone up from milliseconds measured earlier to over 2 seconds now. But the call is still successful. 


Now let's add 1 sec timeout for the reviews service. 

```
cat <<EOF | oc replace -f -
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: reviews-default
spec:
  destination:
    name: reviews
  route:
  - labels:
      version: v2
  httpReqTimeout:
    simpleTimeout:
      timeout: 1s
EOF
```

While the application timeout between the product page and reviews page is 3s, the above change will override that via routing rule and the lowest (1s) takes precedence. Now the Reviews part of the page gives an error

```
Error fetching product reviews!

Sorry, product reviews are currently unavailable for this book.
```

You can also see in Zipkin that the reviews calls ratings and timesout in approximately 1s.  Reviews is retried once and times out in 1s while waiting for ratings.

**Clean up** 

Replace all routing rules to default to version 1 services

```
$ oc replace -f samples/bookinfo/kube/route-rule-all-v1.yaml
```

### Summary

In this exercise we learnt to change the request timeouts for services.