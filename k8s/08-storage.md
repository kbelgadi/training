# Docker storage

storage drivers
volume drivers

Docker install --> FS created: /var/lib/docker/
                                            aufs
                                            containers
                                            image
                                            volumes

docker build :
    --> each line in Dockerfile create a docker layer
    --> docker will not rebuild a layer that has been previously built
    --> layers are readonly - cannot modify them after build

docker run:
    --> docker create a new writable "container layer" on top of the image layer 
    --> container layer is used to store data created by the container: logs, temporary files, etc
    --> container destroyed --> container layer destroyed

if file is modified in readonly layers 
    --> a copy of the file is created in container layer 
    --> copy on write mechanism

container destroyed:
    --> files in container layer destroyed too

To persist files after container destroy: 
1. create a volume
    `docker volume create my_volume`
    --> folder created: /var/lib/docker/volumes/my_volume
2. run the container by mounting the create volume:
    `docker run -v my_volume:/mnt/data my_image`
    --> this is called **volume mounting**
NB: 
* if volume not created before, then docker will create it.
* if the volume location exists, run the container by mounting the existing location as a volume:
    `docker run -v /opt/data:/mnt/data my_image`
    --> this is called **bind mounting**

NB2:
`-v` option is obsolete, the new way: use `--mount` 
`docker run ---mount type=bind,src=/opt/data,target=/mnt/data my_image`

Storage driver:
* maintains layers architecture
* creating a writable layer
* moving file from readonly layers to writable layer
...

Storage drivers :
* AUFS
* ZFS
* BTRFS
* Device Mapper
* Overlay
* Overlay2
(Each with a level of performance and stability)

Election of the storage driver depends on OS
Ubuntu --> default storage driver: AUFS
If AUFS not available, Device Mapper may be a better storage driver option

# Volume driver

Volumes handled by volume drivers plugins

default volume driver plugin: Local
    --> help create volumes on docker host
    --> on: `/var/lib/docker/volumes`

Other volume drivers to create volumes on third party solutions:
Azure file storage
Convoy
DigitalOcean Block Storage
Flocker
GCE-docher
GlusterFS
Portworx
RexRay
VMware VSphere Storage
...

Some volume drivers support multiple providers
Ex, RexRay: AWS EBS, S3, EMC, openstack cinder
`docker run --name mysql --volume-driver rexray/ebs --mount src=ebs-vol,target=/var/lib/mysql mysql`

# CSI Container Storage Interface

In the past, k8s used docker as cont engine
--> all the code to work with docker was embeded into k8s source code

CRI Container Runtime Interface
    --> how k8s comunicate with multiple containers engines (docker, Rkt, CRI-O...) 
    --> new containers engines implements CRI to be compatible with k8s
CNI Container Network Interface
    --> new networking vendors (weaveworks, flannel, cilium...) implement plugin based on CNI standard to work with k8s

CSI Container Storage Interface
    --> new storage provider (Dell EMC, GlusterFS, Portworx, Amazon EBS...) has it own CSI driver

CSI not pecific k8s, it allows any containers orchestration (k8s, cloud foundry, mesos) solution to work with any storage vendor

CSI defines a set of RPC 
    --> Ex: orchestrator calls controllerPublishVolume() to place a workload that uses a vol on a node --- volume provider makes the volume available on a node


# K8s volumes

Docker containers are transient: data created into the container are deleted
    To persist data in container --> attach volume to the container

k8s pod are transient: delete pod => data created into the pod are deleted
    To persist data in pod --> attach volule to pod

```
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: registry.k8s.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /mnt/data
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      path: /data
      type: Directory   
```

hostPath --> ok for single nodes, not recommended for multi-nodes

k8s supports other storage solutions compatible with multi-nodes: NFS, GlusterFS, Flocker, etc

Example:

```
apiVersion: v1
kind: Pod
metadata:
  name: test-ebs
spec:
...
  volumes:
  - name: test-volume
    # This AWS EBS volume must already exist.
    awsElasticBlockStore:
      volumeID: "<volume id>"
      fsType: ext4
```

# Peristent volumes

admin create large pool of storages: persistent volumes
    ... and users pick from it: persistent volumes claims

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
    accessModes:
        - readWriteOnce
    capacity:
        storage: 1Gi
    hostPath:
      path: /data
```

accessModes
* ReadOnlyMany: 
* readWriteOnce
* ReadWriteMany

# Persistent Volume Claim

user create a PVC --> k8s finds the PV that fit the claim based on the request and properties (ex: capacity, access modes).

1 PVC --- 1 PV

to bind PVC to specific PV: 
* labels on PV 
* selector on PVC

if no PV match the PVC request --> PVC ramains in pending state

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc1
spec:
    accessModes:
        - readWriteOnce
    resources:
        requests:
            storage: 500Mi
```

delete PVC -->
    default: PV retain (ready for reuse by other PVC) --> not deleted until admin deletes it
    if reclaimPolicy Delete --> PV deleted after PVC deleted
    if reclaimPolicy Recycle --> data in PV scrapped before to make PV available for other PVCs

Use of PVC in a pod:
```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```

# Storage classes

static provisioning: before PV create, create the disk (ex. on gcp or ebs on aws)

dynamic provisioning: 
    --> define a storage provisioner, 
        declare it in a storage class, 
        use it on PVC manifest to provision a volume on claim with the required size

GCE storage class:
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: google-storage-example
provisioner: kubernetes.io/gce-pd
```

PVC using previous storage class:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc1
spec:
    accessModes:
        - readWriteOnce
    storageClassName: google-storage-example
    resources:
        requests:
            storage: 500Mi
```

there are other provisionners: azureFile, AWSElasticBlocStorage, CephFS, PortworxVolume...

each provisioner has its specific parameters to declare when creation storage class: volume type (standard,ssd), replication frequency, etc



