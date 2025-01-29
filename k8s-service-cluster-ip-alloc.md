# K8S Service ClusterIP Allocation

[back](README#dns)

- services are an abstract way to expose an application running on a set of Pods
- service can have cluster-scoped virtual IP with `type: ClusterIP`
  - clients can connect to that address
  - k8s load balances traffic to that service across backing pods
- virtual IP assignment
  - dynamic - picks from configured IP range for `type: ClusterIP` services
  - static - specify manually from configured IP range
- fyi across cluster every service ClusterIP must be unique
  - i.e. static IP assignment of already allocated IP errors out

## Service Cluster IP Reservation

- sometimes services need to run on well-known IPs
  - e.g. a DNS service for the cluster
    - assuming cluster Service IP range `10.96.0.0/16`
    - assuming DNS Service IP to be `10.96.0.10`

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: CoreDNS
  name: kube-dns
  namespace: kube-system
spec:
  clusterIP: 10.96.0.10
  ports:
    - name: dns
      port: 53
      protocol: UDP
      targetPort: 53
    - name: dns-tcp
      port: 53
      protocol: TCP
      targetPort: 53
  selector:
    k8s-app: kube-dns
  type: ClusterIP
```

- problem: the `10.96.0.10` IP was not reserved
  - if other services already have this IP there will be an error
- ClusterIP range is divided based on `min(max(16, cidrSize / 16), 256)`
  - i.e. never less than 16 or more than 256 with a graduated step between them
  - dynamic IPs use upper range by default then lower range i.e. use lower range for static

| example | service IP range | range size         | band offset                                          | static band start | static band end | range end       |
| ------- | ---------------- | ------------------ | ---------------------------------------------------- | ----------------- | --------------- | --------------- |
| 1       | `10.96.0.0/24`   | `28 - 2 = 254`     | `min(max(16, 256/16), 256) = min(16, 256) = 16`      | `10.96.0.1`       | `10.96.0.16`    | `10.96.0.254`   |
| 2       | `10.96.0.0/20`   | `2^12 - 2 = 4094`  | `min(max(16, 4096/16), 256) = min(256, 256) = 256`   | `10.96.0.1`       | `10.96.1.0`     | `10.96.15.254`  |
| 3       | `10.96.0.0/16`   | `2^16 - 2 = 65534` | `min(max(16, 65536/16), 256) = min(4096, 256) = 256` | `10.96.0.1`       | `10.96.1.0`     | `10.96.255.254` |
