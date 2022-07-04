
# Installing Cloud Pak for Integration 2022.2.1 using CLI


Those are the notes taken during the installation on the existing OpenShift 4.10
The following storage classes were available in the cluster:
```
NAME                        PROVISIONER        
managed-nfs-storage         icpd-nfs.io/nfs      
rook-ceph-block (default)   rook-ceph.rbd.csi.ceph.com
```
The operators are installed in such a way that they are visible in all namespaces (namespace *openshift-operators*). The instances of all capabilities are installed in the namespace *cp4i*. 

Please correct all those parameters to fit your environment and needs.

Please refer to the documentation for more options: <br>
https://www.ibm.com/docs/en/cloud-paks/cp-integration/2022.2?topic=installing



## Catalog sources

Install:
```
cat << EOF | oc apply -f -
---
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: ibm-operator-catalog
  namespace: openshift-marketplace
spec:
  displayName: IBM Operator Catalog
  image: 'icr.io/cpopen/ibm-operator-catalog:latest'
  publisher: IBM
  sourceType: grpc
  updateStrategy:
    registryPoll:
      interval: 45m
EOF
```

Verify:
```
oc get pods -n openshift-marketplace
```

Wait until pod **ibm-operator-catalog-....** is ready. 


## Install operators

Install
```
cat << EOF | oc apply -f -
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: ibm-cp-integration
  namespace: openshift-operators
spec:
  channel: latest
  installPlanApproval: Automatic
  name: ibm-cp-integration
  source: ibm-operator-catalog
  sourceNamespace: openshift-marketplace
EOF
```

Verify
```
oc get csv -n openshift-operators
```

Wait until the operators for individual capabilities are ready (in **Succeeded** state):
```
NAME                                          DISPLAY                                                                                        VERSION   REPLACES                              PHASE
datapower-operator.v1.6.0                     IBM DataPower Gateway                                                                          1.6.0                                           Succeeded
ibm-apiconnect.v3.0.0                         IBM API Connect                                                                                3.0.0                                           Succeeded
ibm-appconnect.v5.0.0                         IBM App Connect                                                                                5.0.0                                           Succeeded
ibm-aspera-hsts-operator.v1.5.0               IBM Aspera HSTS                                                                                1.5.0                                           Succeeded
ibm-cloud-databases-redis.v1.5.2              IBM Operator for Redis                                                                         1.5.2     ibm-cloud-databases-redis.v1.5.1      Succeeded
ibm-common-service-operator.v3.19.0           IBM Cloud Pak foundational services                                                            3.19.0    ibm-common-service-operator.v3.18.0   Succeeded
ibm-cp-integration.v2.0.1                     Install all IBM Cloud Pak for Integration operators                                            2.0.1     ibm-cp-integration.v2.0.0             Pending
ibm-eventstreams.v3.0.2                       IBM Event Streams                                                                              3.0.2     ibm-eventstreams.v3.0.1               Succeeded
ibm-integration-asset-repository.v1.5.0       IBM Automation Foundation assets (previously IBM Cloud Pak for Integration Asset Repository)   1.5.0                                           Succeeded
ibm-integration-operations-dashboard.v2.6.0   IBM Cloud Pak for Integration Operations Dashboard                                             2.6.0                                           Succeeded
ibm-integration-platform-navigator.v6.0.0     IBM Cloud Pak for Integration                                                                  6.0.0                                           Succeeded
ibm-mq.v2.0.0                                 IBM MQ                                                                                         2.0.0                                           Succeeded
```

>**Note:** It could happen that the "top-level" operator **ibm-cp-integration** (the one that we have just used) remains in a *Pending* state forever. That is OK. This operator is used just to fire the installation of other operators. After that, it is not needed anymore. You can even delete it.


## Create namespace (project) for capabilities instances

Create project (e.g. *cp4i*)
```
oc new-project cp4i
```

Obtain the IBM Entitlement Key: <br>
https://myibm.ibm.com/products-services/containerlibrary

