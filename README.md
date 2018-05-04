**!!!THIS IS FOR DEVELOPMENT AND ONLY WORKS ON SINGLE NODE CLUSTERS!!!**

# TL;DR

Start a fresh cluster in the KubeVirt src tree.
```
export KUBEVIRT_NUM_NODES=1
make cluster-up && make cluster-sync
```

Install magic hostpath storage class, CDI, and post a PVC that will have cirros imported to it. 
```
kubectl apply -f "https://raw.githubusercontent.com/davidvossel/hostpath-pvc-vm-disks-examples/master/storage-setup/install.yaml"
kubectl apply -f "https://raw.githubusercontent.com/davidvossel/hostpath-pvc-vm-disks-examples/master/cdi-setup/install.yaml"
kubectl apply -f "https://raw.githubusercontent.com/davidvossel/hostpath-pvc-vm-disks-examples/master/vm-examples/pvc-cirros.yaml"
```

This gives you VM running that consumes the PVC you just created
```
kubectl apply -f "https://github.com/davidvossel/hostpath-pvc-vm-disks-examples/blob/master/vm-examples/vm-cirros.yaml"
```

# KubeVirt PVC Disk Imports for Dev Environments

This gives developers a path for testing KubeVirt PVC disk import work flows
without requiring any block storage. No Gluster/Ceph/Cinder+Ceph/etc... 

Just post a couple of manifests and you're good to go.

## Install Hostpath StorageClass provisioner

The hostpath storageclass lets you create PVs on the local host's filesystem.
This gives us dynamic provisioning of PVCs using local storage.

This manifest is reusing the containers in this project
https://github.com/MaZderMind/hostpath-provisioner

```
kubectl apply -f "https://raw.githubusercontent.com/davidvossel/hostpath-pvc-vm-disks-examples/master/storage-setup/install.yaml"
```

## Install the Containerized Data Importer (CDI)

CDI looks for PVCs with special annotations and injects data into them. In our case
we use CDI to inject disk images into PVs.

```
kubectl apply -f "https://raw.githubusercontent.com/davidvossel/hostpath-pvc-vm-disks-examples/master/cdi-setup/install.yaml"
```

## Import Disk and Start VM.


Example PVC: Post the spec below. The hostpath storageclass provisioner will
dynamically create a PV to satisfy the claim. After that, watch the CDI
deployment automaticall automatically spin up a pod to inject the PV
with the cirros disk.
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: "cirros-pvc"
  labels:
    app: containerized-data-importer
  annotations:
    kubevirt.io/storage.import.endpoint: "https://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img"
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi

```

Example VM: Once CDI pod is done, post the manifest below and you can start a
VM running the Cirros image that CDI imported to the PV. 

```
apiVersion: kubevirt.io/v1alpha1
kind: VirtualMachine
metadata:
  creationTimestamp: null
  name: vm-test-pvc
spec:
  domain:
    devices:
      disks:
      - disk:
          bus: virtio
        name: pvcdisk
        volumeName: pvcvolume
    machine:
      type: ""
    resources:
      requests:
        memory: 64M
  terminationGracePeriodSeconds: 0
  volumes:
  - name: pvcvolume
    persistentVolumeClaim:
      claimName: cirros-pvc
status: {}
```
