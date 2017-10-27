# Canary/Content based routing

Test the application in the browser. "Reviews" output is random each time you access the page.
* Sometimes it hits reviews v1 (No stars)
* Sometimes it hits reviews v2 (black stars)
* Sometimes it hits reviews v3 (red stars)

Default traffic from all users to reviews version v1

```
$ oc create -f create -f samples/bookinfo/kube/route-rule-all-v1.yaml
```

Look at the routerules that are created

```
$ oc get routerule
NAME                  KIND
details-default       RouteRule.v1alpha2.config.istio.io
productpage-default   RouteRule.v1alpha2.config.istio.io
ratings-default       RouteRule.v1alpha2.config.istio.io
reviews-default       RouteRule.v1alpha2.config.istio.io
```

Understand the the `reviews-default` routerule. Note the traffic goes to version reviews v1 by default. Test in browser and you'll see no stars now.

```
$ oc get routerule reviews-default -o yaml
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  clusterName: ""
  creationTimestamp: 2017-10-27T20:58:12Z
  deletionGracePeriodSeconds: null
  deletionTimestamp: null
  name: reviews-default
  namespace: bookinfo
  resourceVersion: "6709"
  selfLink: /apis/config.istio.io/v1alpha2/namespaces/bookinfo/routerules/reviews-default
  uid: 8bad146c-bb59-11e7-9c32-1ad90b5af171
spec:
  destination:
    name: reviews
  precedence: 1
  route:
  - labels:
      version: v1
```      

Redirect specific user to version v2 based on the content in the cookie

```
$ oc create -f samples/bookinfo/kube/route-rule-reviews-test-v2.yaml
```

An additional routerule is added.

```
$ oc get routerule
NAME                  KIND
details-default       RouteRule.v1alpha2.config.istio.io
productpage-default   RouteRule.v1alpha2.config.istio.io
ratings-default       RouteRule.v1alpha2.config.istio.io
reviews-default       RouteRule.v1alpha2.config.istio.io
reviews-test-v2       RouteRule.v1alpha2.config.istio.io

$ oc get routerule reviews-test-v2 -o yaml
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  clusterName: ""
  creationTimestamp: 2017-10-27T21:06:10Z
  deletionGracePeriodSeconds: null
  deletionTimestamp: null
  name: reviews-test-v2
  namespace: bookinfo
  resourceVersion: "6905"
  selfLink: /apis/config.istio.io/v1alpha2/namespaces/bookinfo/routerules/reviews-test-v2
  uid: a89d1211-bb5a-11e7-9c32-1ad90b5af171
spec:
  destination:
    name: reviews
  match:
    request:
      headers:
        cookie:
          regex: ^(.*?;)?(user=jason)(;.*)?$
  precedence: 2
  route:
  - labels:
      version: v2
```
      
Per this newly added rule, if the cookie has `user=jason`, traffic goes to reviews v2 (black stars) otherwise default applies.

Test by Signing in as user 'jason' in the browser. Use any password. You'll only see black starts i.e, reviews v2. If you sign out you will see no stars!!

**Summary:** Assume you created a new version of reviews service v2 and you want to test it as specific user before releasing it or making it generally available. You introduce it as a canary to specific users to test. 