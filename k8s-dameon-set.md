# K8S DaemonSet

[back](README#workload)

- defines pods that provide node-local facilities
- `DaemonSet`ensures that all (or some) `Node`s run a copy of a Pod
  - these pods are tied to `Node` lifecycle
  - deleting DS will also clean up these pods
- cases:
  - running a cluster storage daemon on every node
  - running a logs collection daemon on every node
  - running a node monitoring daemon on every node
- usually `DaemonSet` per daemon type
  - can have multiple per type with different flags/configs

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
        # these tolerations are to have the daemonset runnable on control plane nodes
        # remove them if your control plane nodes should not run pods
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
        - key: node-role.kubernetes.io/master
          operator: Exists
          effect: NoSchedule
      containers:
        - name: fluentd-elasticsearch
          image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
          resources:
            limits:
              memory: 200Mi
            requests:
              cpu: 100m
              memory: 200Mi
          volumeMounts:
            - name: varlog
              mountPath: /var/log
      # it may be desirable to set a high priority class to ensure that a DaemonSet Pod
      # preempts running Pods
      # priorityClassName: important
      terminationGracePeriodSeconds: 30
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
```

- ...
  - `DaemonSet` that runs `fluentd-elasticsearch` Docker image
  - fields: `apiVersion`, `kind`, `metadata`
  - `.spec`
    - `.template` (mandatory) pod template
      - must have labels
      - `.restartPolicy` is the only allowed default `Always`
      - `.nodeSelector` can specify matcher for nodes
      - `.affinity` can specify matcher for nodes by affinity
      - _warning: without both above will create on all nodes_
    - `.selector` for matching pods
      - _warning: can't be mutated_
      - has `matchLabels` (and) `matchExpressions`
      - must match `.spec.template.metadata.labels`
- [can configure scheduling see here](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/#how-daemon-pods-are-scheduled)
- [can configure taints e.g. run on not-ready nodes see here](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/#taints-and-tolerations)

## Communication

- pods in `DaemonSet` can communicate via:
  - `Push`: pods send updates to another service (no client)
  - `NodeIP and Known Port`: reachable on `hostPort` and node IP
  - `DNS`: headless `Service` with same pod selector then discover `endpoints`
  - `Service`: with same pod selector (no way to reach specific `Node`)

## DaemonSet Update

- on node label change DS will add pods to new nodes and delete old
- `kubectl` with `--cascade=orphan` on DS delete leaves pods alive
  - and will be reused on start of new DS with same selector

## DaemonSet Alternatives

- init script
  - daemon processes can be started on a node (e.g. `init`, `upstartd`, `systemd`)
  - advantages of using `DaemonSet` instead:
    - can monitor and manage logs for daemons same way as apps
    - same config and tools (e.g. pod templates, kubectl) for daemons and apps
    - increased isolation in containers with resource limits
- bare pods
  - possible to create pods directly specifying node to run on
  - advantages of using `DaemonSet` instead:
    - replaces deleted or terminated pods e.g. on kernel update etc.
- static pods
  - can create pods via file in a directory watched by Kubelet
    - these static pods can't be managed by `kubectl`
    - _warning: static pods can get deprecated in the future_
- deployments
  - both create pods which are not expected to terminate
  - use deployments for stateless services where host selection is not important
  - use DS where pod needs to run on a certain host or for node-level functionality
