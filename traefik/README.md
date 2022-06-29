# Example of using Wallarm mirroring solution with Traefik

According Traefik [official documentation](https://doc.traefik.io/traefik/routing/services/#mirroring-service)
the mirroring feature can be used either with [File](https://doc.traefik.io/traefik/providers/file/) or with
[IngressRoute](https://doc.traefik.io/traefik/providers/kubernetes-crd/) providers.

## Example with File provider 

Below is an example with File provider. In this example all traffic which complies with route rule `Router-1` will be mirrored.
Replace `${WALLARM_ALB_URL}` and `${WALLARM_ALB_PORT}` with appropriate values of Internal ALB DNS name and port from outputs of [Terraform module](https://registry.terraform.io/modules/wallarm/wallarm/aws/0.9.1?tab=outputs), e.g. `http://wallarm-alb.myinternalzone.local:8080`.
 
Note: we need to route incoming traffic to `mirrored-api` service, then traffic will be mirrored and send further
to appropriate service which is `httpbin` in this example.

Dynamic configuration file example:
```
# Dynamic configuration file
# Note: entry point 'web' is described in static configuration file
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

Static configuration file example:
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

Example from this folder can be simply run on a node which has Docker compose and access to Wallarm ALB deployed in your AWS account.
- Login to the node using ssh
- Clone current repo using `git clone`
- Run Docker compose `cd wallarm-outofband-examples/traefik && docker-compose up`
- Perform simple attack `curl -H "Host: httpbin.local" -sD - http://localhost:80/get\?id='or+1=1--a-<script>prompt(1)</script>'`

After that you can see this attack in Wallarm Cloud console. Have a nice day!