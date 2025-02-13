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
  - `.completions` default 1 (see job types below)
    - non-zero positive number
  - `.completionMode`
    - `NonIndexed` (default) complete when `.spec.completions` successes
    - `Indexed` pods have completion index from 0..`.spec.completions-1`
      - pod annotation: `batch.kubernetes.io/job-completion-index`
      - pod label: `batch.kubernetes.io/job-completion-index`
        - needs `PodIndexLabel` feature gate
      - part of pod hostname with `$(job-name)-$(index)`
      - in container via envvar `JOB_COMPLETION_INDEX`
  - `.parallelism` default 1 (see job types below)
    - non-zero positive number (0 basically pauses job until raised again)

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

## Handling pod/container failure

- container fail
  - `.spec.template.spec.restartPolicy = "OnFailure"`
    - pod stays on node but container re-run
      - i.e. make sure app is restartable in new container
        - else set `.spec.template.spec.restartPolicy = "Never"`
- pod fail (e.g. node gets upgraded, rebooted, deleted etc.)
  - job controller starts a new pod i.e. app is restartable in new pod
    - i.e. handle temp files, locks, incomplete output etc.
  - `.spec.backoffLimit` (default, default value 6)
    - see how backoff counter works (stagger-timed restarts)
  - `.spec.backoffLimitPerIndex` instead to count per-index independently
    - needs `JobBackoffLimitPerIndex` feature gate
- _warning: may get started twice even if all set:_
  - `.spec.parallelism = 1`
  - `.spec.completions = 1`
  - `.spec.template.spec.restartPolicy = "Never"`
- _warning: may start multiple pods i.e. code for concurrency if both:_
  - `.spec.parallelism` > 1
  - `.spec.completions` > 1
- case: fail job after some retries due to logical error in configuration etc.
  - set `.spec.backoffLimit` to retry-till-considered-failed-job (default 6)
    - uses backoff delay staggering i.e. 10s, 20s, 40s etc.
    - number of retries calculation:
      - pods with `.status.phase = "Failed"`
      - if `restartPolicy = "OnFailure"` then retry count in all containers of pods with:
        - `.status.phase` `Pending` or `Running`
    - job fails if either condition reaches `.spec.backoffLimit`
- fyi: set `restartPolicy = "Never"` for debugging a job or use logging
  - else with `restartPolicy = "OnFailure"` debugging is difficult as pod gets killed when backoff limit is reached
