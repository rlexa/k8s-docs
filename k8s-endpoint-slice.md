# K8S EndpointSlice

[back](README#entities)

- EndpointSlice API lets a Service scale it's backends and lets a cluster update it's healthy backends list
- basically a way to track network endpoints
- contains references to a set of network endpoints
- fyi more scalable alternative to Endpoints
  - which was simple way but with limitations (especially scaling)
- fyi mostly owned by a service see `kubernetes.io/service-name`
- fyi auto-created for a service if selector is specified
  - references all pods matching the selector
  - groups network endpoints by unique combo of protocol+port+service.name
    - fyi must be a valid DNS subdomain name
- fyi works as source-of-truth for kube-proxy when routing internally
- fyi see `endpointslice.kubernetes.io/managed-by` for additional non-auto-control-plane management

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: example-abc
  labels:
    kubernetes.io/service-name: example
addressType: IPv4
ports:
  - name: http
    protocol: TCP
    port: 80
endpoints:
  - addresses:
      - "10.1.2.3"
    conditions:
      ready: true
    hostname: pod-1
    nodeName: node-1
    zone: us-west2-a
```

- ...
  - this example EndpointSlice is owned by "example" service
  - fyi default max. 100 endpoints per EndpointSlice (see `--max-endpoints-per-slice`)
  - `addressType`
    - IPv4, IPv6, FQDN
    - fyi: create multiple EndpointSlice when e.g. v4 and v6 needed together
  - `conditions`
    - `ready` maps to pod's `ready`
    - `serving` same as `ready` but stays `true` while terminating
    - `terminating` for a pod it means it has a deletion timestamp stamp
  - `nodeName` name of the node this endpoint is on (topology)
  - `zone` name of the zone this endpoint is on (topology)

## Distribution

- each EndpointSlice has a set of ports applying to all endpoints within the resource
- control plane process:
  - go through existing EndpointSlices
    - remove endpoints that are no longer needed
    - update matching endpoints that changed
  - go through changed (above) EndpointSlices
    - fill up with any new endpoints when needed
  - if still endpoints to add
    - try and fill them into non-changed EndpointSlices
    - or create new ones
