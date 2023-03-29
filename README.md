## Using Helm Charts and Kustomize to Management OpenShift Deployment

In this lab, you will create a ```Helm Chart``` for an application, and deploy it to an OpenShift cluster. Then you will customize the deployed application using ```Kustomize```

## Outcomes

You should be able to:
- Create a Helm Chart for an application.
- Deploy the application to an OpenShift cluster.
- Use Kustomize to modify the deployment configuration of the application.

## Pre-requisites and Dependencies

To perform this exercise, make sure that you have network access to the following resources from your workstation:
- A running OpenShift cluster.
- Container image repository hosted in ```quay.io/redhattraining/exoplanets:v1.0```
- CockroachDB Helm Chart from repository ```https://charts.cockroachdb.com/```


## Summary of Lab Exercise

Lab Sample Application User Story:
- The ```Exoplanets``` application is a web application that shows some information about extra solar planets. 
- This application needs a ```PostgreSQL``` compatible database, such as ```CockroachDB``` to store the information and it has a prepackaged container image in ```Quay```
- Create the Helm Chart for the Exoplanets application and deploy it to the OpenShift cluster according to the following requirements:
- Use ```exochart``` as the Helm Chart name.
- Use the application container in ```quay.io/redhattraining/exoplanets:v1.0```
- Use the ```cockroachdb``` Helm Chart with version 6.0.4 from the repository ```https://charts.cockroachdb.com/``` as the database dependency
- Use port ```8080``` for the application.
- Use the following environmental variables to configure the database connection:

![](helmdb-env-vars.png)

* If no pods are returned, monitoring for user-defined projects is disabled.  
You need to following the instructions in section ```2.3. Preparing to configure the monitoring stack```  
https://access.redhat.com/documentation/en-us/openshift_container_platform/4.10/html-single/monitoring/index#configuring-the-monitoring-stack. 
* In the above documentation, make sure ```enableUserWorkload: true``` is defined in ```cluster-monitoring-config``` ConfigMap in the ```openshift-monitoring``` namespace.  The documentation style can be confusing to some readers.

## Install Kafka CRD with Metrics

- Install the Red Hat AMQ-Streams Operator in your own namepsace (e.g. amq-streams)
- Apply the Kafka cluster definition with metrics defined
- Wait for the Kafka/Zookeeper pods to be in running status

```shell
oc apply -f yaml/kafka-metrics.yaml -n <project-name>
```

## Install PodMonitors and PrometheusRule

- Multiple PodMonitor resources are defined in: 
https://github.com/strimzi/strimzi-kafka-operator/blob/main/examples/metrics/prometheus-install/strimzi-pod-monitor.yaml
- Save a copy of the yaml to your local filesystem, you will be making changes to your local copy in next step.
- For each PodMonitor resource, edit the ```spec.namespaceSelector.matchNames``` property to match the namespace of your choice.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: cluster-operator-metrics
  labels:
    app: strimzi
spec:
  selector:
    matchLabels:
      strimzi.io/kind: cluster-operator
  namespaceSelector:
    matchNames:
      - <project-name> 
  podMetricsEndpoints:
  - path: /metrics
    port: http
...
```

- Deploy the pod monitors to the namespace where your Kafka cluster is running. 
```shell
oc apply -f yaml/strimzi-pod-monitor.yaml -n <project-name>
```
- Deploy the example Prometheus rules to the same project. 
https://github.com/strimzi/strimzi-kafka-operator/blob/main/examples/metrics/prometheus-install/prometheus-rules.yaml. 
(use the copy from GitHub, no need to make changes.)
```shell
oc apply -f prometheus-rules.yaml -n <project-name>
```
- Following screenshots show successful deployment of PodMonitors and PrometheusRule. 
```shell
oc get podmonitor -n <project-name>
```
![](podmonitor.png) 
```shell
oc get prometheusrule -n <project-name>
```
![](promorule.png)

## Grafana Credentials: ServiceAccount, ClusterRoleBinding

- Create a ```ServiceAccount``` for Grafana. Here the resource is named ```grafana-serviceaccount```. 
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: grafana-serviceaccount
  labels:
    app: strimzi
```
- Deploy the ```ServiceAccount``` to the project containing your Kafka cluster. 
```shell
oc apply -f yaml/serviceaccount.yaml -n <project-name>
```
- Create a ```ClusterRoleBinding``` resource that assigns the ```cluster-monitoring-view``` role to the Grafana ServiceAccount. 
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: grafana-cluster-monitoring-binding
  labels:
    app: strimzi
