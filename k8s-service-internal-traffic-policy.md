# K8S Service Internal Traffic Policy

[back](README#dns)

- pods running on the same node should be able to communicate inside the node
- SITP enables internal traffic for routes inside a node

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
  internalTrafficPolicy: Local
```

- ...
  - kube-proxy filters endpoints it routes to based on `spec.internalTrafficPolicy`
  - `Local` - only node local endpoints are considered
  - `Cluster` - (or default) considers all endpoints
