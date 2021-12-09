# Registry and ODF

## usefull link
- https://access.redhat.com/documentation/en-us/red_hat_openshift_container_storage/4.8/html/deploying_openshift_container_storage_using_bare_metal_infrastructure/assembly_uninstalling-openshift-container-storage_rhocs#removing-openshift-container-platform-registry-from-openshift-container-storage-external_rhocs
- https://docs.openshift.com/container-platform/4.9/registry/securing-exposing-registry.html

## Exemple Registry empty (sans disque)
```
apiVersion: imageregistry.operator.openshift.io/v1
kind: Config
metadata:
  name: cluster
spec:
  logLevel: Normal
  rolloutStrategy: RollingUpdate
  operatorLogLevel: Normal
  requests:
    read:
      maxWaitInQueue: 0s
    write:
      maxWaitInQueue: 0s
  observedConfig: null
  managementState: Managed
  proxy: {}
  unsupportedConfigOverrides: null
  storage:
    emptyDir: {}
  # replica 1  
  replicas: 1
```

## Exemple de Registry avec Cephfs

### Ajouter un PVC cephfs
```
oc apply -f - <<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ocs4registry
  namespace: openshift-image-registry
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
  storageClassName: ocs-storagecluster-cephfs
  volumeMode: Filesystem
EOF
```

### Ajouter le PVC dans la CRD Config.imageregistry.operator.openshift.io/v1
```
apiVersion: imageregistry.operator.openshift.io/v1
kind: Config
metadata:
  name: cluster
spec:
  logLevel: Normal
  rolloutStrategy: RollingUpdate
  operatorLogLevel: Normal
  requests:
    read:
      maxWaitInQueue: 0s
    write:
      maxWaitInQueue: 0s
  observedConfig: null
  managementState: Managed
  proxy: {}
  unsupportedConfigOverrides: null
  storage:
    pvc:
      claim: ocs4registry
  replicas: 2
```

## Registry with AWS
``` 
apiVersion: imageregistry.operator.openshift.io/v1
kind: Config
metadata:
  name: cluster
spec:
  logLevel: Normal
  rolloutStrategy: RollingUpdate
  operatorLogLevel: Normal
  requests:
    read:
      maxWaitInQueue: 0s
    write:
      maxWaitInQueue: 0s
  observedConfig: null
  managementState: Managed
  proxy: {}
  unsupportedConfigOverrides: null
  httpSecret: >-
    88c177a8c51dae6b60a3082ef3d770edd5bb5f66c9d0bebe5920d135d91b928ca765f957a92570faa083f36c2a81786a4ac25c6fdac2cc9bbe74b781c6f8440e
  storage:
    managementState: Managed
    s3:
      bucket: cluster2-m6k6x-image-registry-us-east-2-qxjvareliyvtkgsfjlrwma
      encrypt: true
      region: us-east-2
      virtualHostedStyle: false
  replicas: 2
```

---

# Exemple de Build Image avec IS et un imagePullPolicy: Always

## Creer une image stream qui contiendra le build de l'image buildée

```
oc new-project build-image-foo

NAMESPACE=build-image-foo

oc apply -f - <<EOF
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  labels:
    app: ruby-lco
  name: ruby-lco
  namespace: ${NAMESPACE}
EOF
```

## Creer un Build-Config avec en output IS
```
oc apply -f - <<EOF
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: ruby-lco
  namespace: ${NAMESPACE}
spec:
  output:
    to:
      kind: ImageStreamTag
      name: ruby-lco:1.0
  source:
    git:
      ref: master
      uri: 'https://github.com/openshift/ruby-ex.git'
    type: Git
  strategy:
    type: Source
    sourceStrategy:
      from:
        kind: ImageStreamTag
        name: 'ruby:2.7'
        namespace: openshift
      env: []
EOF

oc start-build ruby-lco -n ${NAMESPACE}
```

## Resultat du build

Successfully pushed image-registry.openshift-image-registry.svc:5000/build-image-foo/ruby-lco@sha256:xxxxxx

## Aller voir le contenu dans le fs de la registry
```
oc project openshift-image-registry
oc get pod  | grep image
oc rsh image-registry-xxxxxx-xxxx

ls registry/docker/registry/v2/repositories/
```


## Creer un deployment qui pointe sur l'image buildée avec imagePullPolicy: Always pour forcer le démarrage
```
oc apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ruby-lco
  namespace: ${NAMESPACE}
spec:
  selector:
    matchLabels:
      app: httpd
  replicas: 1
  template:
    metadata:
      labels:
        app: httpd
    spec:
      containers:
        - name: httpd
          image: >-
            image-registry.openshift-image-registry.svc:5000/build-image-foo/ruby-lco:1.0
          ports:
            - containerPort: 8080
          imagePullPolicy: Always            
EOF
```
