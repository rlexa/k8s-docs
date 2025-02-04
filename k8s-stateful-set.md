# K8S StatefulSet

[back](README#workload)

- runs a group of pods, maintains a sticky identity for each of those pods
  - manages deployment, scaling, guarantees ordering and uniqueness of these pods
- why
  - stable unique network identifiers
  - stable persistent storage
  - ordered graceful deployment and scaling
  - ordered automated rolling updates
- like `Deployment` creates pods same spec but those pods are not interchangeable
  - pods are created from the same spec, but are not interchangeable
    - each has persistent ID maintained across rescheduling
  - `.spec`
    - `.volumeClaimTemplates` can be set `PersistentVolumeClaim`
      - `StorageClass` for volume claim is set up to use dynamic provisioning
      - ...or cluster already has `PersistentVolume` with correct `StorageClass`
        - ...and sufficient available storage space
    - `.minReadySeconds` min time for pod to get ready (default 0)
    - `.ordinals` ordinal assignment
      - `.start` if set, pods will get ordinals that to start+`.spec.replicas - 1`
    - `.podManagementPolicy` can relay ordering guarantees
      - `OrderedReady` (default)
      - `Parallel` launch/terminate pods in parallel (scaling only)
    - `.updateStrategy`
      - `RollingUpdate` default
        - [see more details](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#partitions)
      - `OnDelete` won't auto-update pods
        - must manually delete pods to trigger creation with new templates
    - `.replicas` default 1
- limitations
  - storage for a pod must be `PersistentVolume` based on storage class or be pre-provisioned by an admin
  - deleting scaling down a `StatefulSet` won't delete the associated volumes
    - ensures data safety (more valuable than an auto purge of all related resources)
  - currently require headless `Service` for network identity of pods - must create this `Service`
  - don't guarantee pod termination on `StatefulSet` deletion
    - _fyi: ordered and graceful pod termination: scale `StatefulSet` down to 0 before deletion_
  - sometimes broken state requires manual intervention to repair
    - when using rolling updates with default pod management policy `OrderedReady`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
    - port: 80
      name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # has to match .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 3 # by default is 1
  minReadySeconds: 10 # by default is 0
  template:
    metadata:
      labels:
        app: nginx # has to match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
        - name: nginx
          image: registry.k8s.io/nginx-slim:0.24
          ports:
            - containerPort: 80
              name: web
          volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: "my-storage-class"
        resources:
          requests:
            storage: 1Gi
```

- ...
  - here `ReadWriteOnce` access mode for simplicity
    - _fyi: PROD should use `ReadWriteOncePod` instead_
  - headless `Service` named `nginx` controlling network domain
  - `StatefulSet` named `web` with 3 replicas of `nginx` container
    - _fyi: will be launched in unique pods_
  - `volumeClaimTemplates` provides stable storage using `PersistentVolume`s
    - provisioned by a `PersistentVolume` provisioner
  - `.spec.selector` must match of it's `.spec.template.metadata.labels.`
    - or results in validation error during `StatefulSet` creation

## Pod Identity

- `StatefulSet` pods have unique identity
  - consists of an ordinal, a stable network identity, and stable storage
    - default: pods assigned ordinals from 0 to replicas-1
    - controller will add pod label `apps.kubernetes.io/pod-index`
  - identity sticks to pod regardless of node it's (re)scheduled on

### Stable Network ID

- `StatefulSet` pod gets hostname from StatefulSet name and ordinal
  - format `$(statefulset name)-$(ordinal)` e.g. `web-1`
- headless `Service` can control domain of pods
  - format `$(service name).$(namespace).svc.cluster.local`
    - `cluster.local` is default local cluster domain
  - each pod gets matching DNS subdomain
    - format `$(podname).$(governing service domain)`
    - governing service defined by `serviceName` field on `StatefulSet`
- fyi: "negative caching" (normal in DNS) means that results of previous failed lookups are remembered and reused, even after the pod is running, for at least a few seconds
  - i.e. wait for a bit when a pod is created before trying to discover it
  - i.e. if immediate discovery needed then
    - query Kubernetes API directly (e.g. `watch`) without relying on DNS lookups
    - or decrease caching time in Kubernetes DNS provider
      - e.g. edit config map for CoreDNS (default 30 seconds)
- example:
  - cluster domain: `cluster.local`
  - service ns/name: `default/nginx`
  - statefulset ns/name: `default/web`
  - statefulset domain: `nginx.default.svc.cluster.local`
  - pod name: `web-{0..N-1}`
  - pod DNS: `web-{0..N-1}.nginx.default.svc.cluster.local`

### Stable Storage

- each `Pod` gets a `PersistentVolumeClaim` per `VolumeClaimTemplate`
- `nginx` example above:
  - each `Pod` gets a `PersistentVolume`
    - of `StorageClass` `my-storage-class` and 1 GiB of provisioned storage
- _warning: `PersistentVolume`s are not deleted with pods or `StatefulSet`_

### Stable Labels

- `statefulset.kubernetes.io/pod-name` is set to pod label
  - allows attachment of a `Service` to a specific pod
- `apps.kubernetes.io/pod-index` is set to pod ordinal index

## Deployment and Scaling

- pods are created sequentially 0..N-1
  - on scaling all of predecessors must be `Running` and `Ready` first
- pods are deleted sequentially N-1..0
  - on termination all successors must be completely shutdown
- _warning: don't set `pod.Spec.TerminationGracePeriodSeconds` to 0_

## PersistentVolumeClaim Retention

- `.spec.persistentVolumeClaimRetentionPolicy` controls PVC deletion
  - **warning: must enable `StatefulSetAutoDeletePVC` feature gate**
  - `whenDeleted` configures on delete
  - `whenScaled` when scaling down
  - for each above:
    - `Delete` deletes PVCs on pod or set deletion
    - `Retain` (default) does not delete PVCs
- `StatefulSet` controller adds owner references to it's PVCs
