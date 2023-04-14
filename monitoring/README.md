## Monitoring 

For us to be able to monitor the Confluent Platform components, we need to add some additional configuration to the Confluent Platform components configuration in order to collect metrics from Prometheus.

Once updated the components configuration, We can re-deploy them with the usual commands: 

```
kubectl apply -f monitoring/deployment/zk+brokers.yaml
kubectl apply -f monitoring/deployment/other-cp-components.yaml
```

For the monitoring tools, we can deploy them to the same namespace or we can create a new namespace just for the monitoring for a separation of concerns. 

To create a new namespace, use the command: 

```sh
kubectl create namespace monitoring
```

To move the currenct context to the new namespace, use the command: 

```sh
kubectl config set-context --current --namespace=monitoring
```

### Prometheus and Grafana

To monitor the Confluent Platform we are going to use two external services: 
- Prometheus to collect JMX metrics 
- Grafana to visualise these metrics in graphs and dashboards

To install Prometheus and Grafana, there are multiple articles and multiple ways to do it.

#### Prometheus
 
For Prometheus we are going to use a helm chart which is available to download from the Prometheus Community [github repo](https://prometheus-community.github.io/). 

To install it, we use the helm commands: 

```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm repo update

helm install [release-name] prometheus-community/prometheus  
```

If you need to customise your installation, you can download the helm charts, make the changes you need and then run the helm install command pointing helm to your customised repo dir.

You can find a downloaded version of the Prometheus helm charts in the [prometheus](prometheus) folder which you can install by simply running the following command: 

```sh
helm install [release-name] ./prometheus
```
 
#### Grafana

For the Grafana installation, we are going to follow the [guide](https://grafana.com/docs/grafana/latest/setup-grafana/installation/kubernetes/) available in Grafana website which allows to easily create a deployment of Grafana community edition. 

You can find an example of the Grafana manifest for Kubernetes in the [grafana](grafana) folder. To deploy this on Kubernetes, run the following command: 

```sh
kubectl apply -f grafana/grafana-deployment.yml
```

Get Grafana `admin`'s password with this command: 
```
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Once the deployment of Prometheus and Grafana is complete, we can configure Grafana to collect the metrics from Prometheus and show them in dashboards.

### Data Source

To connect Grafana to Prometheus we need to create a new data source which connects to Prometheus, where the metrics come from. From the Grafana UI, go to Data Source and add new data source of type Prometheus and provide the Prometheus url.

### Dashboards

Go to Dashboard tab from the menu and Import from JSON. 
You can import the [dashboards](dashboards) one by one to monitor the different Confluent Platform components.

---
There is currently a bug on JMX Metrics for ZooKeeper where zk-0 instance doesn't appear in the metrics results and the metrics only show 2 ZooKeeper instances available. Once way to fix this is to update the ZooKeeper deployment and start instance count at 1 rather than 0.

--- 

