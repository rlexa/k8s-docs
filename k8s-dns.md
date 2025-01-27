# K8S DNS

[back](README#dns)

- workload discovers services in a cluster via DNS
  - services and pods get auto-DNS
- k8s creates DNS (use instead of IPs) for services and pods
- default: pod DNS search list includes pod's namespace and cluster's default domain
- service namespace
  - e.g. pod in `test` namespace, service `data` in `prod` namespace
    - query `data` returns nil because none in `test`
    - query `prod.data` returns the service
- fyi: see pod's kubelet-auto-configured `/etc/resolv.conf` for more possibilities
- service DNS
  - non-headless i.e. normal services get A or AAAA names
    - depends on IP families of service
    - e.g. `my-svc.my-namespace.svc.cluster-domain.example`
      - this then resolves to the IP in the cluster
  - headless services are basically the same but resolve to a set of IPs
  - SRV records are created for named ports
    - e.g. `_port-name._port-protocol.my-svc.my-namespace.svc.cluster-domain.example`
      - this resolves to the port number and domain (or set for headless)
- pod DNS
  - hostname is `metadata.name` (from WITHIN the pod)
    - pod spec has optional overriding `hostname` and `subdomain`
    - e.g. `hostname=foo` and `subdomain=bar` in namespace `ns` resolves to:
      - `foo.bar.ns.svc.cluster.local` (from within the pod)
  - fyi: if headless service in same namespace as pod exists with the same name as subdomain, A/AAAA records for pod's fully qualified hostname are created

```yaml
apiVersion: v1
kind: Service
metadata:
  name: busybox-subdomain
spec:
  selector:
    name: busybox
  clusterIP: None
  ports:
    - name: foo # name is not required for single-port Services
      port: 1234
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox1
  labels:
    name: busybox
spec:
  hostname: busybox-1
  subdomain: busybox-subdomain
  containers:
    - image: busybox:1.28
      command:
        - sleep
        - "3600"
      name: busybox
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox2
  labels:
    name: busybox
spec:
  hostname: busybox-2
  subdomain: busybox-subdomain
  containers:
    - image: busybox:1.28
      command:
        - sleep
        - "3600"
      name: busybox
```

- ...
  - service name: `busybox-subdomain`
  - pods spec.subdomain `busybox-subdomain`
  - first pod FQDN resolves to:
    - `busybox-1.busybox-subdomain.my-namespace.svc.cluster-domain.example`
    - DNS serves A and/or AAAA records at that name pointing to the pod's IP
  - both pods `busybox1` and `busybox2` will have their own address records
  - fyi EndpointSlice can specify DNS hostname for endpoint addresses, with its IP
  - fyi A/AAAA for pods not generated without `hostname`
    - pod without `hostname` but with `subdomain` will only create A/AAAA for headless services
  - `hostname` command inside the pod returns `busybox-1`
  - `hostname -fqdn` command returns the full name
    - `setHostnameAsFQDN: true` would return full with `hostname` too

## Pod DNS Policy

- `dnsPolicy` in pod spec can be set per pod
  - `Default` inherits name resolution of node pod is running on
  - `ClusterFirst` unknown queries e.g. `www.kubernetes.io` are forwarded upstream (???)
  - `ClusterFirstWithHostNet` for pods running with `hostNetwork` (???)
  - `None` ignores k8s DNS and uses `spec.dnsConfig` instead
  - **fyi: default is `ClusterFirst`, not `Default` (wtf?!)**

## Pod DNS Config

- `dnsConfig` for more (optional) control per pod
  - `nameservers` a list of max 3 IPs used as DNS servers for pod
  - `searches` a list of max 32 search domains for hostname lookup in the pod
  - `options` list of object with `name` and optional `value` property, merged into options generated from DNS policy

```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: dns-example
spec:
  containers:
    - name: test
      image: nginx
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 192.0.2.1 # this is an example
    searches:
      - ns1.svc.cluster-domain.example
      - my.dns.search.suffix
    options:
      - name: ndots
        value: "2"
      - name: edns0
```

- ...
  - container `test`'s `/etc/resolv.conf`:

```
nameserver 192.0.2.1
search ns1.svc.cluster-domain.example my.dns.search.suffix
options ndots:2 edns0
```

- ...
  - for IPv6 do: `kubectl exec -it dns-example -- cat /etc/resolv.conf`, should get

```
nameserver 2001:db8:30::a
search default.svc.cluster-domain.example svc.cluster-domain.example cluster-domain.example
options ndots:5
```

## On Windows Nodes

- `ClusterFirstWithHostNet` not supported, all names with `.` are FQDNs
- multiple DNS resolvers can be used, use powershell `Resolve-DNSName` for resolving
- can have only a single DNS suffix (???)
