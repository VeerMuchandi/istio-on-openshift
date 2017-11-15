# Rules Precedence
In this exercise we will see how the precedence of rules varies the behavior of application of rules on Istio.

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

Let's first update the reviews service to redirect all the traffic to version 3. But this time we will also give it a precedence of 3.

```
$ cat <<EOF | oc replace -f -
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: reviews-default
spec:
  destination:
    name: reviews
  precedence: 3
  route:
  - labels:
      version: v3
EOF
```

If you go and test the application, you will see reviews version 3 i.e, all **red** starts for ratings.


Now just like in content based routing lab we will redirect any traffic coming from user "jason" to reviews version v3. Let's quickly look at the routing rule first.

```
$ cat samples/bookinfo/kube/route-rule-reviews-test-v2.yaml
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: reviews-test-v2
spec:
  destination:
    name: reviews
  precedence: 2
  match:
    request:
      headers:
        cookie:
          regex: "^(.*?;)?(user=jason)(;.*)?$"
  route:
  - labels:
      version: v2
```

Note that there is a precedence of 2 for this rule. Let's apply this rule.

```
$ oc create -f samples/bookinfo/kube/route-rule-reviews-test-v2.yaml
routerule "reviews-test-v2" created
```
Now test the application by logging in as user "jason". You will still see reviews version 3 (**red** stars). **WHY?**

The `precedence` of the routing rules matters. The higher the value of precedence, that rule takes higher precedence. In our case, the `reviews-default` rule has a precedence of 3 whereas the content based rule for user "jason", `reviews-test-v2`, has a precedence of 2. So, even though the rule for "jason" exists, it is never touched due to lower precedence.

Let us lower the precedence for the `reviews-default` rule as follows:

```
$ cat <<EOF | oc replace -f -
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: reviews-default
spec:
  destination:
    name: reviews
  precedence: 1
  route:
  - labels:
      version: v3
EOF
```

If you test the application now, the traffic for user "Jason" hits reviews version 2 (displaying **black** stars for ratings) whereas all other traffic hits version 3 (displaying **red** starts for ratings).

### Summary
In this exercise we have tested the application of routing rules based on their precedence.