For easier manipulation store it to a variable:
```
export ENTITLEMENT_KEY= ...
```

Optionally test if you can connect to the IBM registry
```
docker login cp.icr.io --username cp --password $ENTITLEMENT_KEY

```
or, if you are using podman:
```
podman login cp.icr.io --username cp --password $ENTITLEMENT_KEY
```

Create a secret in your namespace
```
oc create secret docker-registry ibm-entitlement-key --docker-username=cp --docker-password=$ENTITLEMENT_KEY --docker-server=cp.icr.io --namespace=cp4i
``` 

>**Note:** If you decided to use different namespaces for your instances, create the secret in each of them.


## Create instance of the Platform Navigator

Create instance (change the name and storage class to fit your requirements -
**please note that you have to accept license**):
```
cat << EOF | oc apply -f -
---
apiVersion: integration.ibm.com/v1beta1
kind: PlatformNavigator
metadata:
  name: integration-navigator
  namespace: cp4i
spec:
  license:
    accept: true
    license: L-RJON-CD3JKX
  storage:
    class: managed-nfs-storage
  mqDashboard: true
  replicas: 1
  version: 2022.2.1
EOF
```

Verify:
```
oc get PlatformNavigator -n cp4i
```
Please note that it can take up to **45 minutes** before the Navigator is ready. 
```
NAME                    REPLICAS   VERSION      READY   LASTUPDATE   AGE
integration-navigator   1          2022.2.1-0   True    38s          27m
```

To obtain the platform navigator URL run:
```
oc describe PlatformNavigator integration-navigator -n cp4i
```
Note the **UI Endpoint** value.


To obtain the **admin** password, run:
```
oc get secrets -n ibm-common-services platform-auth-idp-credentials -ojsonpath='{.data.admin_password}' | base64 -d && echo ""
```


## Create capabilities instances

