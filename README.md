# Kubernetes

Orchestration framework.

[See non-obvious interesting stuff here.](INTERESTING)

## Concept

See Avril Lavigne "Complicated".

### General concept

- `Object`: yaml manifest based entity ("base class"), some fields:
  - `apiVersion` kubernetes api version to use
  - `kind` what object e.g. "Deployment"
  - `metadata` for unique IDs (name, UID, optional namespace)
  - `spec` state for the object (different per object, contains nested specific fields)
- `Container`: based on provided docker image
- `Pod`: group of containers
  - fyi a pod is only accessible by its internal IP address within the kubernetes cluster
    - expose pod as a service for outside use
  - **pods are ephemeral**, created and renamend and replaced and deleted dynamically
- `Deployment`: checks pod health, re-starts/terminates
- `Node`: a virtual or physical machine running the pods

#### Workload

- workload is a k8s app running in a set of pods
- pod has a lifecycle
  - e.g. a pod critical failure kills all pods in a node
    - i.e. critical failure level is final, new pod needs to be created now
- to make lifecycle control easier pods are not managed directly, instead use workload resources
  - a workload resource configures a controller which ensures:
    - right number of right kind of pods are running to match the defined state
- built-in workload resources
  - `Deployment` and `ReplicaSet`
    - for managing a stateless app workload i.e. pods are interchangeable
  - `StatefulSet`
    - for pods that track state
      - e.g. recording persistent data means: `StatefulSet` matching each pod with a `PersistentVolume`
      - the code running in these pods can replicate data to other pods in the same set
  - `DaemonSet`
    - for pods providing node-local something
    - every time a node is added matching a `DaemonSet` the control plane schedules a pod for that `DaemonSet` onto the new node
    - each pod in a `DaemonSet` performs a job similar to a system daemon on a classic Unix / POSIX server
      - e.g. plugin to run cluster networking, helping manage the node etc.
  - `Job` and `CronJob`
    - for tasks running to completion and then stop (once with a `Job` or regular with `CronJob`)
- fyi there are 3rd party resources available
- see details for
  - [Pod](k8s-pod)
  - [Deployment/Replicaset](k8s-deployment-replicaset)
  - [StatefulSet](k8s-stateful-set)
  - [DaemonSet](k8s-dameon-set)
  - [Job/CronJob](k8s-job-cronjob)
  - `ReplicationController` is obsolete (use `Deplyoment` and `ReplicaSet`)

### Network concept

- IP per pod-in-cluster
  - each pod has private inner network for containers inside
    - processes inside communicate via localhost
- pod network === cluster network: handles comm between pods
  - all pods can comm with each other without proxy or NAT (even across nodes)
    - windows may have a problem here with host-network pods (???)
  - node agents (e.g. kubelet) comm with all pods on that node
- Service API: stable IP or hostname for a group of pods as "backend"
  - auto-managed EndpointSlice objects for pods currently in the service
  - service proxy monitors Service and EndpointSlice objects and routes data traffic accordingly
- Gateway API === Ingress (old name): opens access to Services to clients outside the cluster
  - simpler but less flexible way: LoadBalancer (needs compatible CloudProvider)
- NetworkPolicy: allows control of traffic in-between pods and between pods and outside
- fyi: only some parts are native k8s, rest is provided via APIs:
  - pod network namespace setup: Container Runtime Interface implementations
  - pod network: e.g. on Linus using CNI (Container Network Interface) i.e. CNI Plugins
  - proxy: k8s has kube-proxy but some pod network implementations use their own
  - NetworkPolicy generally implemented by pod network implementation
  - Gateway API has many implementations (generic, bare-metal, cloud service particular etc.)

#### DNS

- [DNS](k8s-dns)
- _TODO IPv4/IPv6 dual-stack_
- _TODO topology aware routing_
- _TODO networking on Windows_
- [Service ClusterIP allocation](k8s-service-cluster-ip-alloc)
- [Service internal traffic policy](k8s-service-internal-traffic-policy)

## Entities

- [Service](k8s-service)
- [Ingress](k8s-ingress) (obsolete)
- [Gateway](k8s-gateway)
- [EndpointSlice](k8s-endpoint-slice)
- [NetworkPolicies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) (TODO when needed)

## Storage

- `Volume`s provide a way for containers to access and share data via filesystem
- data sharing between different local processes within a container, or between different containers, or between Pods
- purposes:
  - populating a configuration file based on a `ConfigMap` or a `Secret`
  - providing temporary scratch space for a pod
  - sharing a filesystem between two different containers in the same pod
  - sharing a filesystem between two different pods (even if those pods run on different nodes)
  - storing data so that it stays available even if the Pod restarts or is replaced
  - passing configuration information to an app running in a container, based on details of the Pod the container is in
    - e.g. telling a sidecar container what namespace the Pod is running in
  - providing read-only access to data in a different container image
