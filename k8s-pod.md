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

### Init containers

- `initContainers`: pod can have a set of init containers
  - status is returnes in `.status.initContainerStatuses`
  - e.g. utils or setups which are not in an app image
- init container
  - always runs to completion during pod init
  - each must complete successfully before next starts
  - failures are treated according to `restartPolicy`
    - on `Never` and init container failure whole pod is a fail
- init containers are the same as normal containers, but:
  - resource limits and requests are handled differently
  - no `lifecycle`, `livenessProbe`, `readinessProbe`, `startupProbe` fields
    - because must run to completion anyway, sequentially
- advantages
  - no need to make an image `FROM` another image just to use a tool
    - e.g. `sed`, `awk`, `python`, or `dig` during setup
  - app image builder and deployer roles work separation
    - no need to build a single app image
  - can have different filesystem view i.e. secrets from app containers
  - init sequence means safe pre-conditions met before rest is started
  - more security by moving privileged access only to init containers for setup
- examples
  - wait for a `Service` by shell command
    - `for i in {1..100}; do sleep 1; if nslookup myservice; then exit 0; fi; done; exit 1`
  - register pod with a remote server from the downward API with a command
    - `curl -X POST http://$MANAGEMENT_SERVICE_HOST:$MANAGEMENT_SERVICE_PORT/register -d 'instance=$(<POD_NAME>)&ip=$(<POD_IP>)'`
  - wait before starting the app container with a command
    - `sleep 60`
  - clone a Git repository into a `Volume`
  - place values into cfg file
    - then run a template tool to dynamically generate cfg file for app containers

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app.kubernetes.io/name: MyApp
spec:
  containers:
    - name: myapp-container
      image: busybox:1.28
      command: ["sh", "-c", "echo The app is running! && sleep 3600"]
  initContainers:
    - name: init-myservice
      image: busybox:1.28
      command:
        [
          "sh",
          "-c",
          "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done",
        ]
    - name: init-mydb
      image: busybox:1.28
      command:
        [
          "sh",
          "-c",
          "until nslookup mydb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done",
        ]
```

- ...
  - simple pod with two `initContainers`
  - first waits for `myservice`
  - second waits for `mydb`
  - then pod runs app container from `containers`
  - test with `kubectl apply -f myapp.yaml`
    - output e.g.: `pod/myapp-pod created`
  - check immediately with `kubectl get -f myapp.yaml`
    - output among other things e.g. `STATUS: Init:0/2`
  - or check with details: `kubectl describe -f myapp.yaml`
  - see logs
    - `kubectl logs myapp-pod -c init-myservice`
    - `kubectl logs myapp-pod -c init-mydb`
  - init containers will be waiting to discover their services
  - add the services to make them appear:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myservice
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
---
apiVersion: v1
kind: Service
metadata:
  name: mydb
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9377
```

- ...
  - apply with `kubectl apply -f services.yaml`
    - output: `service/myservice created` `service/mydb created`
  - check with `kubectl get -f myapp.yaml`
    - output among other things `STATUS: Running`

### Sidecar containers

- fyi: may have to enable `SidecarContainers` feature gate
- sidecars are _regular init containers_ i.e. like rules inits but remain running
  - can have probes
  - i.e. just set `restartPolicy` to `Always` in `initContainers` to make one
    - `readinessProbe` can be set, result will be `ready` state of pod
  - share all the benefits of init containers e.g. ordering
    - fyi: if sidecar starts successfully it's enough to start next init container
- sidecars run along the main app containers within the same pod
  - fyi: on pod termination app containers are stopped first, then sidecars
  - have independent lifecycles from app containers
    - i.e. update, scale, maintain sidecars without affecting the apps
    - i.e. changing image of sidecar restarts container, not whole pod
  - e.g. for logging, monitoring, security, data sync
  - e.g. web application:
    - local webserver is sidecar container
    - web app itself is app container

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: alpine:latest
          command:
            [
              "sh",
              "-c",
              'while true; do echo "logging" >> /opt/logs.txt; sleep 1; done',
            ]
          volumeMounts:
            - name: data
              mountPath: /opt
      initContainers:
        - name: logshipper
          image: alpine:latest
          restartPolicy: Always
          command: ["sh", "-c", "tail -F /opt/logs.txt"]
          volumeMounts:
            - name: data
              mountPath: /opt
      volumes:
        - name: data
          emptyDir: {}
