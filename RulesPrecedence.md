# Rules Precedence
In this exercise we will see how the precedence of rules varies the behavior of application of rules on Istio. When there are multiple rules for a given destination, they are evaluated in the order they appear in the VirtualService, meaning the first rule in the list has the highest priority.

### Pre-requisites
* A running Istio Cluster
* Sample BookInfo Application deployed
* Destination rules created
* Create virtual services that would default to v1 i.e, `kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml`

## Understanding Precedence

If you test the application now, all the traffic will be flowing to reviews version v1 ie., you will see **no stars**


Let us now apply the rule to direct the traffic for user `jason` to reviews v3
as in [Request Routing and Identity Based Routing](./RequestRouting.md) with some minor modifications. The changes here are to apply the routing rule to v1 before matching the user `jason`.

```
  - route:
    - destination:
        host: reviews
        subset: v1
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
```

Let us apply the rule.

```
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
EOF

```

Now test the application without logging in as user `jason` first and then by logging in as user `jason`. In both cases, you will see a result with **no stars**.

**Why?**
 When there are multiple rules for a given destination, they are evaluated in the order they appear in the VirtualService, meaning the first rule in the list has the highest priority.
 
So if you want to evaluate the user `jason` and apply that user specific rule before falling over to the default, the order should be different as below.

```

  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1

```


Now let us correct the precedence and apply again

```
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
EOF

```
Once you apply the corrected precedence and test again you will observe

* when you are logged in as user `jason` you will see **black stars**
* when you are log out you will see **no stars**

## Cleanup 

To clean up, remove the routing rules by deleting the virtual services created earlier.

```
kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

## Summary

In this exercise we have tested the application of routing rules based on their precedence.




