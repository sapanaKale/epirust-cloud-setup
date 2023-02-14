# EpiRust Cloud Setup

### Pre-requisites: 

1. Docker or any container runtime to build image
2. Kubernetes Cluster
3. Kubectl and Helm installed on your machine 


## Observability Setup

Prometheus and Grafana (monitoring and alerting)

- Create the namespace for Monitoring - `kubectl create namespace monitoring`
- Add Prometheus helm repo - `helm repo add prometheus-community https://prometheus-community.github.io/helm-charts`
- Refer to understand different configurations of prometheus
  - https://github.com/prometheus-community/helm-charts
  - https://artifacthub.io/packages/helm/prometheus-community/prometheus
- Install the helm chart for prometheus using - `helm install prom prometheus-community/prometheus -n monitoring`
- Add Grafana helm repo - `helm repo add grafana https://grafana.github.io/helm-charts`
- Refer to understand different configurations of grafana - 
  - https://github.com/grafana/helm-charts
  - https://artifacthub.io/packages/helm/grafana/grafana
- Install the grafana helm chart using - ` helm install graf grafana/grafana -n monitoring`
- Get your grafana `admin` user password by - `kubectl get secret --namespace monitoring graf-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo`
- Access grafana on `localhost:3000` using - `export POD_NAME=$(kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=graf" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace monitoring port-forward $POD_NAME 3000`
- Configure the data source as prometheus in Grafana ui.  Use url - `http://prom-prometheus-server.monitoring.svc.cluster.local:80`
- To access prometheus ui on `localhost:9090` use -  `export POD_NAME=$(kubectl get pods --namespace monitoring -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace monitoring port-forward $POD_NAME 9090`
- Look at different metrics available at prometheus ui.
- Configure the Grafana dashboards for metrics you want to monitor.
- Refer and import the few cluster level metric and kafka metrics dashboard from here - `https://github.com/sapanaKale/epirust_grafana_dashboards`


## Tracing
Jaeger 

- Create the namespace for Tracing - `kubectl create namespace tracing`
- Refer to the Jaeger-all-in-one kubernetes manifest file here - `k8s-manifests/jaeger-all-in-one`
- Deploy jaeger using - `kubectl apply -f k8s-manifests/jaeger-all-in-one -n tracing`
- To open jaeger ui on local, run - `kubectl port-forward svc/jaeger-frontend 16686:16686 -n tracing` and access it on `localhost:16686`


## Logging
Elastic stack kubernetes
- Create namespace for logging - `kubectl create namespace logging`
- Add Elastic stack helm repo - `helm repo add elastic https://Helm.elastic.co`
- Refer to understand different configurations of Elastic stack (for elasticsearch, filebeat and kibana)
  - https://github.com/elastic/helm-charts
- Install the helm chart for elasticsearch using - `helm install elasticsearch elastic/elasticsearch -n logging`
- Install the helm chart for kibana using - `helm install elasticsearch elastic/elasticsearch -n logging`
  - In case of error while installing kibana, make sure all roles, rolebindings, serviceaccounts, configmaps and secrets regarding older deployment kibana are deleted.
    - Update the filebeat image version in `elastic/helm-charts/filebeat/values.yaml` to `imagaTag: "8.4.3"` 
      - Update the filebeat config - `filebeat.inputs` section with following
        ```
        filebeat.autodiscover:
          providers:
          - type: kubernetes
            node: ${NODE_NAME}
            hints.enabled: true
            hints.default_config:
              type: container
              paths:
                - /var/log/containers/*${data.kubernetes.container.id}.log
              processors:
                - dissect:
                    tokenizer: "[%{timestamp} %{log_level} %{source_file}] %{message}"
                    field: "message"
                    target_prefix: "dissect"
          ```
- Increase the memory for filebeat daemonset  in `elastic/helm-charts/filebeat/values.yaml`, in case of OOMkilled error 
- Install the helm chart for filebeat using - `helm install filebeat elastic/filebeat -n logging -f elastic/helm-charts/filebeat/values.yaml`
- To access kibana ui on your local, Run - `kubectl port-forward deployment/kibana-kibana 5601 -n logging`
- Enter username as `elastic` and retrieve elastic user's password `kubectl get secrets --namespace=logging elasticsearch-master-credentials -ojsonpath='{.data.password}' | base64 -d`
- After logging in to kibana, create the data view using filebeat as data stream
- Go to `discover` in `Analytics` section on kibana, add the appropriate filters to check logs.


## Application Setup

### Kafka
	
- Create namespace for kafka - `kubectl create namespace kafka`
- Add cp-helm-charts to helm repo - `helm repo add confluentinc https://confluentinc.github.io/cp-helm-charts/`
- Clone cp-helm-charts gitHub repo from - `git clone https://github.com/confluentinc/cp-helm-charts`
- Open the above repository and update the `values.yaml` file to disable all other kafka utilities other than `cp-zookeeper` and `cp-kafka`. Also update the number of brokers and resources accordingly.
- Install kafka helm chart - `helm install kafka confluentinc/cp-helm-charts -f cp-helm-charts/values.yaml -n kafka`
- Update the following values in cp-kafka `values.yaml` file 
  1. offsets.topic.replication.factor = 3
  2. group.max.session.timeout.ms = 86400000
  3. min.insync.replicas = 1 
  4. default.replication.factor = 1


