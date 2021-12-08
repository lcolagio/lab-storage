# Registry and ODF

## usefull link
- https://access.redhat.com/documentation/en-us/red_hat_openshift_container_storage/4.8/html/deploying_openshift_container_storage_using_bare_metal_infrastructure/assembly_uninstalling-openshift-container-storage_rhocs#removing-openshift-container-platform-registry-from-openshift-container-storage-external_rhocs



## Registry empty

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

## Registry with Cephfs

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

# Exemple de Build Image argocdrepo avec le plug-in vault


## Creer une image stream qui contiendra le build de l'image argocd-vault-plugin

```
oc new-project build-image-foo

ARGOCD_VAULT_PLUGIN_NAMESPACE=build-image-foo

oc apply -f - <<EOF
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  labels:
    app: argocd-vault-plugin
  name: argocd-vault-plugin
  namespace: ${ARGOCD_VAULT_PLUGIN_NAMESPACE}
EOF
```

## Creer un build config

```
ARGOCD_REPO_SOURCE_IMAGE=registry.redhat.io/openshift-gitops-1/argocd-rhel8:v1.3.1

oc apply -f - <<EOF
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  labels:
    app: argocd-vault-plugin
  name: argocd-vault-plugin
  namespace: ${ARGOCD_VAULT_PLUGIN_NAMESPACE}
spec:
  output:
    to:
      kind: ImageStreamTag
      name: argocd-vault-plugin:131-150
  source:
    type: Dockerfile
    dockerfile: |
      FROM ${ARGOCD_REPO_SOURCE_IMAGE}
      USER root
      RUN curl -L -o /usr/local/bin/argocd-vault-plugin https://github.com/IBM/argocd-vault-plugin/releases/download/v1.5.0/argocd-vault-plugin_1.5.0_linux_amd64
      RUN chmod +x /usr/local/bin/argocd-vault-plugin
      USER argocd
  strategy:
    dockerStrategy:
      buildArgs:
      # - name: "NO_PROXY"
      #   value: "localhost,127.0.0.0,127.0.0.1,127.0.0.2,localaddress,.localdomain.com,.laposte.fr"
    type: Docker
EOF

oc start-build argocd-vault-plugin -n ${ARGOCD_VAULT_PLUGIN_NAMESPACE}
```

## Resultat du build

Successfully pushed image-registry.openshift-image-registry.svc:5000/build-image-foo/argocd-vault-plugin@sha256:d489c1e62201edfd5abe638375428885e117b46efdf9734d9470961748539160

```
oc get pod -n openshift-image-registry | grep image
```