subjects:
  - kind: ServiceAccount
    name: grafana-serviceaccount
    namespace: amq-streams
roleRef:
  kind: ClusterRole
  name: cluster-monitoring-view
  apiGroup: rbac.authorization.k8s.io
```
- Deploy the ```ClusterRoleBinding``` to the project containing your Kafka cluster. 
```shell
oc apply -f yaml/clusterrolebinding.yaml -n <project-name>
```

## Deploying Grafana with a Prometheus datasource
- OCP includes a ```Thanos Querier``` instance in the ```openshift-monitoring``` project. It is used to aggregate platform metrics.  
- To consume the required platform metrics, your Grafana instance requires a ```Prometheus data source``` that can connect to ```Thanos Querier```. To configure this connection, you create a ```ConfigMap``` that authenticates, by using a token, to the ```oauth-proxy``` sidecar that runs alongside Thanos Querier. A ```datasource.yaml``` file is used as the source of the config map. 
- Finally, you deploy the Grafana application with the config map mounted as a ```volume``` to the project containing your Kafka cluster. 

### Grafana/Prometheus Deployment Steps
- Get the access token of the Grafana ServiceAccount
```shell
oc serviceaccounts get-token grafana-serviceaccount -n <project-name>
```
- Create a ```datasource.yaml``` file containing the Thanos Querier configuration for Grafana. 
Paste the access token into the ```httpHeaderValue1``` property as indicated:  
```yaml
apiVersion: 1
datasources:
- name: Prometheus
  type: prometheus
  url: https://thanos-querier.openshift-monitoring.svc.cluster.local:9091
  access: proxy
  basicAuth: false
  withCredentials: false
  isDefault: true
  jsonData:
    timeInterval: 5s
    tlsSkipVerify: true
    httpHeaderName1: "Authorization"
  secureJsonData:
    httpHeaderValue1: 'Bearer ${GRAFANA-ACCESS-TOKEN}"
  editable: true
```
- Create a config map named ```grafana-config``` from the datasource.yaml file:
```shell
oc create configmap grafana-config --from-file=datasource.yaml -n <project-name>
```
- Deploy the Grafana application to the project containing your Kafka cluster:
```shell
oc apply -f yaml/grafana.yaml -n <project-name>
```
- Verify the Grafana pod is running
- Create Openshift Route to Grafana portal using either Openshift Admin Console or command line
```shell
oc create route edge <grafana-route-name> --service=grafana --namespace=<project-name>
```
#### Troubleshooting Tips: Grafana Connection Issues to Prometheus Datasource
- Delete the existing ```Prometheus Data Source``` from Grafana->Configuration->Data Sources
- Add a new ```Prometheus Data Source```
- Fill out these fields in the data source configuration screen
  - URL: ```https://thanos-querier.openshift-monitoring.svc.cluster.local:9091```
  - Skip TLS Verify: ```true```
  - Add a Custom HTTP Header called ```Authorization```
  - Value in this format: ```Bearer <your-token>```
  - Click ```Save & Test``` button
  - If error, repeat above steps, take care to not have any spaces or typos in the bearer token string. 

![](promo-datasource.png)

## Importing the example Grafana dashboards
- An example Grafana Kafka Dashboard can be found here. 
https://github.com/strimzi/strimzi-kafka-operator/blob/main/examples/metrics/grafana-dashboards/strimzi-kafka.json
- To upload it: Grafana->Dashboard->Import->Upload JSON file. 
- If successfully installed, you should see a dashboard named ```Strimzi Kafka```

![](strimzi-kafka.png)
