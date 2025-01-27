# K8S Gateway

[back](README#entities)

- makes network services available (extensible, role-oriented, protocol-aware configuration mechanism)
- role-oriented: kinds are modeled after orga roles
  - infrastructure Provider: manages infra for multi-isolated-clusters serving multi-tenants (e.g. cloud provider)
  - Cluster Operator: manages clusters and policies, network access, app permissions etc.
  - Application Developer: manages app in cluster and app level cfg and service compositions
- portable: defined as custom resource translating into multiple implementations
- expressive: kinds support traffic routing like header-based, weighting etc.
- extensible: links custom resources to various layers (granular customization)
- resource model
  - GatewayClass: set of gateways with common cfg managed by a controller implementing it
    - describes the gateway controller responsible for managing Gateways of this class
  - Gateway: instance of traffic handler e.g. cloud load balancer
    - associated with exactly one GatewayClass
    - can filter the routes that may be attached to its listeners
  - HTTPRoute: http-specific rules for mapping traffic from a Gateway listener to backend network endpoints
    - one or more route kinds such as HTTPRoute are associated to Gateways

## GatewayClass

- gateways can be implemented by different controllers and must reference a GatewayClass for that

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: example-class
spec:
  controllerName: example.com/gateway-controller
```

- ...
  - controller is configured to manage GatewayClasses with that `controllerName`
  - i.e. gateways of this class will be managed by the implementation controller

## Gateway

- defines a network endpoint e.g. in-cluster proxy accepting HTTP traffic

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: example-gateway
spec:
  gatewayClassName: example-class
  listeners:
    - name: http
      protocol: HTTP
      port: 80
```

- ...
  - listens to HTTP on port 80
  - `addresses` field is unspecified so address or hostname will be assigned by controller
    - address is then used as an endpoint for processing traffic of backend endpoints defined in routes

## HTTPRoute

- routing behavior from Gateway listener to backend network endpoints

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: example-httproute
spec:
  parentRefs:
    - name: example-gateway
  hostnames:
    - "www.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /login
      backendRefs:
        - name: example-svc
          port: 8080
```

- ...
  - HTTP from `example-gateway` with the `Host` header `www.example.com` and request path `/login` will be routed to `example-svc` service on 8080

## Request Flow

- client prepares a HTTP request to `http://www.example.com`
- client DNS resolver queries and learns a mapping to one or more IPs associated with the Gateway
- client sends a request to Gateway IP
- reverse proxy receives the HTTP request and uses the `Host` header to match Gateway and HTTPRoute
- optionally reverse proxy performs request header and/or path matching based on rules of HTTPRoute
- optionally reverse proxy modifies request e.g. adds/removes headers based on rules of HTTPRoute
- reverse proxy forwards request to one or more backends
