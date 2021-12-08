# Gestion des quotas pour le stockage

## Usefull link

* https://kubernetes.io/docs/concepts/policy/resource-quotas/

* https://docs.openshift.com/container-platform/4.9/applications/quotas/quotas-setting-per-project.html#quotas-sample-resource-quota-definitions_quotas-setting-per-project

## Création d'un ressource quota sur ODF

```
oc delete project quota-foo
oc new-project quota-foo

cat <<EOF | oc create -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-foo-ocs-storagecluster-ceph-rbd
  namespace: quota-foo
spec:
  hard:
    # Across all persistent volume claims, the sum of storage requests cannot exceed this value.
    ## requests.storage: 

    # The total number of PersistentVolumeClaims that can exist in the namespace.
    ## persistentvolumeclaims:

    # Across all persistent volume claims associated with the <storage-class-name>
    ## <storage-class-name>.storageclass.storage.k8s.io/requests.storage
    ## <storage-class-name>.storageclass.storage.k8s.io/persistentvolumeclaims
    
    ocs-storagecluster-ceph-rbd.storageclass.storage.k8s.io/requests.storage: 5Gi
    ocs-storagecluster-ceph-rbd.storageclass.storage.k8s.io/persistentvolumeclaims: 2
    
EOF
```


## Test The total number of PersistentVolumeClaims

```
oc delete PersistentVolumeClaim rbd1
oc delete PersistentVolumeClaim rbd2
oc delete PersistentVolumeClaim rbd3
oc delete PersistentVolumeClaim rbd4

cat <<EOF | oc create -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rbd1
  namespace: quota-foo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: ocs-storagecluster-ceph-rbd
  volumeMode: Filesystem
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rbd2
  namespace: quota-foo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: ocs-storagecluster-ceph-rbd
  volumeMode: Filesystem
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rbd3
  namespace: quota-foo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: ocs-storagecluster-ceph-rbd
  volumeMode: Filesystem
EOF
```

## Test the sum of storage requests PersistentVolumeClaims

```
oc delete PersistentVolumeClaim rbd1
oc delete PersistentVolumeClaim rbd2
oc delete PersistentVolumeClaim rbd3
oc delete PersistentVolumeClaim rbd4

cat <<EOF | oc create -f -
apiVersion: v1432005

kind: PersistentVolumeClaim
metadata:
  name: rbd1
  namespace: quota-foo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
  storageClassName: ocs-storagecluster-ceph-rbd
  volumeMode: Filesystem
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rbd2
  namespace: quota-foo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
  storageClassName: ocs-storagecluster-ceph-rbd
  volumeMode: Filesystem
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rbd3
  namespace: quota-foo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
  storageClassName: ocs-storagecluster-ceph-rbd
  volumeMode: Filesystem
EOF

```

## Annexes

Liste de StorageClass ODF

```
# StorageClass ODF
## ocs-storagecluster-cephfs
## ocs-storagecluster-ceph-rbd
## openshift-storage.noobaa.io
```
