
# An example how to put software in a Container

## Build Images

### Build on RHEL 8 

```bash
podman build \
  -t own-apache:rhel7 \
  -f Dockerfile.rhel .
```

### Build on OpenShift 4

```bash
oc new-build https://github.com/openshift-examples/own-apache-container.git \
--name httpd

oc new-build https://github.com/openshift-examples/own-apache-container.git \
--context-dir=anyuid --name httpd-anyuid
```

#### Build with RHEL basic image on OpenShift 4

```bash
# Create imagestream
oc import-image rhel7:7.6 \
  --from=registry.access.redhat.com/rhel7/rhel:7.6  \
  --namespace openshift \
  --confirm

# Create entitlement secret
oc create secret generic etc-pki-entitlement \
    --from-file /etc/pki/entitlement/xxxxxx.pem \
    --from-file /etc/pki/entitlement/xxxxxx-key.pem 

oc apply -f - <<EOF
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  annotations:
    openshift.io/generated-by: OpenShiftNewBuild
  creationTimestamp: null
  labels:
    build: own-apache-container-rhel7
  name: own-apache-container-rhel7
spec:
  nodeSelector: null
  output:
    to:
      kind: ImageStreamTag
      name: own-apache-container-rhel7:latest
  postCommit: {}
  resources: {}
  source:
    git:
      uri: https://github.com/openshift-examples/own-apache-container.git
    type: Git
    # IMPORTANT: mount the rhel entitlement
    secrets:
    - secret:
        name: etc-pki-entitlement
      destinationDir: etc-pki-entitlement
  strategy:
    dockerStrategy:
      # IMPORTANT: to select the rhel dockerfile
      dockerfilePath: Dockerfile.rhel
      from:
        kind: ImageStreamTag
        name: rhel7:7.6
        namespace: openshift
    type: Docker
  triggers:
  - type: ConfigChange
  - imageChange: {}
    type: ImageChange
status:
  lastVersion: 0
EOF
```


# Deploy Images

```
oc new-app httpd
oc expose svc/httpd

oc new-app httpd-anyuid
oc expose svc/httpd-anyuid
# FAIL, because by default it is not allowed to run POD's with uid 0

# Create service account
oc create sa anyuid

# Add privileges to service account to run POD's with uid 0
oc adm policy add-scc-to-user -z anyuid anyuid 

oc patch dc/httpd-anyuid --patch '{"spec":{"template":{"spec":{"serviceAccount": "anyuid", "serviceAccountName": "anyuid"}}}}'

```
# Add persistent volume

```yaml
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    annotations:
        volume.beta.kubernetes.io/storage-provisioner: kubernetes.io/glusterfs
    name: httpd
spec:
    accessModes:
    - ReadWriteMany
    resources:
        requests:
            storage: 1Gi
    storageClassName: glusterfs-ocs
EOF
```

```bash
oc get pvc --watch

oc set volume dc/httpd --add --name=httpd-volume-1 -t pvc --claim-name=httpd --overwrite  --mount-path=/var/www/html/

oc set volume dc/httpd-anyuid --add --name=httpd-anyuid-volume-1 -t pvc --claim-name=httpd --overwrite  --mount-path=/var/www/html/
```

# Rollback 
```
oc rollback dc/httpd-anyuid
```