# Interesting stuff not obvious through online tutorials

[back](README)

## General concept reasoning

- need to run logic: use `Image`s (i.e. Docker)
- need something to run `Image`s on: see `Node`s
- need a layer in k8s for `Image`s: see `Container`s
- need a layer to make `Container`s controllable: see `Pod`s
- need controllers to manage replicas etc.: see e.g. `Deployment`
- need exposing logic as endpoint in/out of k8s: see `Service`
- need DNS and reverse-proxying: see `Gateway`

## General concept

- (Docker) `Image`s run in `Container`s (ephemeral)
- `Container`s run in `Pod`s (ephemeral)
  - `Pod` is needed to provide a controlling possibility of `Container`s
  - usually container-per-pod
- `Pod`s run on `Node`s (actual running instances e.g. physical machine)
  - auto-assigned by controllers
- `Pod`s run in workloads e.g. `Deployment`s for stateless
  - so a `Deployment` is controller entity, manages replicas of `Pod`s on `Node`s
- `ReplicaSet`s maintain a stable number of pods
  - usually itself auto-managed by `Deployment`
- `StatefulSet` are `Deployment`s with persistence
  - can lead to failure state where manual termination of pods is required

## `kubectl` util

- apply file: `kubectl apply -f myapp.yaml`
  - can apply whole directories instead of single files
- check results: `kubectl get -f myapp.yaml`
- check with details: `kubectl describe -f myapp.yaml`
- check logs: `kubectl logs myapp-pod -c myservice`
- any `-f` file command also works with `-R` for recursive nested dirs

## 'Pod'

- pod runs containers
  - and can have init containers
    - run first, to completion, in sequence
    - good for e.g. privileged access for setup taken out of actual app containers
  - and can have sidecar containers
    - run alongside app containers
    - good for e.g. webserver in sidecar, actuall webapp in app container

## Gotchas

- make sure `Service`s run before components that depend on them
  - issue: else DNS won't be available yet
