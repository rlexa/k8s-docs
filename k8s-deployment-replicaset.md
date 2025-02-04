# K8S Deployment / ReplicaSet

[back](README#workload)

## Deployment

- manages set of pods to run an app workload (usually stateless)
- provides declarative updates for pods and `ReplicaSet`s
- is a description of the desired state (controller then makes it happen)
- _caution: do not manage `ReplicaSet`s created by `Deployment`_

### Deployment spec

- needs: `.apiVersion`, `.kind`, `.metadata` and `.spec`
- `.metadata.name` is part of naming of the `Deployment`'s pods
  - i.e. DNS label restricted
- `.spec`
  - `.template` (mandatory), same schema as `Pod`
    - must specify labels and restart policy
      - don't overlap with labels from other controllers
      - `restartPolicy` is per default `Always` (and also only allowed)
  - `.selector` (mandatory) targets pods for this `Deployment`
    - must match `.spec.template.metadata.labels`
    - _warning: do not change labels and do not create new pods with similar labels_
  - `.replicas` number of desired pods (default 1)
    - don't set if autoscaler is configured
  - `.strategy` how to replace old pods by new
    - `Recreate` kill all old then create new
    - `RollingUpdate` (default) for graceful scaling
      - `maxUnavailable` number or percentage
      - `maxSurge` number or percentage of max over-desired
  - `.progressDeadlineSeconds` until `Deployment` progress is deemed a failure
    - default 600
    - must be greater than `minReadySeconds`
  - `.minReadySeconds` for a pod to become available on creation
    - default 0
  - `.revisionHistoryLimit` default 10
  - `.paused` boolean, on `true` changes to template won't trigger rollout

### Create Deployment

- creates `ReplicaSet` with `nginx` pods

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.14.2
          ports:
            - containerPort: 80
```

- ...
  - `Deployment` named `nginx-deployment` is created
  - `name` will become basis for `ReplicaSet`s and `Pod`s created later
  - creates a `ReplicaSet` that creates three replicated `Pod`s
  - `.spec.selector` field defines `ReplicaSet` matcher for `Pod`s to manage
    - here a label defined in `Pod` template (`app: nginx`)
  - fyi: `.spec.selector.matchLabels` field is a map of `{key,value}` pairs
    - single `{key,value}` in the map is equivalent to:
      - an element of `matchExpressions`
        - key field is "key"
        - operator is "In"
        - values array contains only "value"
  - `.spec.template` field
    - sreates one container named `nginx`
- **WARNING: must specify selector and template**
  - **WARNING: must not overlap labels and selectors with other Deployments**
- **WARNING: only changes to `.spec.template` trigger rollouts**
- create via `kubectl apply -f my-nginx-deployment.yaml`
- check with `kubectl get deployments`
  - `NAME` lists `Deployment` names in the namespace
  - `READY` shows replicas (`ready/desired` format)
    - e.g. when immediate: 0/3, later: 3/3
  - `UP-TO-DATE` shows replicas that have been updated to current state
  - `AVAILABLE` shows replicas available to users
  - `AGE` shows time app has been running
- check rollout `kubectl rollout status deployment/nginx-deployment`
  - output e.g. `Waiting for rollout to finish: 2 out of 3 new replicas have been updated... deployment "nginx-deployment" successfully rolled out`
- list replicas with `kubectl get rs`
  - name format: `[DEPLOYMENT-NAME]-[HASH]` e.g. `nginx-deployment-75675f5897`
- list generated pod labels with `kubectl get pods --show-labels`, output e.g.:

```
NAME                                READY     STATUS    RESTARTS   AGE       LABELS
nginx-deployment-75675f5897-7ci7o   1/1       Running   0          18s       app=nginx,pod-template-hash=75675f5897
nginx-deployment-75675f5897-kzszj   1/1       Running   0          18s       app=nginx,pod-template-hash=75675f5897
nginx-deployment-75675f5897-qqcnn   1/1       Running   0          18s       app=nginx,pod-template-hash=75675f5897
```

### Update Deployment

- case: update image in pods
  - `kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1`
  - or `kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1`
    - `deployment/nginx-deployment` is the `Deployment`
    - `nginx` is the `Container`
  - or update the yaml and `kubectl edit deployment/nginx-deployment`
  - check rollout with `kubectl rollout status deployment/nginx-deployment`
  - check info with `kubectl get deployments`
    - or details with `kubectl describe deployments`
  - check replicas with `kubectl get rs`
  - check pods with `kubectl get pods`
- fyi `Deployment` ensures that max. 25% (default) Pods are down while updated
- fyi `Deployment` ensures that max. 125% (default) Pods are active
  - so for case above: creates new Pod, deletes old Pod, repeats

#### Label selectors

- try to plan up front, don't change afterwards
- **WARNING: `apps/v1` `Deployment`s label selector is immutable after creation**
- selector additions
  - pod template labels in `Deployment` spec must be updated with new label
    - otherwise: validation error
  - non-overlapping change i.e. new selector does not select old `ReplicaSet`s and `Pod`s
    - i.e. all old `ReplicaSet`s are orphaned and new are created
- selector updates
  - changes existing value in selector key
  - results in same behavior as additions
- selector removals
  - removes existing key from `Deployment` selector
  - no changes in pod template labels required
  - existing `ReplicaSet`s not orphaned, new ones not created
  - _warning: removed label still exists in existing Pods and ReplicaSets_

### Deployment rollback

- `Deployment` rollout history exists by default
  - a revision is created on rollout (template change), not e.g. on scaling
- e.g. mistake in version 1.16.1 done via:
  - `kubectl set image deployment/nginx-deployment nginx=nginx:1.161`
    - output: `deployment.apps/nginx-deployment image updated`
    - rollout stuck, check:
      - `kubectl rollout status deployment/nginx-deployment`
      - output: `Waiting for rollout to finish: 1 out of 3 new replicas have been updated...`
      - stop watcher with Ctrl-C
    - check replicas `kubectl get rs`
      - will probably show e.g. 3 old replicas and 1 new desired replica
    - check pods `kubectl get pods`
      - will probably show e.g. 3 old and non-ready 1 with `ImagePullBackOff`
    - check deployment details `kubectl describe deployment`
  - now check revisions `kubectl rollout history deployment/nginx-deployment`
    - will show `CHANGE-CAUSE` column last entry as:
      - `kubectl set image deployment/nginx-deployment nginx=nginx:1.161`
    - check details of revision `kubectl rollout history deployment/nginx-deployment --revision=2`
  - now rollback to previous `kubectl rollout undo deployment/nginx-deployment`
    - or specific `kubectl rollout undo deployment/nginx-deployment --to-revision=2`

### Deployment scaling

- scale by `kubectl scale deployment/nginx-deployment --replicas=10`
- assuming horizontal Pod autoscaling is enabled in cluster
  - can set up autoscaler for Deployment, choose min/max pod number
  - `kubectl autoscale deployment/nginx-deployment --min=10 --max=15 --cpu-percent=80`

#### Proportional scaling

- supports multiple versions of an application at the same time
- case: `Deployment` `replicas`=10, `maxSurge`=3, and `maxUnavailable`=2
  - `kubectl get deploy` shows 10 everything

```
NAME                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment     10        10        10           10          50s
```

- ...
  - shows current state, 10 everything
  - update to new incorrect image from inside the cluster
    - `kubectl set image deployment/nginx-deployment nginx=nginx:sometag`
    - output: `deployment.apps/nginx-deployment image updated`
    - image update starts a new rollout with `ReplicaSet` nginx-deployment-1989198191
      - becomes blocked due to `maxUnavailable`
    - check `kubectl get rs`:

```
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-1989198191   5         5         0         9s
nginx-deployment-618515232    8         8         8         1m
```

- ...
  - now new scaling request for the `Deployment` comes along
  - autoscaler increments the `Deployment` replicas to 15
  - controller needs to decide where to add these new 5 replicas
  - if not proportional scaling: add all 5 in new `ReplicaSet`
  - with proportional scaling: spread new replicas across all `ReplicaSet`s
    - in current example: +3 replicas in old `ReplicaSet`, +2 in new
  - check `kubectl get deploy` and `kubectl get rs`

```
NAME                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment     15        18        7            8           7m
```

```
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-1989198191   7         7         0         7m
nginx-deployment-618515232    11        11        11        7m
```

### Deployment rollout pause/resume

[see here](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#pausing-and-resuming-a-deployment)

### Deployment stats

[see here](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#deployment-status)

## ReplicaSet

- maintains stable set of replica pods running at any given time
- usually: just use `Deployment` which auto-manages `ReplicaSet`s
  - use directly for custom orchestration requirements
  - use directly when no updates needed at all
- fields: `apiVersion`, `kind`, `metadata`
  - `.metadata.name` is part of pods names
  - `.spec`
    - `.replicas` number for how many pods it should be maintaining (default 1)
    - `.selector` specifies how to identify pods it can acquire
    - `.template` pod template
      - `labels` required e.g. `tier: frontend`
      - `.spec.restartPolicy` must be `Always` (default)
      - `.metadata.labels` must match `.spec.selector`
- `pod.metadata.ownerReferences` links to owning `ReplicaSet`
  - every pod has this field
  - `ReplicaSet` discovers pods by selector and this field being empty

```yaml
# frontend.yaml

apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  # modify replicas according to your case
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
        - name: php-redis
          image: us-docker.pkg.dev/google-samples/containers/gke/gb-frontend:v5
```

- ...
  - apply `kubectl apply -f frontend.yaml`
  - check replicas `kubectl get rs`
  - check `ReplicaSet` state `kubectl describe rs/frontend`
  - check pods `kubectl get pods`
    - e.g. output contains `frontend-gbgfx` pod
    - check ownership `kubectl get pods frontend-gbgfx -o yaml`
      - will show `ownerReferences` nested object among other things

### Non-Template Pods

- bare pods can be created
  - recommended: make sure they don't have labels matching selector `ReplicaSet`s
    - reason: a `ReplicaSet` selects pods by template by is not limited to

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
  labels:
    tier: frontend
spec:
  containers:
    - name: hello1
      image: gcr.io/google-samples/hello-app:2.0
---
apiVersion: v1
kind: Pod
metadata:
  name: pod2
  labels:
    tier: frontend
spec:
  containers:
    - name: hello2
      image: gcr.io/google-samples/hello-app:1.0
```

- ...
  - these pods don't have an owner reference and match frontend `ReplicaSet`
    - they will be immediately acquired by it
    - they will be immdetiately terminated being over max of it's desired count
    - check `kubectl get pods`

```
NAME             READY   STATUS        RESTARTS   AGE
frontend-b2zdv   1/1     Running       0          10m
frontend-vcmts   1/1     Running       0          10m
frontend-wtsmm   1/1     Running       0          10m
pod1             0/1     Terminating   0          1s
pod2             0/1     Terminating   0          1s
```

- ...
  - however if these pods are created first and `ReplicaSet` afterwards
    - then they will stay active and there will be plus single frontend pod

### More specifics

[see more](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/#working-with-replicasets)
