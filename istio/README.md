# Example of using Wallarm mirroring solution with Istio

Prerequisites:
- Istio discovery and Istio gateway are [installed](https://istio.io/latest/docs/setup/install/)
- Istio sidecar [injection enabled](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/) within namespace or on per-pod basis
- Wallarm ALB is accessible from Kubernetes cluster

This example describes how to use Wallarm mirroring solution in environment with Istio service mesh. This example
shows how to mirror traffic which comes to Kubernetes service `httbin`. Mirroring will work for both traffic types: 
local traffic inside the cluster and external traffic which comes from outside the cluster.

To turn on mirroring in environment with Istio service mesh we need to do the following steps: 
- Create Istio `ServiceEntry` resource which points to Wallarm ALB deployed
- Add `spec.http.route.mirror` section into existing Istio `VirtualService` resource

NOTE: in examples below you need to replace `${WALLARM_ALB_HOST}` and `${WALLARM_ALB_PORT}` with appropriate values.

Example of Istio `ServiceEntry` resource is shown below.
```
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: wallarm-external-svc
spec:
  hosts:
    - ${WALLARM_ALB_HOST}
  location: MESH_EXTERNAL
  ports:
    - number: ${WALLARM_ALB_PORT}
      name: http
      protocol: HTTP
  resolution: DNS
```

Example of Istio `VirtualService` resource is shown below. This example assumes that:
- Istio `Gateway` resource with name `httpbin-gateway` is deployed in the same namespace, in order to route incoming traffic from outside the Kubernetes cluster using DNS hostname `httpbin.local`.
- Kubernetes service with name `httpbin` is deployed and points to workload (Pods).
```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
    - httpbin
    - httpbin.local
  gateways:
    - httpbin-gateway
    - mesh
  http:
    - route:
        - destination:
            host: httpbin
            port:
              number: 80
          weight: 100
      mirror:
        host: ${WALLARM_ALB_HOST}
        port:
          number: ${WALLARM_ALB_PORT}
```

Example of Istio `Gateway` resource which is needed if we want to route incoming traffic from outside the Kubernetes cluster
```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  selector:
    istio: ingress
    app: istio-ingress
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "httpbin.local"
```