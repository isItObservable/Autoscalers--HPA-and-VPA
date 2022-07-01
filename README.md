# Is it Observable
<p align="center"><img src="/image/logo.png" width="40%" alt="Is It observable Logo" /></p>

## Episode dedicated to Autoscaling
<p align="center"><img src="/image/hpa-128.png" width="40%" alt="hpa" /> & <img src="/image/vpa.png" width="40%" alt="vpa" /></p>
This repository contains the files utilized during the tutorial presented in the dedicated IsItObservable episode related to Autoscaling
What you will learn
* How to deploy the custom metric server
* How to deploy the promehteus adapter
* How to configure the prometheus adpater to use a custom metrics
* How to take advantage of HPA 
* How to take advantage of VPA 


This repository showcase the usage of the VPA and HPA by using GKE with :
* the HipsterShop
* Prometheus
* The prometheus adapter
* Nginx ingress controller



## Prerequisite
The following tools need to be install on your machine :
- jq
- kubectl
- git
- gcloud ( if you are using GKE)
- Helm

## Deployment Steps in GCP 

You will first need a Kubernetes cluster with 2 Nodes.
You can either deploy on Minikube or K3s or follow the instructions to create GKE cluster:
### 1.Create a Google Cloud Platform Project
```
PROJECT_ID="<your-project-id>"
gcloud services enable container.googleapis.com --project ${PROJECT_ID}
gcloud services enable monitoring.googleapis.com \
    cloudtrace.googleapis.com \
    clouddebugger.googleapis.com \
    cloudprofiler.googleapis.com \
    --project ${PROJECT_ID}
```
### 2.Create a GKE cluster
```
ZONE=us-central1-b
gcloud container clusters create onlineboutique \
--project=${PROJECT_ID} --zone=${ZONE} \
--machine-type=e2-standard-2 --num-nodes=4
```

### 3.Clone the Github Repository
```
git clone https://github.com/isItObservable/Autoscalers--HPA-and-VPA
cd Autoscalers--HPA-and-VPA
```
### 4.Deploy Nginx Ingress Controller with the Prometheus exporter
```
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
  --set controller.enableLatencyMetrics=true --set prometheus.create=true
```

#### 1. get the ip adress of the ingress gateway
Since we are using Ingress controller to route the traffic , we will need to get the public ip adress of our ingress.
With the public ip , we would be able to update the deployment of the ingress for :
* hipstershop
* grafana
```
IP=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -ojson | jq -j '.status.loadBalancer.ingress[].ip')
```
#### 2. Update the manifest files

update the following files to update the ingress definitions :
```
sed -i "s,IP_TO_REPLACE,$IP," kubernetes-manifests/k8s-manifest.yaml
sed -i "s,IP_TO_REPLACE,$IP," grafana/ingress.yaml
sed -i "s,IP_TO_REPLACE,$IP," K6/load.yaml
```

### 5.Prometheus
#### 1.Deploy
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack --set sidecar.datasources.enabled=true --set sidecar.datasources.label=grafana_datasource --set sidecar.datasources.labelValue="1" --set sidecar.dashboards.enabled=true
```
#### 2. Configure Prometheus by enabling the feature remote-writer

To measure the impact of our experiments on use traffic , we will use the load testing tool named K6.
K6 has a Prometheus integration that writes metrics to the Prometheus Server.
This integration requires to enable a feature in Prometheus named: remote-writer

To enable this feature we will need to edit the CRD containing all the settings of promethes: prometehus

To get the Prometheus object named use by prometheus we need to run the following command:
```
kubectl get Prometheus
```
here is the expected output:
```
NAME                                    VERSION   REPLICAS   AGE
prometheus-kube-prometheus-prometheus   v2.32.1   1          22h
```
We will need to add an extra property in the configuration object :
```
enableFeatures:
- remote-write-receiver
```
so to update the object :
```
kubectl edit Prometheus prometheus-kube-prometheus-prometheus
```
After the update your Prometheus object should look  like :
```
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  annotations:
    meta.helm.sh/release-name: prometheus
    meta.helm.sh/release-namespace: default
  generation: 2
  labels:
    app: kube-prometheus-stack-prometheus
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: kube-prometheus-stack
    app.kubernetes.io/version: 30.0.1
    chart: kube-prometheus-stack-30.0.1
    heritage: Helm
    release: prometheus
  name: prometheus-kube-prometheus-prometheus
  namespace: default
spec:
  alerting:
  alertmanagers:
  - apiVersion: v2
    name: prometheus-kube-prometheus-alertmanager
    namespace: default
    pathPrefix: /
    port: http-web
  enableAdminAPI: false
  enableFeatures:
  - remote-write-receiver
  externalUrl: http://prometheus-kube-prometheus-prometheus.default:9090
  image: quay.io/prometheus/prometheus:v2.32.1
  listenLocal: false
  logFormat: logfmt
  logLevel: info
  paused: false
  podMonitorNamespaceSelector: {}
  podMonitorSelector:
  matchLabels:
  release: prometheus
  portName: http-web
  probeNamespaceSelector: {}
  probeSelector:
  matchLabels:
  release: prometheus
  replicas: 1
  retention: 10d
  routePrefix: /
  ruleNamespaceSelector: {}
  ruleSelector:
  matchLabels:
  release: prometheus
  securityContext:
  fsGroup: 2000
  runAsGroup: 2000
  runAsNonRoot: true
  runAsUser: 1000
  serviceAccountName: prometheus-kube-prometheus-prometheus
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector:
  matchLabels:
  release: prometheus
  shards: 1
  version: v2.32.1
