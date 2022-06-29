# Example of using Wallarm mirroring solution with Traefik

According Traefik [official documentation](https://doc.traefik.io/traefik/routing/services/#mirroring-service)
the mirroring feature can be used either with [File](https://doc.traefik.io/traefik/providers/file/) or with
[IngressRoute](https://doc.traefik.io/traefik/providers/kubernetes-crd/) providers.

Below is an example with File provider. In this example all traffic which complies with `Router-1` route rule will be mirrored.
NOTE:
- Replace `${WALLARM_ALB_URL}` and `${WALLARM_ALB_PORT}` with appropriate Internal ALB DNS name and port from [Terraform module outputs](https://registry.terraform.io/modules/wallarm/wallarm/aws/0.9.1?tab=outputs).
- We need to route incoming traffic to `mirrored-api` service, then traffic will be mirrored and send further
to appropriate service which is `httpbin` in our case.

Dynamic configuration file example
```
# Dynamic configuration file
# Note: entry 'web' point described in static configuration file
http:
  services:
    mirrored-api:
      mirroring:
        service: "httpbin"
        mirrors:
          - name: "wallarm"
            percent: 100

    wallarm:
      loadBalancer:
        servers:
          - url: "${WALLARM_ALB_URL}:${WALLARM_ALB_PORT}/"
    httpbin:
      loadBalancer:
        servers:
          - url: "http://httpbin:80/"

  routers:
    Router-1:
      entryPoints:
        - "web"
      rule: "Host(`httpbin.local`)"
      service: "mirrored-api"

```

Static configuration file example
```
api:
  insecure: true
  dashboard: true
  debug: true

providers:
  file:
    directory: /etc/traefik/dynamic
    filename: config.yaml
    watch: true

entryPoints:
  web:
    address: ":80"

log:
  level: DEBUG
  format: text
```
