# CLOPS-Node-Problem-Detector

The node-problem-detector is daemonset. This tool aims to make various node problems visible to the upstream layers in cluster management stack. It is a daemon which runs on each node, detects node problems and reports them to apiserver.

[Github-Homepage](https://github.com/kubernetes/node-problem-detector)

[Helm Chart](https://github.com/deliveryhero/helm-charts/tree/master/stable/node-problem-detector)


**The Node Problem Detector can detect:**

  - container runtime issues
     - unresponsive runtime daemons
  - hardware issues:
     - bad CPU
     - bad memory
     - bad disk
  - kernel issues:
     - kernel deadlock conditions
     - corrupted file systems
     - unresponsive runtime daemons
  - infrastructure daemon issues:
     - NTP service outages

## Pre-Request 
 - [Cluster Access](https://gitlab.cloudifyops.com/devops_framework/eks_tf_jenkins/-/blob/main/README.md)
 - [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

| Deploy                       | install Type | Chart Version    | App Version |
| :---------                   | :-------     | :--------------- | :-----------|
| `Prometheus`                 | `Helm`       | 19.2.0           | v2.41.0     |
| `Grafana`                    | `Helm`       | 6.48.0           |  9.3.1      |
| `node-problem-detector`      | `Helm`       | 2.3.2            |  v0.8.12    |

 

## step 1 - Install Helm chart 
  [Helm](https://helm.sh/) charts are configuration ymls(Package Manager) which are used for managing the Kubernetes resources

  ```
  curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
  chmod 700 get_helm.sh
  ./get_helm.sh

  ```

## step 2 - Deploy Node Problem Detector
 git hub [link](https://github.com/deliveryhero/helm-charts/tree/master/stable/node-problem-detector)
 1. clone the repo 
  ```
   git clone https://github.com/deliveryhero/helm-charts

  ```
 2. Add Node Problem Dector helm chart repository
  ```
   helm repo add deliveryhero https://charts.deliveryhero.io/
 
  ```
 3. inside the helam chart directory
  ```
  cd /helm-charts/stable/node-problem-detector/
  ```
 4. To install the chart with the release name 
  ```
  helm install <release-name> deliveryhero/node-problem-detector -n <namespace>
  ```
 5. Verify helm list
 ```
 helm list -A
 ``` 
 6. verify the pods are running 

 ```
 kubectl get pods -n <namespace> 
 ```
 7. verify the logs & node problem_dector is started
```
kubectl logs <pod name> -n <namespace>
```
 8. inside the pods 
```
kubectl exec -it <pod name> /bin/bash -n <namespace>
```
 9. verify the logs 
```
 curl localhost:20257/metrics
```
![curl](https://gitlab.cloudifyops.com/devops-toolset/clops-node-problem-detector/-/raw/main/images/image__2_.png)
 10. change the values.yml  
```
 cd /helm-charts/stable/node-problem-detector/
 vi values.yml
 //change the metrics to true 
```
![metrics](https://gitlab.cloudifyops.com/devops-toolset/clops-node-problem-detector/-/raw/main/images/image__3_.png)
 11. verify the logs 
```
 helm upgrade <release name> deliveryhero/node-problem-detector -f values.yaml
```
 12. verify the revision

 ```
 helm list -A
 ```



## step 3 - Install Prometheus

  install the prometheus using the helm chart.
   
 1. Add Prometheus helm chart repository
 ```
 helm repo add prometheus-community https://prometheus-community.github.io/helm-charts 

 ```
 2. Update the helm chart repository
 ```
 helm repo update 

 ```
 3. Create prometheus namespace
```
kubectl create namespace prometheus

```
4. Install the prometheus
```
helm install prometheus prometheus-community/prometheus \
    --namespace prometheus \
    --set alertmanager.persistentVolume.storageClass="gp2" \
    --set server.persistentVolume.storageClass="gp2" 

```

5. Verify the pods are running 
```
kubectl get all -n prometheus 

```
6. View the Prometheus dashboard by forwarding the deployment ports
```
kubectl port-forward deployment/prometheus-server 9090:9090 -n prometheus

```
7. Verify the pod security group ports(9090) are open

8. local host
```
http://localhost:9090/graph
```

![prometheus](https://gitlab.cloudifyops.com/devops-toolset/clops-node-problem_dector/-/raw/main/images/Screenshot_2022-12-29_060932.jpg)

## step 4 - install grafana

install the Grafana using the Helm Chart

1. Add the Grafana helm chart repository
```
helm repo add grafana https://grafana.github.io/helm-charts 

```
2. Update the helm chart repository

```
helm repo update 
```
3. Now we need to create a Prometheus data source so that Grafana can access the Kubernetes metrics. Create a yaml file prometheus-datasource.yaml and save the following data source configuration into it 
```
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-server.prometheus.svc.cluster.local
      access: proxy
      isDefault: true

```
4. Create a namespace grafana
```
kubectl create namespace grafana

```
5. Install the Grafana
```
helm install grafana grafana/grafana \
    --namespace grafana \
    --set persistence.storageClassName="gp2" \
    --set persistence.enabled=true \
    --set adminPassword='<password>' \
    --values prometheus-datasource.yaml \
    --set service.type=LoadBalancer 

```
6. Verify the Grafana installation by using the following kubectl command
```
kubectl get all -n grafana
```
7.Public AWS IP of Grafana Kubernetes Server- To Access the Grafana dashboard we need to find Public AWS IP address and for that use the following command
```
kubectl get service -n grafana 

```
6. Get Grafana ELB URL using this command
```
export ELB=$(kubectl get svc -n grafana grafana -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

echo "http://$ELB"

```
7. When logging in, use the username `admin` and get the password hash by running the following

```
kubectl get secret --namespace grafana grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

```
8. Import Grafana dashboard from Grafana Labs

 Now we have set up everything in terms of Prometheus and Grafana. For the custom Grafana Dashboard, we are going to use the open source grafana dashboard

 On opensource grafana dashboard you can find many opensource dashboards which can be imported directly into the Grafana.
  - import a dashboard 6417 (Pods Monitoring Dashboard) and 3119 (Cluster Monitoring Dashboard)
  - setting -> datasource
  - save & Test
  ![grafana](https://gitlab.cloudifyops.com/devops-toolset/clops-node-problem_dector/-/raw/main/images/config_data_source.jpg)
  

9. import a dashboard 15549 (Node Problem Detector) -> select the prometheus as datasource
 
 Dashboard -> Link[https://grafana.com/grafana/dashboards/15549-node-problem-detector/]

10. Add to Dashboard Name it as Node Problem Dector 

  ![grafana-db](https://gitlab.cloudifyops.com/devops-toolset/clops-node-problem-detector/-/raw/main/images/npd.jpg)
  


## Error Troubleshooting

During the prometheus setup you might ran into issue where you are trying install the helm prometheus but may lear to following issue -

- pods are pending status (its a an persistentVolume claim)
- Refer [link](https://docs.aws.amazon.com/eks/latest/userguide/csi-iam-role.html) 

## Reference link

- [datree.io](https://www.datree.io/helm-chart/node-problem-detector-delivery-hero)
- [artifacthub.io](https://artifacthub.io/packages/helm/deliveryhero/node-problem-detector)
- [kubernetes.io](https://kubernetes.io/docs/tasks/debug/debug-cluster/monitor-node-health/)
- [stackoverflow](https://stackoverflow.com/questions/48134835/how-to-use-k8s-node-problem-detector)
- [metrics issue](https://github.com/kubernetes/node-problem-detector/issues/259)











