# K8S Job

[back](README#workload)

- represents one-off task that runs to completion and stops
- creates 1..X pods, retries exec until a specified number completes with success
- deleting a `Job` cleans up it's pods, suspending will delete active pods until resumed
- usual case: one `Job` runs a single `Pod`
  - but can run multiple pods in parallel
- job labels are prefixed `batch.kubernetes.io/` for `job-name` and `controller-uid`
- fields `apiVersion`, `kind`, `metadata`
  - `.metadata.name` is part of name of pods (DNS label rules)
  - `.selector` should not be used but is optional
- `.spec`
  - `.template` (mandatory) pod template
    - must specify labels
    - must specify `RestartPolicy` `Never` or`OnFailure`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
        - name: pi
          image: perl:5.34.0
          command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```

- ...
  - computes PI to 2000 places and prints it out, takes ~10s
  - apply `kubectl apply -f job.yaml`
    - output `job.batch/pi created`
  - see job status `kubectl describe job pi`
  - see completed pods `kubectl get pods`
  - see logs `kubectl logs jobs/pi`

## Job types

- non-parallel
  - only one pod is started unless it fails
  - task complete as soon as the pod succeeds
- parallel with fixed completion count
  - `.spec.completions` non-zero positive number
  - task complete when completions number reached
  - with `.spec.completionMode="Indexed"` each pod gets an index
    - range: [0 ... `.spec.completions-1`]
- parallel with a work queue
  - no `.spec.completions`, defaults to `.spec.parallelism`
  - pods must coordinate work amongst themselves or an external service
    - e.g. a pod fetches a batch of up to N items from the work queue
  - each pod must see whether all peers are done and thus task is done
  - when any pod terminates with success no new pods are created
  - task completed when at least one pod succeeds and all pods are terminated
    - after first success pods are considered "done" and should be exiting
