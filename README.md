# Prometheus monitoring

<img width="40" src="images/prometheus.svg"/>

<br />

## Create a Kubernetes cluster

### Option 1 : create an EKS cluster 

```
eksctl create cluster
```

### Option 2 : create a cluster with Linode Kubernetes Engine (LKE)

After creating the cluster on Linode, use the command below to be able to connect to it locally. 

```
chmod 400 ~/Downloads/prom-cluster-kubeconfig.yaml
export KUBECONFIG=~/Downloads/prom-cluster-kubeconfig.yaml
```

![Image](/images/nodes.png "Clsuter setting on Linode")