```

##### Jobs with sidecars

- sidecar doesn't prevent `Job` from completing after app container is done (???)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: myjob
spec:
  template:
    spec:
      containers:
        - name: myjob
          image: alpine:latest
          command: ["sh", "-c", 'echo "logging" > /opt/logs.txt']
          volumeMounts:
            - name: data
              mountPath: /opt
      initContainers:
        - name: logshipper
          image: alpine:latest
          restartPolicy: Always
          command: ["sh", "-c", "tail -F /opt/logs.txt"]
          volumeMounts:
            - name: data
              mountPath: /opt
      restartPolicy: Never
      volumes:
        - name: data
          emptyDir: {}
```

### Ephemeral containers

- temporarily running container e.g. for troubleshooting, inspection
  - pod already running so most container fields disallowed
    - e.g. ports, livenessProbe, readinessProbe not allowed
    - e.g. setting resources
- when to use: if `kubectl exec` not enough for troubleshooting
  - fyi: use `shareProcessNamespace` to make that easier

## User namespaces

- namespace isolates user running in container from the one in host
  - process-as-root in container can run as non-root user in the host
    - i.e. fully privileged inside user namespace but not outside
- _caution: this doc is for container runtimes using Linux namespaces for isolation_
  - e.g. Docker Engine
- user namespaces is a Linux feature to map container users to other users in host
  - also pod user namespace capabilities are void outside of it
- `pod.spec.hostUsers` set to `false` to opt-in in user namespaces
  - kubelet picks host UIDs/GIDs a pod is mapped to
  - `runAsUser`, `runAsGroup`, `fsGroup`, etc. in `pod.spec` refer to user in container
- on creating a pod (default) several new namespaces are used for isolation
  - network namespace to isolate the network of the container
  - PID namespace to isolate the view of processes
  - if user namespace is used this will isolate users in container from users in node
    - i.e. containers can run as root and be mapped to a non-root user on host
      - in container process thinks it's root
        - `apt`, `yum` are ok to use
      - in reality it is not e.g. check with `ps aux` from host
        - `ps` shows user who is not same as user inside the container command
- capabilities of pod are mostly void outside of pod e.g.
  - `CAP_SYS_MODULE` limited to pod using user namespaces (can't load kernel modules)
  - `CAP_SYS_ADMIN` limited to pod user namespace and invalid outside of it
- **WARNING: without user namespace root container has root on node**
  - i.e. in case of "container breakout" (???)
- [see details](https://kubernetes.io/docs/concepts/workloads/pods/user-namespaces/)

## Downward API

- downward api: two ways to expose pod and container fields to a running container
  - as envvars
  - as files that are populated by a special volume type
- why: if container needs info about self or cluster without using Kubernetes clients
- `fieldRef` volume allows passing pod-level info
  - pod spec always defines at least one container
  - `resourceFieldRef` allows passing container-level info
- availabe via envvar and `fieldRef` api
  - `metadata.name` pod name
  - `metadata.namespace` pod namespace
  - `metadata.uid` pod unique ID
  - `metadata.annotations['KEY']` pod KEY annotation value
  - `metadata.labels['KEY']` pod KEY label value
- availabe via envvar (but not `fieldRef` api)
  - `spec.serviceAccountName` pod service account
  - `spec.nodeName` node name
  - `status.hostIP` node primary IP
  - `status.hostIPs` IP addresses (dual-stack version of `status.hostIP`)
    - first is always same as `status.hostIP`
  - `status.podIP` pod primary IP (usually IPv4)
  - `status.podIPs` IP addresses (dual-stack version of `status.podIP`)
    - first is always same as `status.podIP`
- availabe via `fieldRef` api (but not envvar)
  - `metadata.labels` all pod labels
    - format: `label-key="escaped-value"`, token per line
  - `metadata.annotations` all pod annotations
    - format: `annotation-key="escaped-value"`, token per line
- available via `resourceFieldRef` api (container-level info)
  - `resource: limits.cpu`
  - `resource: requests.cpu`
  - `resource: limits.memory`
  - `resource: requests.memory`
  - `resource: limits.hugepages-*`
  - `resource: requests.hugepages-*`
  - `resource: limits.ephemeral-storage`
  - `resource: requests.ephemeral-storage`
  - fyi if not specified will still show default info values