```

#### 3. Deploy the Grafana ingress
```
kubectl apply -f grafana/ingress.yaml
```


#### 4. Get The Prometheus service
```
PROMETHEUS_SERVER=$(kubectl get svc -l app=kube-prometheus-stack-prometheus -o jsonpath="{.items[0].metadata.name}")
sed -i "s,PROMETHEUS_SERVER,$PROMETHEUS_SERVER," kubernetes-manifests/k8s-manifest.yaml
sed -i "s,PROMETHEUS_SERVER,$PROMETHEUS_SERVER," K6/load.yaml
```
#### 5. Deploy the Nginx ServiceMonitor
```
kubectl apply -f prometheus/servicemonitor.yaml -n ingress-nginx
```

#### 6. Deploy the Prometheus adapter
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install --name prom-adapter prometheus-community/prometheus-adapter --set prometheus.url=http://${PROMETHEUS_SERVER}.default.svc.cluster.local
```

When installed you can use the following command to see all the metrics that are now exposed to Kubernetes
```
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/"
```
It will show you all the metrics currently exposed by prometheus.


### 6. Deploy the hipstershop
```
kubectl create ns hipster-shop
kubectl apply -n hipster-shop -f kubernetes-manifests/k8s-manifest.yaml
```

### 7. HPA

#### 1. Discover custom metric 
The prometheus adapter preinstall a configuration file to map standard prometheus metrics.
```
kubectl get cm prom-adapter-prometheus-adapter
```
#### 2. Customize our Adapter

in our example we would like autoscale the workload of the hispter-shop based on the traffic.
The traffic would be measure with the help of the metrics exposed by the nginx exporter.
Let's use the number of request/s as the metric to trigger the austoscaling scenario.

Here is the PromQL that we need to apply:
sum(rate(nginx_ingress_controller_requests{status=~"[0-3]+",exported_namespace="hipster-shop"}[2m])) by (exported_service)

Here is the configuration that we will need to apply in the Prometheus Adapter : 
```
- seriesQuery: '{__name__="nginx_ingress_controller_requests", status=~"[0-3]+"}'
  seriesFilters: []
  resources:
    overrides:
      exported_namespace:
        resource: namespace
      exported_service:
        resource: service
  name: 
    matches: 'nginx_ingress_controller_requests'
    as: 'ingress_request'
  metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
 ```
Let's modify the prometheus adpater configuration file to add our new rule:
```
kubectl edit cm prom-adapter-prometheus-adapter
```
add you new rule with the exisiting rules.

To let the prometheus adapter takes our new rule, lets restart the prometheus adapter. ( in our case we will simply delete the prometheus adapter pod)
```
kubectl delete pod -l app.kubernetes.io/name=prometheus-adapter
```
#### 3. Deploy the HPA 
```
kubectl apply -f hpa/hpa.yaml
```

Now that we have an HPA in place, let's generate some extra load to reach our autoscaling threshold
```
kubectl apply -f k6/load.yaml
```

Now let's observe our HPA, in my case i would use Dynatrace .

### 8. VPA

#### 0. Requiements
If you are using GKE you will need to grand your current Google cluster admin role.
```
gcloud info | grep Account    # get current google identity
Account: [myname@example.org]

kubectl create clusterrolebinding myname-cluster-admin-binding --clusterrole=cluster-admin --user=myname@example.org
Clusterrolebinding "myname-cluster-admin-binding" created
```

Before deploying the VPA let's remove our HPA and stop our current load test :
```
kubectl delete -f k6/load.yaml
kubectl delete -f hpa/hpa.yaml
```

#### 1. Deploy VPA


THe deployment of the VPA is available in the following repository : https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler/hack
Therefore we first need to clone the following repository :
```
git clone https://github.com/kubernetes/autoscaler
cd autoscaler/vertical-pod-autoscaler/hack
./hack/vpa-up.sh
```
#### 2. Create VPA for the Frontend deployment in Recommendation mode
let's start the VPA in recommendation mode 
```
kubectl apply -f VPA/vpa.yaml
```
And now run a load test to see the suggested values of the vpa:
```
kubectl apply -f k6/load.yaml
```
let's wait few minutes and look at the suggested values:
```
kubectl describe vpa frontend-vpa -n hipster-shop
```
#### 3. Update the VPA
```
kubectl edit vpa frontend-vpa -n hipster-shop
```
Change the `UpdateMode` from `Off` to `Auto` 

### 9 [Optional] **Clean up**: TODO
```
gcloud container clusters delete onlineboutique \
    --project=${PROJECT_ID} --zone=${ZONE}
```




