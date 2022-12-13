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

For more details on how to Deploy and Manage a Cluster on Linode Kubernetes Engine (LKE), [Here the docs](https://www.linode.com/docs/guides/deploy-and-manage-a-cluster-with-linode-kubernetes-engine-a-tutorial/).

## Deploy our microservices application in LKE

### [See more details about application](https://github.com/hotiaDiallo/deploy-shop-microservices)

<br />

![Image](/images/shop-archi.png)

<br />

```
kubectl create namespace online-shop
kubectl apply -f config-shop-ms.yaml -n online-shop
```

## Monitoring with Prometheus

To be able to use `Prometheus` to monitor our cluster, we need to deploy `Prometheus Operator Stack` 

But how to deploy Prometheus Operator Stack in our cluster. To do that, we have 3 options

### Option 1 : 
Create all configuration yaml files and deploy them in the right effort. But this method is `inefficient` and required `lot of effort`

### Option 2 : Using an operator 
Find out a `prometheus operator` and deploy it in K8S cluster. 

### Option 2 : Using Helm
We use `Helm` to deploy a prometheus oprator. This `more efficient`

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring
helm ls
```

[Link to the chart: https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack]

### Check Prometheus Stack deployed resources

![Image](/images/prom-stack.png)

## Access Prometheus UI

```
kubectl port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090 -n monitoring &
```

### Access Grafana

```
kubectl port-forward svc/monitoring-grafana 8080:80 -n monitoring &
user: admin
pwd: prom-operator
```

Now that we have deployed prometheus in our cluster, we need to ask ourselves, `What are we going to monitor?`

In our case, it is going to be: 
- The cluster : k8s components 
- The 3rd party applications (like Redis)
- our own applications

