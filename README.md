# Kubernetes

Orchestration framework.

## Concept

See Avril Lavigne "Complicated".

### General

- `Object`: yaml manifest based entity, some fields:
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

### Network

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

### Entities

- [Service](k8s-service)
- [Ingress](k8s-ingress) (obsolete)
- [Gateway](k8s-gateway)
- [EndpointSlice](k8s-endpoint-slice)

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