- e.g. data persistence
  - on-disk files in a container are ephemeral
  - problem: container crashes or is stopped, state is not saved and is lost
    - during a crash, kubelet restarts the container with a clean state
- e.g. shared storage

  - problem: multiple containers in a pod need to share files

### Storage Details

- [Volumes](k8s-volumes)

## Configuration

### Best Practice

- when defining configurations specify the latest stable API version
- store cfg files in version control before being pushing to cluster
- use YAML rather than JSON
- group related objects in single file whenever it makes sense
  - [guestbook example](https://github.com/kubernetes/examples/blob/master/guestbook/all-in-one/guestbook-all-in-one.yaml)
- `kubectl` commands can be called on a directory (e.g. apply whole directory)
- don't set default values (zero configuration)
- put object descriptions in annotations

#### Pod

- don't use naked pods (use ReplicaSet or Deployment)
  - won't be rescheduled in the event of a node failure

#### Service

- create services before corresponding workloads and before dependant workloads
  - k8s starts container and provides envvars for currently running services
  - e.g. service `foo` exists:
    - `FOO_SERVICE_HOST=<the host the Service is running on>`
    - `FOO_SERVICE_PORT=<the port the Service is running on>`
  - fyi: DNS does not have this restriction
- DNS server plugin is optional but strongly recommended
- don't specify pod `hostPort` nor `hostNetwork` when possible
  - for debugging purposes instead use the `apiserver proxy` or `kubectl port-forward`
  - for explicitly exposing a pod port on the node use `NodePort` Service
- use headless services (which have a ClusterIP `None`) for service discovery when you don't need kube-proxy load balancing

#### Labels

- use semantic labels for application or Deployment
  - e.g. `{ app.kubernetes.io/name: MyApp, tier: frontend, phase: test, deployment: v3 }`
  - use these labels to select correct pods for other resources
    - e.g. service selects `tier: frontend` pods
    - e.g. service selects `phase: test` components of `app.kubernetes.io/name: MyApp`
- use Kubernetes common labels for common use cases
  - allows tools, including `kubectl` and dashboard, to work in an interoperable way
- manipulate labels for debugging with `kubectl label`
  - remove relevant labels from pod, it's controller will create new pod (???)
    - useful for debugging previously live pod in quarantine

#### Using `kubectl`

- use `kubectl apply -f <directory>` which looks for all k8s .yaml, .yml, and .json files
- use label selectors for `get` and `delete` instead of object names
- use `kubectl create deployment` and `kubectl expose` to quickly create single-container env
  - e.g. for a service then to access an application in a cluster

### [TODO continue...](https://kubernetes.io/docs/concepts/configuration/configmap/)

## Setup

- install kubernetes
  - windows powershell: `winget install -e --id Kubernetes.kubectl`
  - test: `kubectl version --client`
- install `minikube` (for running kubernetes cluster, needs Docker)
  - (see most up-to-date installer)
- update minikube+kubernetes: `minikube kubectl -- get po -A`

### Start

- start cluster: `minikube start`
- access cluster: `kubectl get po -A`
- dashboard: `minikube dashboard`

### Manage

- pause: `minikube pause`
- unpause: `minikube unpause`
- haltcluster: `minikube stop`
- change default memory limit (requires a restart): `minikube config set memory 9001`
- browse catalog of easily installed kubernetes services: `minikube addons list`
- create another cluster running an older kubernetes release: `minikube start -p aged --kubernetes-version=v1.16.1`
- delete all clusters: `minikube delete --all`

### Deployment (from minikube example)

- create sample: `kubectl create deployment hello-minikube --image=kicbase/echo-server:1.0`
- expose port: `kubectl expose deployment hello-minikube --type=NodePort --port=8080`
- see whether is running: `kubectl get services hello-minikube`
- let minikube open browser (knows port): `minikube service hello-minikube`
  - alternatively expose service's 8080 to 7080 and open in browser: `kubectl port-forward service/hello-minikube 7080:8080`

### Deployment (from kubernetes example)

- create deployment: `kubectl create deployment hello-node --image=registry.k8s.io/e2e-test-images/agnhost:2.39 -- /agnhost netexec --http-port=8080`
- check deployments: `kubectl get deployments`
- check pods: `kubectl get pods`
- check events: `kubectl get events`
- check cfg: `kubectl config view`
- check logs (replace pod name): `kubectl logs current-pod-name`
- expose pod as a service: `kubectl expose deployment hello-node --type=LoadBalancer --port=8080`
  - `--type=LoadBalancer` flag indicates that you want to expose your service outside of the cluster
- check services: `kubectl get services`
- on minikube access service: `minikube service hello-node`
  - on cloud it would get an IP
- clean up service: `kubectl delete service hello-node`
- clean up deployment: `kubectl delete deployment hello-node`

## Real world experiments

- [Timeseries topology](k8s-example-timeseries-topology)
