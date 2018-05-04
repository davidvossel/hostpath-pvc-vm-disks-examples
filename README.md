**!!!THIS IS FOR DEVELOPMENT AND ONLY WORKS ON SINGLE NODE CLUSTERS!!!**

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


Example PVC: Post this spec, and watch the CDI automatically spin up a pod to
inject the PV with the cirros disk.

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

Example VM: Once CDI pod is done, post this manifest and you can start a VM
using the PVC CDI imported to. 

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
