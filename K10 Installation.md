# Kasten K10

## Usefull link

* https://docs.kasten.io/install/openshift/operator.html
* https://blog.kasten.io/learn-the-best-way-to-install-kasten-k10-on-openshift

## Install K10

```
oc new-project kasten-io \
  --description="Kubernetes data management platform" \
  --display-name="Kasten K10"

helm repo add kasten https://charts.kasten.io/

kubectl annotate volumesnapshotclass ocs-storagecluster-cephfsplugin-snapclass k10.kasten.io/is-snapshot-class=true
kubectl annotate volumesnapshotclass ocs-storagecluster-rbdplugin-snapclass k10.kasten.io/is-snapshot-class=true

curl https://docs.kasten.io/tools/k10_primer.sh | bash

helm install k10 kasten/k10 --namespace=kasten-io \
    --set scc.create=true \
    --set secrets.awsAccessKeyId="${AWS_ACCESS_KEY_ID}" \
    --set secrets.awsSecretAccessKey="${AWS_SECRET_ACCESS_KEY}"
```


# Define route UI
```
OCP_CLUSTER=cluster2.sandbox1652.opentlc.com

oc delete route k10

cat <<EOF | oc apply -f -
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: k10
  namespace: kasten-io
spec:
  host: k10-kasten-io.apps.${OCP_CLUSTER}
  to:
    kind: Service
    name: gateway
    weight: 100
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: None
  wildcardPolicy: None
EOF

# Goto K10 UI
echo  https://k10-kasten-io.apps.${OCP_CLUSTER}/k10/#/
```


## Creation d'un bucket externe
```
export OADP_NAMESPACE=kasten-io
export OADP_OBJECT_STORAGE_CLAIM=kasten-io-obc
export NOOBA_BUCKET_CLASS=noobaa-default-bucket-class

# delete obc
oc delete obc ${OADP_OBJECT_STORAGE_CLAIM}
oc delete secret ${OADP_OBJECT_STORAGE_CLAIM}-credentials

# create OBC Noobaa
oc create -f -<<EOF
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: ${OADP_OBJECT_STORAGE_CLAIM} 
  namespace: ${OADP_NAMESPACE}
spec:
  storageClassName: openshift-storage.noobaa.io
  generateBucketName: ${OADP_OBJECT_STORAGE_CLAIM} 
  additionalConfig:
    bucketclass: ${NOOBA_BUCKET_CLASS}
EOF
```

### Create crendential
```
export OADP_BUCKET_NAME=$(oc get cm ${OADP_OBJECT_STORAGE_CLAIM} -n ${OADP_NAMESPACE} -o jsonpath="{.data.BUCKET_NAME}")
export NOOBAA_S3_ENDPOINT=$(oc get route s3 -n openshift-storage -o jsonpath="{.spec.host}")
export NOOBAA_S3_ACCESS_KEY_ID=`oc get secret -n openshift-storage noobaa-admin -o jsonpath='{.data.AWS_ACCESS_KEY_ID}'|base64 -d`
export NOOBAA_S3_SECRET_ACCESS_KEY=`oc get secret -n openshift-storage noobaa-admin -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}'|base64 -d`

echo ${OADP_BUCKET_NAME}
echo ${NOOBAA_S3_ENDPOINT}
echo ${NOOBAA_S3_ACCESS_KEY_ID}
echo ${NOOBAA_S3_SECRET_ACCESS_KEY}
```

# Create alias s3cmd
```
alias s3-noobaa-k10='AWS_ACCESS_KEY_ID=${NOOBAA_S3_ACCESS_KEY_ID} AWS_SECRET_ACCESS_KEY=${NOOBAA_S3_SECRET_ACCESS_KEY} aws --endpoint https://${NOOBAA_S3_ENDPOINT} --no-verify-ssl s3'
s3-noobaa ls | grep ${OADP_OBJECT_STORAGE_CLAIM}
```