- fyi: [see backoff limit per index details](https://kubernetes.io/docs/concepts/workloads/controllers/job/#backoff-limit-per-index)

### Pod failure policy

- `.spec.podFailurePolicy` enables cluster to handle pod failures
  - based on container exist codes
  - and pod condiitions
- i.e. sometimes `.spec.backoffLimit` is not enough control e.g.:
  - optimize costs by avoiding unnecessary Pod restarts
    - terminate job as soon as one pod fails with specific exit code
  - guarantee job ends even if disrupted
    - can ignore pod failures caused by e.g.:
      - preemption, API-initiated eviction, taint-based eviction
    - i.e. does't count towards `.spec.backoffLimit` limit of retries

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-pod-failure-policy-example
spec:
  completions: 12
  parallelism: 3
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: main
          image: docker.io/library/bash:5
          command: ["bash"] # simulate bug, triggers FailJob action
          args:
            - -c
            - echo "Hello world!" && sleep 5 && exit 42
  backoffLimit: 6
  podFailurePolicy:
    rules:
      - action: FailJob
        onExitCodes:
          containerName: main # optional
          operator: In # one of: In, NotIn
          values: [42]
      - action: Ignore # one of: Ignore, FailJob, Count
        onPodConditions:
          - type: DisruptionTarget # indicates Pod disruption
```

- ...
  - first there is rule of pod failure policy: `FailJob`
    - specifies: job marked failed if main container fails with 42
  - container `main` rules:
    - exit 0 means success
    - exit 42 means entire job failed
    - exit anything else fails container and pod, normal backoff restart process
  - second rule of pod failure policy: `Ignore`
    - exclude pod disruptions from counting towards `backoffLimit`
  - fyi: `restartPolicy: Never` means no restarting `main` in this pod
  - fyi: if job fails all pending/running pods will be terminated
- requirements
  - if `.spec.podFailurePolicy` then set pod `.spec.restartPolicy: Never`
  - `.spec.podFailurePolicy.rules`: executed in order, first fail stops
  - see `spec.podFailurePolicy.rules[*].onExitCodes.containerName`
    - targets a specific container (else all containers)
    - when specified must match one of pod's container or `initContainer`
  - `spec.podFailurePolicy.rules[*].action` values
    - `FailJob` mark job as failed, kill all pods
    - `Ignore` don't increment `backoffLimit`, replace pod
    - `Count` default way, increment `backoffLimit`
    - `FailIndex` avoid unnecessary retries within index of failed pod
      - needs backoff limit pre index settings

### Success policy

- needs `JobSuccessPolicy` feature gate
- indexed job: use `.spec.successPolicy`, based on pods that succeed
- default: job succeeds when `.spec.completions` count succeeds
- alternative cases:
  - simulations with different param might not need all of them to succeed
  - leader-worker pattern (only leader success is enough) e.g. PyTorch
- `.spec.successPolicy`
  - policy can handle job success based succeeded pods
  - with only `succeededIndexes`: all these indexes must succeed
    - must be list of intervals between 0 and `.spec.completions-1`
  - with only `succeededCount`: number of succeeded indexes for big success
  - when both of above:
    - succeeded indexes from subset of indexes in `succeededIndexes` reaches `succeededCount`
- fyi: multiple rules in `.spec.successPolicy.rules` evaluated in order
- fyi: `.spec.backoffLimit` and `.spec.podFailurePolicy` evaluated before success policies

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-success
spec:
  parallelism: 10
  completions: 10
  completionMode: Indexed # Required for the success policy
  successPolicy:
    rules:
      - succeededIndexes: 0,2-3
        succeededCount: 1
  template:
    spec:
      containers:
        - name: main
          image: python
          # Provided that at least one of the Pods with 0, 2, and 3 indexes has succeeded,
          # the overall Job is a success.
          command:
            - python3
            - -c
            - |
              import os, sys
              if os.environ.get("JOB_COMPLETION_INDEX") == "2":
                sys.exit(0)
              else:
                sys.exit(1)
      restartPolicy: Never
```

- ...
  - both `succeededIndexes` and `succeededCount` specified
  - success when either specified indexes 0, 2, or 3 succeed
  - on success: job `SuccessCriteriaMet` condition equals `SuccessPolicy`
    - will terminate all rest pods and go to `Complete` condition

### Termination and cleanup

- job completes, pods not created anymore but usually also not deleted
  - i.e. for checking logs and diagnostics, incl. job object itself
  - i.e. user has to delete the pods manually
    - `kubectl delete jobs/pi` or `kubectl delete -f ./job.yaml`
      - this way pods are also deleted
- default job completion process
  - pod fails `restartPolicy=Never` or container exits in error `restartPolicy=OnFailure`
  - job defers to `.spec.backoffLimit` process
  - when `.spec.backoffLimit` reached job marked failed
    - any running pods will be terminated
- another alternative: active deadline
  - set `.spec.activeDeadlineSeconds`
    - applies to the duration of the job (not pods etc.)
  - after deadline reached job fails with `type: Failed` and `DeadlineExceeded`
  - fyi: `.spec.activeDeadlineSeconds` precedes `.spec.backoffLimit`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-with-timeout
spec:
  backoffLimit: 5
  activeDeadlineSeconds: 100
  template:
    spec:
      containers:
        - name: pi
          image: perl:5.34.0
          command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
```

- ...
  - fyi: both job and pod have `activeDeadlineSeconds`
  - fyi: `restartPolicy` applies to pod, not job
    - there is no auto job restart after `type: Failed`
    - i.e. job termination after `.spec.activeDeadlineSeconds` and `.spec.backoffLimit` result in a permanent Job failure
      - _warning: requires manual intervention to resolve_

### Terminal job conditions

- two terminal states/conditions
  - `Complete`
    - succeeded pods >= `.spec.completions`
    - `.spec.successPolicy` criteria met
  - `Failed`
    - failed pods > `.spec.backoffLimit`
    - job runtime > `.spec.activeDeadlineSeconds`
    - indexed job with `.spec.backoffLimitPerIndex` has failed indexes.
    - failed indexes in job > `spec.maxFailedIndexes`
    - failed pod matches rule in `.spec.podFailurePolicy` with `FailJob`
- terminal conditions usually only set after all pods terminated
  - process
    - job meets fail or success criteria
    - set `FailureTarget` or `SuccessCriteriaMet` condition to job
      - _fyi: can be used to determine ack/nak without waiting on terminal condition_
        - e.g. for asap job replacement (but will start in parallel so think about resources)
    - this triggers pod termination
      - `terminationGracePeriodSeconds` can delay termination

### Job auto clean up

- jobs are usually not cleaned up (except if managed e.g. by `CronJob`)
- `.spec.ttlSecondsAfterFinished` TTL controller alternative
  - cascade deleted job i.e. pods etc. delted together with job
  - unset means no auto-delete, 0 means immediate auto-delete
- _recommended: do use TTL_
  - reason: `orphanDependents` is default for unmanaged jobs
    - unmanaged jobs are directly created jobs (i.e. not `CronJob`)
    - i.e. pods will become orphans and stay in system
      - eventually garbage collected, still can cause problems

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-with-ttl
spec:
  ttlSecondsAfterFinished: 100
  template:
    spec:
      containers:
        - name: pi
          image: perl:5.34.0
          command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
```

- ...
  - job `pi-with-ttl` auto deleted 100s after finish

## Job patterns

- job object can process set of independent but related work items
  - e.g. emails to send, frames to render, files to transcode, NoSQL DB keys ranges to scan
  - consider this case a _batch job_
  - _fyi: [see for examples](https://kubernetes.io/docs/concepts/workloads/controllers/job/#job-patterns)_
- parallel computation patterns for batch jobs
  - job-per-work-item vs. one job for all
    - 1-per-1: creates overhead for user and system with a lot of job objects
    - 1-for-all: A single job for all work items is better for large numbers of items
  - pod-per-work-item vs. pod processes multiple work items
    - 1-per-1: requires less modification to existing code and containers
    - 1-for-some: better for large numbers of items
  - work queue
    - requires running queue service
    - requires modifications to program or container to make it use the queue
    - other approaches easier to adapt to existing containerised app
  - job is associated with headless service
    - can enable pods in gob to communicate with each other for collaboration

| Pattern                                 | `.spec.completions` | `.spec.parallelism` |
| --------------------------------------- | ------------------- | ------------------- |
| Queue with Pod Per Work Item            | x                   | any                 |
| Queue with Variable Pod Count           | null                | any                 |
| Indexed Job with Static Work Assignment | x                   | any                 |
| Job with Pod-to-Pod Communication       | x                   | x                   |
| Job Template Expansion                  | 1                   | should be 1         |

## Advanced usage

### Job suspension

- `.spec.suspend` (bool)
  - can start jobs in suspended state or suspend while executing
  - fyi: on resume `.status.startTime` will be reset to current time
    - i.e. `.spec.activeDeadlineSeconds` will be will also be reset
- on job suspend
  - running non `Completed` pods are terminated with `SIGTERM` signal
    - graceful termination period will be honored
    - these pods won't increment `completions`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: myjob
spec:
  suspend: true
  parallelism: 1
  completions: 5
  template:
    spec: ...
```

- ...
  - job is defined as suspended
    - check with `kubectl get job myjob -o yaml`
  - suspend via e.g.
    - `kubectl patch job/myjob --type=strategic --patch '{"spec":{"suspend":true}}'`
  - resume via e.g.
    - `kubectl patch job/myjob --type=strategic --patch '{"spec":{"suspend":false}}'`
  - check with `kubectl get jobs/myjob -o yaml`

```yaml
apiVersion: batch/v1
kind: Job
# .metadata and .spec omitted
status:
  conditions:
    - lastProbeTime: "2021-02-05T13:14:33Z"
      lastTransitionTime: "2021-02-05T13:14:33Z"
      status: "True"
      type: Suspended
  startTime: "2021-02-05T13:13:48Z"
```

- ...
  - type `Suspended` and status `True` means suspended
  - `lastTransitionTime` here means for since when
  - status `False` would mean was suspended but now running
  - if such condition not there then job was never stopped
  - can also check events `kubectl describe jobs/myjob`

```
Name:           myjob
...
Events:
  Type    Reason            Age   From            Message
  ----    ------            ----  ----            -------
  Normal  SuccessfulCreate  12m   job-controller  Created pod: myjob-hlrpl
  Normal  SuccessfulDelete  11m   job-controller  Deleted pod: myjob-hlrpl
  Normal  Suspended         11m   job-controller  Job suspended
  Normal  SuccessfulCreate  3s    job-controller  Created pod: myjob-jvb44
  Normal  Resumed           3s    job-controller  Job resumed
```

- ...
  - last 4 events caused by `.spec.suspend` toggling
    - e.g. no pods were created but pod creation restarted asap on job resume

### Mutable scheduling directives

- sometimes parallelism shoul happen on same /something/ unit (???)
- fields in job pod template that can be updated are:
  - node affinity, selector,
  - tolerations
  - labels, annotations
  - scheduling gates

### Specifying pod selector

- `.spec.selector` is not needed normally
  - system will pick a selector not overlapping with other jobs
  - can be dangerous if non-unique selector is specified
    - other controllers will match selector or wrong pods will be targeted
- case: job `old` is running
  - existing pods should keep running
  - rest of pods it creates should use different template
  - job should have a new name
  - can't update job because these fields are not updatable
  - so: delete job `old` but leave pods running
    - `kubectl delete jobs/old --cascade=orphan`
  - before deletion: note selector
    - `kubectl get job old -o yaml`

```yaml
kind: Job
metadata:
  name: old
  ...
spec:
  selector:
    matchLabels:
      batch.kubernetes.io/controller-uid: a8f3d00d-c6d2-11e5-9f87-42010af00002
  ...
```

- ...
  - above would be similar output of the get command
  - now: create new job with same selector
    - pods label: `batch.kubernetes.io/controller-uid=a8f3d00d-c6d2-11e5-9f87-42010af00002`
    - specify `manualSelector: true` in new job

```yaml
kind: Job
metadata:
  name: new
  ...
spec:
  manualSelector: true
  selector:
    matchLabels:
      batch.kubernetes.io/controller-uid: a8f3d00d-c6d2-11e5-9f87-42010af00002
  ...
```

- ...
  - new job itself will have different uid from `...-42010af00002`

### Job tracking with finalizers

- controller tracks pods belonging to jobs and notices such pod removal from API
  - how: creates pods with finalizer `batch.kubernetes.io/job-tracking`
  - removes finalizer only after pod accounted for in job status

### Elastic indexed jobs

- scale indexed jobs by mutating `.spec.parallelism` and `.spec.completions`
  - such that `.spec.parallelism == .spec.completions`
- fyi: on scaling down pods with higher indexes are removed
- cases: scaling batch workloads like PyTorch training jobs

### Delayed creation of replacement pods

- fyi: needs `JobPodReplacementPolicy` feature gate
- controller recreates pods on fail or while terminating
  - i.e. pods count can be greater than `parallelism` or per-index
- `.spec.podReplacementPolicy: Failed` replace pods only on full termination
- default replacement policy depends on whether job has `podFailurePolicy`
  - if not set
    - no `podReplacementPolicy` defaults to `TerminatingOrFailed`
  - if set
    - no `podReplacementPolicy` defaults to `Failed`
- fyi: can check `.status.terminating` of job
  - with `kubectl get jobs/myjob -o yaml`

```yaml
apiVersion: batch/v1
kind: Job
# .metadata and .spec omitted
status:
  # three Pods are terminating and have not yet reached the Failed phase
  terminating: 3
```

### Delegation of managed job objects to external controller

- fyi: needs `JobManagedBy` feature gate
- can disable built-in job controller for specific job
  - delegate reconciliation of the job to external controller
- `spec.managedBy` set to not `kubernetes.io/job-controller`
  - make sure the value is an installed controller

## Alternatives

- bare pods
  - on node reboot or fail pod is terminated and not restarted
    - job would replace with new pods
  - recommeded: use job even for single pod
- replication controller
  - manages pods not expected to terminate (e.g. web servers)
    - job manages pods expected to terminate (e.g. batch tasks)
    - i.e. jobs are complementary to replication controller
  - job is only appropriate for:
    - pods with `RestartPolicy` equal to `OnFailure` or `Never`
      - fyi: default is `Always`
- single job starts controller pod
  - pattern: single job creates pod
    - which creates other pods
    - acting like a controller for those pods
    - is flexible but complex and less integration with k8s
  - example:
    - job starts pod which runs a script
    - script starts Spark master controller
      - runs a spark driver
      - then cleans up
  - advantage:
    - overall process gets completion guarantee of a job object
    - maintains complete control over pod creation and collaboration

# K8S CronJob

- creates jobs on repeating schedule written in `Cron` format
  - e.g. for backups, report generation
- one `CronJob` is like line of crontab (cron table) file in Unix
- `.metadata.name` is part of auto-created jobs
  - DNS label rules and **no more than 52 chars**
    - because 11 chars will be auto-appended
- `.spec`
  - `.jobTemplate` (mandatory) job template (but nested)
    - _warning: changing only affects new jobs_
    - _warning: jobs must be idempotent_
    - fyi: sets job annotation `batch.kubernetes.io/cronjob-scheduled-timestamp`
      - originally scheduled creation time for the job in RFC3339
  - `.startingDeadlineSeconds` skips that job instance if not started in that time
    - over-deadline jobs treated as failed
    - if not specified there won't be any deadline (can get dangerous)
    - _warning: don't set under 10s as controller checks every 10s_
      - i.e. could happen that job is never scheduled
    - fyi: after too many missed schedule attempts (100) logs error
  - `.concurrencyPolicy` for when jobs are croned in parallel
    - `Allow` (default) doesn't matter
    - `Forbid` skips new job if old is still running
      - `.startingDeadlineSeconds` could mean that it's still started if inside deadline
    - `Replace` new job replaces old job
    - fyi: policy only targets jobs inside the same `CronJob`
    - fyi: if `startingDeadlineSeconds` is large or unset (default) and `concurrencyPolicy=Allow`
      - jobs will always run at least once
  - `.suspend` can be toggled but won't stop running jobs
    - _warning: all missed jobs will be started on untoggle without deadline!_
  - `.successfulJobsHistoryLimit` default 3, keeps succeeded jobs
  - `.failedJobsHistoryLimit` default 1, keeps failed jobs
  - `.timeZone` [see zones](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) e.g. "Etc/UTC"
    - defaults to local time zone of controller
    - fyi: time zone database from Go standard library is included as fallback
  - `.schedule` (mandatory) see below
    - _warning: `CRON_TZ` or `TZ` nor supported_

```
# ┌───────────── minute (0 - 59)
# │ ┌───────────── hour (0 - 23)
# │ │ ┌───────────── day of the month (1 - 31)
# │ │ │ ┌───────────── month (1 - 12)
# │ │ │ │ ┌───────────── day of the week (0 - 6) (Sunday to Saturday)
# │ │ │ │ │                                   OR sun, mon, tue, wed, thu, fri, sat
# │ │ │ │ │
# │ │ │ │ │
# * * * * *
```

- ...
  - e.g. `0 3 * * 1` is "weekly on a Monday at 3 AM"
  - fyi: includes "Vixie cron" step values [see here](https://www.freebsd.org/cgi/man.cgi?crontab%285%29)
  - fyi: ? in schedule means \* i.e. any value for a given field
  - [see generation web tools](https://crontab.guru/)
  - fyi: also has macros

| Entry                  | Description                                | Equivalent to |
| ---------------------- | ------------------------------------------ | ------------- |
| @yearly (or @annually) | once a year, midnight, 1 Jan               | 0 0 1 1 \*    |
| @monthly               | once a month, midnight, first day of month | 0 0 1 \* \*   |
| @weekly                | once a week, midnight, Sunday morning      | 0 0 \* \* 0   |
| @daily (or @midnight)  | once a day, midnight                       | 0 0 \* \* \*  |
| @hourly                | once an hour, beginning of hour            | 0 \* \* \* \* |

- example:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: hello
              image: busybox:1.28
              imagePullPolicy: IfNotPresent
              command:
                - /bin/sh
                - -c
                - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

- ...
  - prints current time and hello message every minute
