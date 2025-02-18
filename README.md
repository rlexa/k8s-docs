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

##### Workload scaling

- horizontal
  - fyi: can be done manually with `kubectl` (changing replicaset number)
  - `HorizontalPodAutoscaler` i.e. HPA auto-scales (native)
  - `kubectl scale deployment/my-nginx --replicas=1`
    - hard manual downscale to 1
  - `kubectl autoscale deployment/my-nginx --min=1 --max=3`
    - auto scale between in a range
    - _warning: requires metrics (???)_
- vertical
  - fyi: can be done manually by patching the resource definition
  - `VerticalPodAutoscaler` i.e. VPA auto-scales (not native)
    - needs an installed "Metrics Server" (???)
    - [see for more details](https://kubernetes.io/docs/concepts/workloads/autoscaling/#scaling-workloads-vertically)

#### Workload management

- app is deployed and exposed via service, now what?
- fyi: can group resources in single yaml by using `---` separator
  - can then be created by single apply command
  - resources in yaml are created in order
  - alternative: multiple "-f" file arguments for apply command
- recommended: microservices in single yaml and all of app in same directory
  - can then be created by single apply command on directory
  - nested directories also possible by adding `--recursive` or `-R`
- fyi: apply also works with url
  - `kubectl apply -f https://k8s.io/examples/application/nginx/nginx-deployment.yaml`
- fyi: can also delete in bulk, not just apply
  - e.g. by selector `kubectl delete deployment,services -l app=nginx`
- fyi: chaining and filtering is possible
  - `kubectl get $(kubectl create -f docs/concepts/cluster-administration/nginx/ -o name | grep service/ )`
  - `kubectl create -f docs/concepts/cluster-administration/nginx/ -o name | grep service/ | xargs -i kubectl get '{}'`
    - create resources under examples/application/nginx/
    - prints resources created with `-o name` format (as resource/name)
    - grep only the service
    - print it with `kubectl get`
- update app without outage e.g. with new image or tag
  - use e.g. rollout to gradually shift, example:
    - `kubectl create deployment my-nginx --image=nginx:1.14.2`
      - output `deployment.apps/my-nginx created`
    - ensure there is 1 replica
      - `kubectl scale --replicas 1 deployments/my-nginx --subresource='scale' --type='merge' -p '{"spec":{"replicas": 1}}'`
        - output `deployment.apps/my-nginx scaled`
    - allow to add temp replicas on rollout (surge max 100%)
      - `kubectl patch --type='merge' -p '{"spec":{"strategy":{"rollingUpdate":{"maxSurge": "100%" }}}}'`
        - output `deployment.apps/my-nginx patched`
    - update nginx
      - `kubectl edit deployment/my-nginx`
        - change `.spec.template.spec.containers[0].image` to `nginx:1.16.1`
  - check with rollout command
    - `kubectl apply -f my-deployment.yaml`
    - `kubectl rollout status deployment/my-deployment --timeout 10m`
      - waits for rollout to finish
      - fyi: can just check status with `--watch=false`
      - fyi: can also pause/resume/cancel rollout
- update app cfg which must be replaced
  - `kubectl replace -f https://k8s.io/examples/application/nginx/nginx-deployment.yaml --force`
    - deletes then replaces
- canary deployments e.g. for a/b releases
  - for example use tags e.g.
    - same: `labels.app: guestbook`, `labels.tier: frontend`
    - then add track `.labels.track: stable` and `.labels.track: canary`
      - with different images
  - frontend service then omits track in `.selector`
    - traffic now redirected to both apps
  - now can test then update stable to canary's image and remove canary
- attach annotations: arbitrary non-id metadata retrieved by API clients e.g. libs and tools
  - `kubectl annotate pods my-nginx-v4-9gw19 description='my frontend running nginx'`
  - `kubectl get pods my-nginx-v4-9gw19 -o yaml`
    - output among other things: `.metadata.annotations.description: my frontend running nginx`

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

## Tools

- [see for more](https://landscape.cncf.io/guide#app-definition-and-development--application-definition-image-build)
- Helm
  - manages helm charts i.e. packages of pre-configured k8s resources
- Kustomize
  - traverses k8s manifest to add/remove/update cfg options

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
