# K8S Volumes

[back](README#storage-details)

- volume is a directory accessible to containers in a pod
  - pod is launched
  - container process sees
    - filesystem from container image
    - volumes mounted at specified paths
      - must be specified per container
      - fyi: volume can not mount another volume (but see `subPath`)
- pod can use any number of volumes
  - ephemeral volumes have lifetime of pod
  - persistent volumes exist beyond
- also possible but not documented here:
  - mount propagation
    - for sharing volumes mounted by a container to other containers in the same pod
      - or even to other pods on the same node
  - readonly mounts
    - `.spec.containers[].volumeMounts[].readOnly` makes a volume readonly for only that container

## CSI (Container Storage Interface)

- standard interface for container orchestration systems (e.g. Kubernetes) exposing storage systems to container workloads
- (???) [see here](https://kubernetes.io/docs/concepts/storage/volumes/#csi)

## Type Overview

- deprecated: `awsElasticBlockStore`, `azureDisk`, `azureFile`, `cephfs`, `cinder`, `gcePersistentDisk`, `gitRepo`, `portworxVolume`, `vsphereVolume`, `flexVolume`
- removed: `glusterfs`, `rbd`
- alpha: `image`

### `configMap`

- `ConfigMap` provides a way to inject configuration data into pods
- data stored in `ConfigMap` can be referenced in volume of type `configMap`
  - and then consumed by containerized applications running in a pod
  - referene a ConfigMap by providing the name of the ConfigMap in the volume
  - e.g. mount the `log-config` `ConfigMap` onto a pod called `configmap-pod`:

```
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
    - name: test
      image: busybox:1.28
      command: ['sh', '-c', 'echo "The app is running!" && tail -f /dev/null']
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: log-config
        items:
          - key: log_level
            path: log_level.conf
```

- ...
  - `log-config` `ConfigMap` is mounted as a volume
  - all contents stored in its `log_level` entry are in pod at `/etc/config/log_level.conf`
  - this path is derived from the volume's `mountPath` and path keyed with `log_level`
  - fyi `ConfigMap` is always mounted readonly
  - fyi `subPath` mounting of a `ConfigMap` won't be updated
  - fyi text is exposed as UTF-8, see `binaryData` for other encodings

### downwardAPI

- makes "info about pod" available to applications [see here](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/)
  - readonly files in plain text
- fyi `subPath` `downwardAPI` won't be updated

### emptyDir

- `emptyDir` volume created on pod assignment to a node and deleted on pod removal
- all containers in the pod can read and write to same files in this volume
- fyi data is safe across container crashes (as it does not remove the pod from a node)
- cases
  - scratch space e.g. a disk-based merge sort
  - checkpointing a long computation for recovery from crashes
  - holding files that a content-manager container fetches while a webserver container serves the data
- `emptyDir.medium` field controls where `emptyDir` volumes are stored
  - default: medium backing the node i.e. disk, SSD, network storage
  - `Memory`: a tmpfs (RAM-backed filesystem) is mounted (fast but limiting the container memory)
- `emptyDir.sizeLimit` caution: read up on memory size allocation in this case due to ramifications

```yaml
# normal configuration
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
    - image: registry.k8s.io/test-webserver
      name: test-container
      volumeMounts:
        - mountPath: /cache
          name: cache-volume
  volumes:
    - name: cache-volume
      emptyDir:
        sizeLimit: 500Mi

---
# memory configuration
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
    - image: registry.k8s.io/test-webserver
      name: test-container
      volumeMounts:
        - mountPath: /cache
          name: cache-volume
  volumes:
    - name: cache-volume
      emptyDir:
        sizeLimit: 500Mi
        medium: Memory
```

### fc (fibre channel)

- mounting existing fibre channel (???) block storage volume in a pod

### hostPath

- mounts a file or directory from the host node's filesystem into your pod
- _warning: security risks, do not use if possible_
- [see for more details](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath)

### iscsi

- mount iSCSI (SCSI over IP) volume (???)
- _warning: must have your own iSCSI server running with the volume created_

### local

- mounted local storage device such as a disk, partition or directory
- can only be used as a statically created PersistentVolume
- fyi: is better alternative to `hostPath`
- _caution: if a node becomes unhealthy `local` volume becomes inaccessible by the pod_

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - example-node
```

- ...
  - `nodeAffinity` must be set when using `local` volumes
    - Kubernetes scheduler uses `nodeAffinity` to schedule these pods to the correct node (???)
  - `volumeMode` (default: `Filesystem`) can be set to `Block` to expose as raw block device (???)
- recommended: create a StorageClass with `volumeBindingMode` set to `WaitForFirstConsumer`
  - (???) [read up](https://kubernetes.io/docs/concepts/storage/volumes/#local)
- _caution: requires manual cleanup and deletion by the user (if no external static provisioner)_

### nfs (Network File System)

- mount existing NFS (Network File System)
- _warning: must have existing NFS_

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
    - image: registry.k8s.io/test-webserver
      name: test-container
      volumeMounts:
        - mountPath: /my-nfs-data
          name: test-volume
  volumes:
    - name: test-volume
      nfs:
        server: my-nfs-server.example.com
        path: /my-nfs-volume
        readOnly: true
```

### persistentVolumeClaim

- a way for "claim" durable storage (e.g. iSCSI volume) without knowing the details of cloud environment

### projected

- maps several existing volume sources into the same directory

### secret

- used to pass sensitive information e.g. passwords
- fyi: backed by tmpfs (a RAM-backed filesystem) i.e. never written to non-volatile storage
- fyi: must create a Secret in the Kubernetes API before you can use it
  - a Secret is always mounted as `readOnly`
  - fyi: `subPath` mounted secrets won't update

## Using `subPath`

- share one volume for multiple uses in a single pod
- `volumeMounts[*].subPath` specifies a sub-path inside the referenced volume instead of its root
- example: LAMP (Linux Apache MySQL PHP) using a single volume
  - _warning: not recommended for prod_
  - PHP app code and assets map to the html folder
  - MySQL is stored in mysql folder

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-lamp-site
spec:
  containers:
    - name: mysql
      image: mysql
      env:
        - name: MYSQL_ROOT_PASSWORD
          value: "rootpasswd"
      volumeMounts:
        - mountPath: /var/lib/mysql
          name: site-data
          subPath: mysql
    - name: php
      image: php:7.0-apache
      volumeMounts:
        - mountPath: /var/www/html
          name: site-data
          subPath: html
  volumes:
    - name: site-data
      persistentVolumeClaim:
        claimName: my-lamp-site-data
```

### subPath with expanded environment variables

- `subPathExpr` field constructs `subPath` directory names from downward API environment variables
  - _warning: `subPath` and `subPathExpr` properties are mutually exclusive_
- example: pod uses `subPathExpr` to create a directory pod1 within the `hostPath` volume /var/log/pods
  - `hostPath` volume takes the pod name from the downwardAPI
  - host directory /var/log/pods/pod1 is mounted at /logs in the container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
    - name: container1
      env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
      image: busybox:1.28
      command:
        [
          "sh",
          "-c",
          "while [ true ]; do echo 'Hello'; sleep 10; done | tee -a /logs/hello.txt",
        ]
      volumeMounts:
        - name: workdir1
          mountPath: /logs
          # The variable expansion uses round brackets (not curly brackets).
          subPathExpr: $(POD_NAME)
  restartPolicy: Never
  volumes:
    - name: workdir1
      hostPath:
        path: /var/log/pods
```

# TODO

- [rest of storage docs](https://kubernetes.io/docs/concepts/storage/)
