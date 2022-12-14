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

![Image](/images/shop-ui.png)

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

![Image](/images/prom-UI.png)

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

## Let's begin 
we are going to configure our monitoring stack to notify us whenever something unexpected happens.  

### use case 1 : cluster monitoring
- CPU usage > 50%
- Pod cannot start

`Step 1 :` Create Alert rules 

It consists of defining define alert conditions based on Prometheus expression language expressions and to send notifications about firing alerts to an external service

`How do we create Alert rules?`

Because Prometheus is running in our cluster as a Prometheus operator which extends the Kubernetes API, we can use `Custom resources`. So we create our resources and the operator takes it and tells Prometheus to relaod the alert rules. 

Here is an example rule for `CPU usage > 50%`

```
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: main-rules
  namespace: monitoring
  labels:
    app: kube-prometheus-stack 
    release: monitoring
spec:
  groups:
  - name: main.rules
    rules:
    - alert: HostHighCpuLoad
      expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100) > 50
      for: 2m
      labels:
        severity: warning
        namespace: monitoring
      annotations:
        description: "CPU load on host is over 50%\n Value = {{ $value }}\n Instance = {{ $labels.instance }}\n"
        summary: "Host CPU load high"
```

After create the resource in the cluster, we can see now in the UI that our Alert rule has been add to the existing list. 

![Image](/images/cpu-usage.png)


Now we have the Alert rule configured, Let's test to see if if the `Prometeus will fire an alert` (send the alert to the AlertManager)

```
kubectl run cpustress --image containerstack/cpustress -- --cpu 4 --timeout 60s --metrics-brief
```

![Image](/images/cpu-stress-fired.png)

### use case 2 : 3rd party application monitoring

1- Deploy `Redis Exporter`

his job is : 
- get metrics data from the redis service, by ussing the given endpoint (redis://redis-cart:6379)
- translate these metrics to Prometheus understandable metrics 
- expose the translated metrics under `/metrics` endpoint. 

For this project, we are going to use Helm to deploy the exporter application and all required configuration

More details on :
- https://github.com/oliver006/redis_exporter

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add stable https://charts.helm.sh/stable
helm repo update

helm install redis-exporter prometheus-community/prometheus-redis-exporter -f redis-values.yaml
```
Create Alert rules 

In our case, we are going to monitor our Redis application to when : 
- Redis is down 
- there are too many connections 

```
rules:
  - alert: RedisDown
    expr: redis_up == 0
    for: 0m
    labels:
      severity: critical
    annotations:
      summary: Redis down (instance {{ $labels.instance }})
      description: "Redis instance is down\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
  - alert: RedisTooManyConnections
    expr: redis_connected_clients > 100
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: Redis too many connections (instance {{ $labels.instance }})
      description: "Redis instance has {{ $value }} connections\n LABELS = {{ $labels }}"
```
    
![Image](/images/redis-rules.png)


## configure Alertmanager
