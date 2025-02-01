# K8S Pod

[back](README#workload)

- smallest deployable units of computing created and managed in Kubernetes
- group of one or more containers with
  - shared storage
  - shared network resources
  - specification hot to run containers
  - co-located and co-scheduled, run in shared context
  - e.g. like apps on same physical/virtual machine
- fyi: can contain init containers run during pod startup
- fyi: can inject ephemeral containers for debugging a running pod
- used in two ways
  - running single container
    - most common use
    - pod is a wrapper for managing the container by k8s
  - running multi-containers which work together
    - containers which work toghether and share resources
    - i.e. forming a single unit
    - only use in specific instances where containers are tightly coupled
    - don't use to provide replication

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.14.2
      ports:
        - containerPort: 80
```

- create pod: `kubectl apply -f https://k8s.io/examples/pods/simple-pod.yaml`
- fyi: pods are not created directly, instead use workload resources
- fyi: pod name must be a valid DNS subdomain or even DNS label
  - i.e. max len 63, start with alpha, end with alphanum, can have '-'

## Pod Template

- workload resource controller creates Pods from a pod template
- is a spec for creating a pod
- fyi: can include envvars

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello
spec:
  template:
    # pod template start
    spec:
      containers:
        - name: hello
          image: busybox:1.28
          command: ["sh", "-c", 'echo "Hello, Kubernetes!" && sleep 3600']
      restartPolicy: OnFailure
    # pod template end
```

- ...

  - above is a simple job
    - starts a single container
      - which prints a message then pauses

- fyi: modifying a pod template or switching to a new one has no effect on running pods
  - workload resource instead creates replacement pods
    - each resource implements own rules
    - e.g. `StatefulSet`:
      - gets updated with a pod template change
      - starts to create new pods base don new template
      - eventually old pods are replaced by new pods

## Pod update and replacement

- on pod template change the workload resource controller
  - creates new pods based on new template
  - does not update or patch existing pods
- k8s provides pod commands for `patch` and `replace` but
  - most of metadata is immutable
  - ...and many other limitations so not a good idea

## Pod sharing and communication

- pod can specify a set of shared storage volumes
  - all containers in pod can access those
  - persistent data will survive a container restart
- each pod gets unique IP for each address family
  - each container shares the network namespace incl. IP and ports
  - within, containers can communicate with each other via `localhost`
    - also can communicate via SystemV semaphores or POSIX shared memory
  - within, the system hostname is the same as configured `name` of pod

## Pod security

- `securityContext` field used for constraints on pod or individual containers
  - e.g. drop specific Linux capabilities to avoid the impact of a CVE
  - e.g. force all processes in the pod to run as a non-root user
  - e.g. set a specific seccomp profile
  - e.g. set Windows security options i.e. containers run as HostProcess
- _CAUTION: avoid privileged mode in Linux_

## Static pods

- managed directly by kubelet daemon on a specific node
  - most pods are managed by e.g. Deployment here it's directly `kubelet`
- main use is self-hosted control pane (???)

## Pod lifecycle

- pod lifecycle progress
  - `Pending` is starting phase
  - `Running` if at least one primary container is running
  - `Succeeded` all containers terminated and none failed, or
  - `Failed` all containers terminated and min one failed
- container lifecycle
  - `Waiting` e.g. while pulling secrets or image
  - `Running` is executing
    - `postStart` hook would be completed at this point
  - `Terminated` it ran and stopped, `kubectl` will show reason and exit code
    - `preStop` hook would be completed at this point
- pod is relatively ephemeral like containers themselves, it is
  - is created
  - assigned UID
  - scheduled to run on nodes where they remain until termination/deletion
- node death: pods run or scheduled on it are marked for deletion
- pod readiness can be extended by `spec.redinessGates` (see elsewhere)

### Pod container problem handling

- `spec.restartPolicy` determines handling
  - `Always` auto-restarts after any termination
  - `OnFailure` restarts after termination with !0 exist-code
  - `Never` does not restart
- first crash: leads to immediate restart based on policy
- repeated crashes: k8s applies exponential backoff delay based on policy (up to 5min)
- `CrashLoopBackOff` state: means backoff delay currently in effect
- backoff reset: container runs for a time e.g. 10min and k8s resets backoff
  - i.e. next crash will be a first crash again

#### Investigation of a `CrashLoopBackOff` issue

- check logs via `kubectl logs <name-of-pod>`
- check events via `kubectl describe pod <name-of-pod>`
- check configuration including envvars, mounted volumes, required resources
- check resource limits i.e. raise CPU and memory allocated
- debug application i.e. run container image locally or in a dev environment

### Container probes

- probe: periodic diagnostic by `kubelet` on a container
  - ExecAction: via container runtime
  - TCPSocketAction: directly by kubelet
  - HTTPGetAction: directly by kubelet
- check mechanisms
  - `exec` command, OK when returns 0
    - can be bad for performance see `initialDelaySeconds`, `periodSeconds`
  - `grpc` call, OK when returns status `SERVING`
  - `httpGet` on pod IP on specified path and port, OK when returns 200 or below 400
  - `tcpSocket` on pod IP and port, OK if port open
- probe type
  - `livenessProbe` whether container running
    - none means `Success`
    - on fail will go to restart
    - fyi: not needed if process crashes itself when issues arise
  - `readinessProbe` whether container ready to respond to requests
    - none means `Success`
    - default state is `Failure` while init delay is active
    - on fail will be removed from endpoints of services
    - fyi: needed when want to delay networkt traffic until really ready
    - fyi: needed when container should be able to become inactive for maintenance
    - fyi: needed when container depends on other backends
      - i.e. self is live but dependencies not yet
    - fyi: needed when relying on loading or preparing large data or similar
  - `startupProbe` whether application in container started
    - none means `Success`
    - other probes are disabled if this provided until this succeeds
    - on fail will go to restart

### Pod termination

- important to allow processes to be terminated gracefully (rather than `KILL` signal)
- i.e. should react to `TERM`/`SIGTERM` signal (with a grace period then `KILL`)

### Pod garbage collection

- failed pods' objects stay until human or controller explicitly removes them
- PodGC removes terminated pods when threshold gets exceeded or when:
  - pods are orphaned i.e. node doesn't exist anymore
  - unsceduled terminating pods (???)
  - pods bound to non-ready node tainted by `node.kubernetes.io/out-of-service`

### TODO https://kubernetes.io/docs/concepts/workloads/pods/init-containers/
