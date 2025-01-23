# K8S Service

[back](README#service)

- expose app in cluster (even split across multiple backends) as a single endpoint
- other aspect: for when pod A needs pod B where both are ephemeral and change, Service keeps track of that dependency
- Service API: abstraction for exposing pods over a network
- a service defines a logical set of endpoints each (usually) representing a pod along with access policy
- example: stateless image-processing backend with 3 replicas where clients don't need to know which replica currently is available etc.
- selector: targets set of pods by their labels (or use without selector, see below)
- if workload speaks HTTP: can choose an Ingress (is not a Service type but an entry point to a cluster)
  - or use Gateway API implementation instead
- service discovery: mechanism for viewing available EndpointSlice objects
- definition, e.g. set of pods listening on TCP 9376, labelled "app.kubernetes.io/name=MyApp"
  - (see yaml below)
  - creates: a Service named `my-service`
  - service uses: default ClusterIP service type
  - service targets: TCP port 9376 on any pod with the `app.kubernetes.io/name: MyApp` label
  - service is assigned: an IP address (cluster IP) used by the virtual IP address mechanism
  - service controller **continuously** scans matching pods and updates the set of EndpointSlices
  - fyi: `targetPort` is by default same as `port`
  - fyi: `protocol` is by default TCP

```yaml
apiVersion: v1
kind: Service
metadata:
name: my-service
spec:
selector:
    app.kubernetes.io/name: MyApp
ports:
    - protocol: TCP
    port: 80
    targetPort: 9376
```

## Ports

- hint: multiple ports can be exposed
- hint: ports should have names (see yaml below)
  - the http-web-svc name will work even when containerPort changes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app.kubernetes.io/name: proxy
spec:
  containers:
    - name: nginx
      image: nginx:stable
      ports:
        - containerPort: 80
          name: http-web-svc

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app.kubernetes.io/name: proxy
  ports:
    - name: name-of-service-port
      protocol: TCP
      port: 80
      targetPort: http-web-svc
```

## EndpointSlice

- object representing a slice of backend network endpoints for a service
- fyi k8s tracks and auto-scales endpoint slices based on endpoints inside

## Without selectors

- examples:
  - external DB cluster in PROD but local DB locally
  - service pointing to another service in different namespace or cluster
  - migrating workload to k8s and testing with only a portion of backends first
- service can be defined without pod selectors
  - (see yaml below)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
```

- ...
  - EndpointSlices are now not auto-created
  - add them manually (see yaml below)

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name:
    # by convention, use the name of the Service as a prefix for the name of the EndpointSlice
    my-service-1
  labels:
    # You should set the "kubernetes.io/service-name" label. Set its value to match the name of the Service
    kubernetes.io/service-name: my-service
addressType: IPv4
ports:
  - name: http # should match with the name of the service port defined above
    appProtocol: http
    protocol: TCP
    port: 9376
endpoints:
  - addresses:
      - "10.4.5.6"
  - addresses:
      - "10.1.2.3"
```

- ...
  - EndpointSlice names must be unique in a namespace
  - link slice to service by `kubernetes.io/service-name` on the slice
  - (fyi there are more rules here but not relevant for this docs)

NEXT: https://kubernetes.io/docs/concepts/services-networking/service/#application-protocol
