# Example of using Wallarm mirroring solution with Envoy

All traffic which goes to `httpbin` cluster will be mirrored to `wallarm` cluster.
NOTE: Replace `${WALLARM_ALB_FQDN}` and `${WALLARM_ALB_PORT}` with appropriate values of Internal ALB DNS name and port from outputs of [Terraform module](https://registry.terraform.io/modules/wallarm/wallarm/aws/0.9.1?tab=outputs).
`${WALLARM_ALB_FQDN}` should be FQDN and not URL, e.g. `wallarm-alb.myinternalzone.local`

Envoy configuration file example:
```
static_resources:
  listeners:
  - address:
      socket_address:
        address: 0.0.0.0
        port_value: 80
    filter_chains:
    - filters:
        - name: envoy.filters.network.http_connection_manager
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
            stat_prefix: ingress_http
            codec_type: AUTO
            route_config:
              name: local_route
              virtual_hosts:
              - name: backend
                domains:
                - "*"
                routes:
                - match: { prefix: "/" }
                  route:
                    cluster: httpbin
                    request_mirror_policies:
                     - cluster: wallarm
                       runtime_fraction: { default_value: { numerator: 100 } }
            http_filters:
            - name: envoy.filters.http.router
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
  - name: wallarm
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: wallarm
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: ${WALLARM_ALB_FQDN}
                port_value: ${WALLARM_ALB_PORT}
  - name: httpbin
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: httpbin
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: httpbin
                port_value: 80
```

Example from this folder can be simply run on a node which has Docker compose and access to Wallarm ALB deployed in your AWS account.
- Login to the node using ssh
- Clone current repo using `git clone`
- Run Docker compose `cd wallarm-outofband-examples/envoy && docker-compose up`
- Perform simple attack `curl -sD - http://localhost:80/get\?id='or+1=1--a-<script>prompt(1)</script>'`

After that you can see this attack in Wallarm Cloud console.
