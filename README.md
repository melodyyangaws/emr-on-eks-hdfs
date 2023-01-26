# Deploying HDFS on AWS EKS

Some [RSS experiments in EMR on EKS](https://github.com/melodyyangaws/emr-on-eks-remote-shuffle-service) requires HDFS as the secondary shuffle storage. This repository contains the helm chart to simplify the HDFS installation.

Firstly, follow the [instructions here](https://github.com/melodyyangaws/emr-on-eks-remote-shuffle-service#infrastructure) to provision and configure an [AWS EKS cluster](https://aws.amazon.com/eks/) that the HDFS Helm Chart can be deployed on.

By default, this chart deploys:
* a single name node, configured with 1 x 50GB data volume backed by EBS
* 2 x data nodes, each configured with 2 x 550GB data volumes backed by Fsx for Lustre.
* configuration for a replication factor of 3
* dyanmically provisioned FSx for Lustre volume per datanode pod, ie. the PV & PVC are not required to create manually.

NOTE: to create a FSx file system on the fly, the datanode pod will be in pending status for approx. 5 minutes.

## Quick start
Spin up the namenode and datanode containers locally for a quick test. They automatically attach your local disk as the storage.
```bash
docker-compose -f docker/docker-compose.yaml up
```
Access the HDFS NameNode web UI at: http://localhost:9870

```
docker-compose -f docker/docker-compose.yaml  stop
```

## Deploy Helm chart

### Prerequisite
1. Install [kubectl & awscli](https://www.eksworkshop.com/020_prerequisites/k8stools/)
2. Deploy [FSx for Lustre CSI driver](https://docs.aws.amazon.com/eks/latest/userguide/fsx-csi.html) to an Amazon EKS
```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-fsx-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"
```
3. Create the storage class manifest
```bash
kubectl -f helm/fsx/storage-class.yaml
kubectl get sc
```

Before the deployment, adjust the [values.yaml](./helm/values.yaml) based on your EKS settings. For example, we use the `nodeSelector` to assign a specific compute instance type and the AZ-b to the HDFS cluster:

```yaml
  nodeSelector: 
    app: sparktest
    eks.amazonaws.com/nodegroup: c59b
```

### Deploy to EKS

```bash
helm install hdfs . -f ./values-eks-alb.yaml
helm test hdfs
```

### Scale the datanode
```bash
#EXAMPLE
kubectl scale statefulsets hdfs-datanode --replicas=3

kubectl scale statefulsets hdfs-datanode --replicas=<N number desired>
```