>**Important:** Please note that there are several templates available for each of the capability instances. In most of the cases, we are using here the basic, development, and non-production configurations. Please see the documentation (https://www.ibm.com/docs/en/cloud-paks/cp-integration/2022.2?topic=installing-deploying-instances-capabilities) or the user interface of your Platform Navigator for more combinations.

Please change the storage class names to fit your environment. We used here *managed-nfs-storage* where the file type and *rook-ceph-block* where the block type is needed.

Note also that the **license has to be accepted** for each instance.

### API Management

```
cat << EOF | oc apply -f -
apiVersion: apiconnect.ibm.com/v1beta1
kind: APIConnectCluster
metadata:
  labels:
    app.kubernetes.io/instance: apiconnect
    app.kubernetes.io/managed-by: ibm-apiconnect
    app.kubernetes.io/name: apiconnect-minimum
  name: apic1
  namespace: cp4i
spec:
  license:
    accept: true
    use: nonproduction
    license: L-RJON-CEBLEH
  profile: n1xc7.m48
  version: 10.0.5.0
  storageClassName: rook-ceph-block
EOF
```

### Automation assets

```
cat << EOF | oc apply -f -
apiVersion: integration.ibm.com/v1beta1
kind: AssetRepository
metadata:
  name: asset-repo1
  namespace: cp4i
spec:
  license:
    accept: true
    license: L-RJON-CD3JKX
  replicas: 1
  storage:
    assetDataVolume:
      class: managed-nfs-storage
    couchVolume:
      class: rook-ceph-block
  version: 2022.2.1-0
EOF
```

### Event Streams

```
cat << EOF | oc apply -f -
apiVersion: eventstreams.ibm.com/v1beta2
kind: EventStreams
metadata:
  name: es1
  namespace: cp4i
spec:
  adminApi: {}
  adminUI: {}
  apicurioRegistry: {}
  collector: {}
  license:
    accept: true
    use: CloudPakForIntegrationNonProduction
  requestIbmServices:
    iam: true
    monitoring: true
  restProducer: {}
  strimziOverrides:
    kafka:
      authorization:
        authorizerClass: com.ibm.eventstreams.runas.authorizer.RunAsAuthorizer
        supportsAdminApi: true
        type: custom
      config:
        default.replication.factor: 3
        inter.broker.protocol.version: '3.2'
        log.cleaner.threads: 6
        log.message.format.version: '3.2'
        min.insync.replicas: 2
        num.io.threads: 24
        num.network.threads: 9
        num.replica.fetchers: 3
        offsets.topic.replication.factor: 3
      listeners:
        - authentication:
            type: scram-sha-512
          name: external
          port: 9094
          tls: true
          type: route
        - authentication:
            type: tls
          name: tls
          port: 9093
          tls: true
          type: internal
      metricsConfig:
        type: jmxPrometheusExporter
        valueFrom:
          configMapKeyRef:
            key: kafka-metrics-config.yaml
            name: metrics-config
      replicas: 3
      storage:
        type: ephemeral
    zookeeper:
      metricsConfig:
        type: jmxPrometheusExporter
        valueFrom:
          configMapKeyRef:
            key: zookeeper-metrics-config.yaml
            name: metrics-config
      replicas: 3
      storage:
        type: ephemeral
  version: 11.0.2
EOF
```

### Integration dasboard (App Connect Enterprise)

```
cat << EOF | oc apply -f -
apiVersion: appconnect.ibm.com/v1beta1
kind: Dashboard
metadata:
  name: ace-dash1
  namespace: cp4i
spec:
  license:
    accept: true
    license: L-APEH-CCHL5W
    use: CloudPakForIntegrationNonProduction
  pod:
    containers:
      content-server:
        resources:
          limits:
            cpu: 250m
            memory: 512Mi
          requests:
            cpu: 50m
            memory: 50Mi
      control-ui:
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 50m
            memory: 125Mi
  replicas: 1
  storage:
    size: 5Gi
    type: persistent-claim
    class: managed-nfs-storage
  useCommonServices: true
  version: 12.0-lts
EOF
```

### Integration design

```
cat << EOF | oc apply -f -
apiVersion: appconnect.ibm.com/v1beta1
kind: DesignerAuthoring
metadata:
  name: ace-design1
  namespace: cp4i
spec:
  couchdb:
    replicas: 1
    storage:
      size: 10Gi
      type: persistent-claim
      class: rook-ceph-block
  designerFlowsOperationMode: local
  license:
    accept: true
    license: L-APEH-CCHL5W
    use: CloudPakForIntegrationNonProduction
  replicas: 1
  useCommonServices: true
  version: 12.0-lts
EOF
```

### Integration tracing

```
cat << EOF | oc apply -f -
apiVersion: integration.ibm.com/v1beta2
kind: OperationsDashboard
metadata:
  labels:
    app.kubernetes.io/instance: ibm-integration-operations-dashboard
    app.kubernetes.io/managed-by: ibm-integration-operations-dashboard
    app.kubernetes.io/name: ibm-integration-operations-dashboard
  name: tracing1
  namespace: cp4i
spec:
  license:
    accept: true
    license: CP4I
  storage:
    configDbVolume:
      class: managed-nfs-storage
    sharedVolume:
      class: managed-nfs-storage
    tracingVolume:
      class: rook-ceph-block
  version: 2022.2.1-0-lts
EOF
```

### Messaging (MQ)

```
cat << EOF | oc apply -f -
apiVersion: mq.ibm.com/v1beta1
kind: QueueManager
metadata:
  name: mq1
  namespace: cp4i
spec:
  license:
    accept: true
    license: L-RJON-CD3JKX
    use: NonProduction
  queueManager:
    name: QUICKSTART
    resources:
      limits:
        cpu: 500m
      requests:
        cpu: 500m
    storage:
      queueManager:
        type: ephemeral
      defaultClass: rook-ceph-block
  template:
    pod:
      containers:
        - env:
            - name: MQSNOAUT
              value: 'yes'
          name: qmgr
  version: 9.3.0.0-r1
  web:
    enabled: true
EOF
```

