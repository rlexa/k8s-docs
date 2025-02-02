# Interesting stuff not obvious through online tutorials

[back](README)

- `kubectl` util
  - apply file: `kubectl apply -f myapp.yaml`
    - can apply whole directories instead of single files
  - check results: `kubectl get -f myapp.yaml`
  - check with details: `kubectl describe -f myapp.yaml`
  - check logs: `kubectl logs myapp-pod -c myservice`
- pod runs containers
  - and can have init containers
    - run first, to completion, in sequence
    - good for e.g. privileged access for setup taken out of actual app containers
  - and can have sidecar containers
    - run alongside app containers
    - good for e.g. webserver in sidecar, actuall webapp in app container
- gotchas
  - make sure `Service`s run before components that depend on them
    - issue: else DNS won't be available yet
