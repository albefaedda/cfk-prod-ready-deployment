# Confluent for Kubernetes - from Dev to Production ready deployment

Deployment of Confluent for Kubernetes, step-by-step, PLAINTEXT -> SSL -> SASL/PLAIN + LDAP -> RBAC/MDS, with custom users for Confluent Platform components and added Monitoring with Prometheus and Grafana

## Prerequisites

### Create a namespace

```sh
kubectl create namespace confluent
```

### Switch context to this namespace

```sh
kubectl config set-context --current --namespace confluent
```

### Create a Storage Class

It is recommended to create a Storage class with a Retain reclaim policy so that the Persistent Volumes (PVs) are not deleted when the deployment is deleted but can be restored with a new deployment with no data loss.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: production-storage-class
provisioner: kubernetes.io/gce-pd
reclaimPolicy: Retain
allowVolumeExpansion: false
volumeBindingMode: WaitForFirstConsumer
```

### Install Confluent for Kubernetes Operator

If you're installing CFK in a private environment, with no internet connection, you need to download the helm chart and uncompress it on your installation machine. On top of this, you need to download the images for Confluent operator and all the Confluent Platform components and load the on your private registry (make sure to download the images for the correct architecture (e.g. ARM, AMD). You can then update the image information on the helm/confluent-for-kubernetes/values.yaml file.

```yaml
###
## Image pull secret
imagePullSecretRef:  myR3g1$7ry
## Confluent Operator Image Information
##
image:
  registry: image-registry.myorg.com
  repository: confluentinc/confluent-operator
  pullPolicy: IfNotPresent
  tag: "0.581.34"

```

 On the same file, if you want to use a pre-created service-account, you can set it as the one to use for the installation. 
 
 ```yaml
##
## ServiceAccount
## If enabled it will create, otherwise it will
## not create
##
serviceAccount:
  create: false
  name: "my-service-account"

 ```
 
 Once your configuration is ready, you can install the operator with helm: 

```sh
helm upgrade --install confluent-operator \
  helm/confluent-for-kubernetes/ \
  --namespace confluent
```

## Installation plan

In this example, we are going to deploy the Confluent Platform components in a GKE cluster with a node pool of 6 nodes. 
You can find a simple Terraform configuration for the creation of a GKE cluster [here](https://github.com/albefaedda/cc-hybrid-architecture-exercise/tree/main/gke-cluster-terraform).

The Brokers (3) and Zookeeper (3) instances will be spread in the cluster and placed each in their own node making use of Kubernetes topologySpreadConstraint and pod affinity/anti-affinity rules. 
No other rules for the other Confluent Platform components (Connect, KsqlDB, Schema Registry, Control Center, REST Proxy) - each will be deployed with one only instance.

For the deployment of the Confluent Platform components, we split the yaml configuration in two files:
- one containing Zookeeper and Kafka CRs
- one containing all the other Confluent Platform components CRs

Note: I'm limiting the CPU/memory/storage resources to low values for all the components

Installation steps:
- [plaintext](plaintext/README.md)
- [ssl encryption](ssl/README.md)
- [authentication with LDAP](authn-with-ldap/README.md)
- [authorisation with RBAC/MDS](authz-rbac-mds/README.md)
- [monitoring](monitoring/README.md)