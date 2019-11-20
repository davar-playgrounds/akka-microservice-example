# Kamon
Manual for configuring Kamon/Jaeger for this Akka example

Requirements:
- Scala 2.11 or 2.12
- Java 8+

# Helm setup
- ```TILLER_NAMESPACE=kube-system```
- ```kubectl create serviceaccount --namespace $TILLER_NAMESPACE tiller```
- ```kubectl create clusterrolebinding $TILLER_NAMESPACE:tiller --clusterrole=cluster-admin --serviceaccount=$TILLER_NAMESPACE:tiller```
- ```helm init --wait --service-account tiller --tiller-namespace=$TILLER_NAMESPACE```
- verify that helm client and server version match ```helm version```

## Application Configuration
- Check the following files:
    - ```service1/build.sbt```
    - ```service1/src/main/resources/application.conf```
    - ```service2/build.sbt```
    - ```service2/src/main/resources/application.conf```
- In the file ```service1/src/main/scala/de/innfactory/svc1/Service1.scala``` add ```Kamon.init()``` at the beginning of the application
- Make the above step for service 2 ```service2/src/main/scala/de/innfactory/svc2/Service2.scala``` add ```Kamon.init()``` at the beginning of the application
- In the file ```k8s/service1.yaml``` and ```k8s/service2.yaml``` mention the annotation for prometheus
- Build and update the service
- To check if Kamon works run the following command ```kubectl port-forward YOURPODNAME 5266:5266```
- Launch your browser and navigate to ```http://localhost:5266```
    - There you should see the Kamon status page
   
## InfluxDB
- Create namespace ```kubectl create monitoring-and-tracing```
- Install InfluxDB ```helm install stable/influxdb --name influxdb --namespace monitoring-and-tracing```
- Access InfluxDB API on port 8086 via ```kubectl port-forward --namespace monitoring-and-tracing $(kubectl get pods --namespace monitoring-and-tracing -l app=influxdb -o jsonpath='{ .items[0].metadata.name }') 8086:8086```
- Check if pods are running: ```kubectl get pods -n monitoring-and-tracing```
- Connect to the CLI: ```kubectl exec -i -t --namespace monitoring-and-tracing $(kubectl get pods --namespace monitoring-and-tracing -l app=influxdb -o jsonpath='{.items[0].metadata.name}') /bin/sh```
    - Start InfluxDB client: ```influx```
    - Create database: ```CREATE DATABASE "prometheus"```

## OPTIONAL Chronograph
- Create namespace ```kubectl create monitoring-and-tracing```
- Install Chronograph ```helm install stable/chronograf --name chronograph --namespace monitoring-and-tracing```
- Access Chronograph on port 8888 via ```kubectl port-forward --namespace namespace monitoring-and-tracing $(kubectl get pods --namespace namespace monitoring-and-tracing -l app=chronograph-chronograf -o jsonpath='{ .items[0].metadata.name }') 8888```

## Prometheus setup
- Create namespace ```monitoring-and-tracing``` if it doesn't exist
- In the file ```/PATH/TO/YOUR/PROJECT/akka-microservice-sample/monitoring-and-tracing/prometheus/velues.yaml``` the remote_write, remote_read value was added for InfluxDB
- In the above file a scrape config for Kamon with the name ```kamon-prometheus``` was added
- Install prometheus ```helm install --name prometheus --namespace monitoring-and-tracing /PATH/TO/YOUR/PROJECT/akka-microservice-sample/monitoring-and-tracing/prometheus/```
- Check if pods are running: ```kubectl get pods -n monitoring-and-tracing```
- If something goes wrong you can delete prometheus with```helm delete --purge prometheus```
- Access prometheus on port 9090 via ```kubectl port-forward -n monitoring-and-tracing YOUR-PROMETHEUS-POD-NAME 9090:9090```

## Validate if Prometheus can write to InfluxDB
- Connect to the InfluxDB CLI ```kubectl exec -i -t --namespace monitoring-and-tracing $(kubectl get pods --namespace monitoring-and-tracing -l app=influxdb -o jsonpath='{.items[0].metadata.name}') /bin/sh```
    - Start InfluxDB client ```influx```
    - Use prometheus database ```use prometheus```
    - Check if there are prometheus entrys with ```SHOW MEASUREMENTS```

## Grafana setup
- Create namespace ```monitoring-and-tracing``` if it doesn't exist
- Install Grafana ```helm install --name grafana --namespace monitoring-and-tracing /PATH/TO/YOUR/PROJECT/akka-microservice-sample/monitoring-and-tracing/grafana/```
- Check if pods are running ```kubectl get pods -n monitoring-and-tracing```
- If something goes wrong you can delete prometheus with```helm delete --purge grafana```
- Access Grafana on port 3000 via ```kubectl port-forward -n monitoring-and-tracing YOUR-GRAFANA-POD-NAME 3000:3000```
- Get the admin password with ```kubectl get secret --namespace monitoring-and-tracing grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo``` 
- Add a datasource with the url ```http://prometheus-server.monitoring-and-tracing.svc.cluster.local```
    - On the Start page select ```Add datasource``` -> ```Prometheus``` -> URL ```http://prometheus-server.monitoring-and-tracing.svc.cluster.local``` -> klick ```Save & Test```
- Add a datasource with the url ```http://influxdb.monitoring-and-tracing.svc.cluster.local``` as explained above
- Import dashboards
    - Click on the ```+``` on the left navigation -> ```Import```
    - Import dashboard from ```/PATH/TO/YOUR/PROJECT/akka-microservice-sample/monitoring-and-tracing/grafana/dashboards/Kamon Akka-1573324363254.json```
    - Import dashboard from ```/PATH/TO/YOUR/PROJECT/akka-microservice-sample/monitoring-and-tracing/grafana/dashboards/InfluxDB Test-1573329837055.json```
    - The Group and Router metrics are empty because this application doesn't have one

## Elasticsearch setup
- Create namespace ```monitoring-and-tracing``` if it doesn't exist
- Install configmap ```kubectl create -f /PATH/TO/YOUR/PROJECT/akka-microservice-sample/monitoring-and-tracing/elasticsearch/configmap.yaml -n monitoring-and-tracing```
- Install Elasticsearch  ```kubectl create -f /PATH/TO/YOUR/PROJECT/akka-microservice-sample/monitoring-and-tracing/elasticsearch/elasticsearch.yaml -n monitoring-and-tracing```
- Access Elasticsearch on port 9200 via ```kubectl port-forward -n monitoring-and-tracing elasticsearch-0 9200:9200```
- Make http request to ```http://localhost:9200/_cat/indices?v```
    - You can find the default user/password in the file ```/PATH/TO/YOUR/PROJECT/akka-microservice-sample/monitoring-and-tracing/elasticsearch/configmap.yaml``` 
    - If there are some Jaeger indices, Jaeger can access the database

## Jaeger setup
- Create namespace ```monitoring-and-tracing``` if it doesn't exist
- Install Jaeger ```kubectl create -f /PATH/TO/YOUR/PROJECT/akka-microservice-sample/monitoring-and-tracing/jaeger/jaeger-production-template.yaml -n monitoring-and-tracing```
- Access Jaeger on port 16686 via ```kubectl port-forward -n monitoring-and-tracing JAEGER_QUERY_POD_NAME 16686:16686```

## Kamon metric hint
- time-buckets for metrics with a unit in the time dimension. Everything is scaled to seconds
- information-buckets for all units in the information dimension. Everything is scaled to bytes
- default-buckets are used when there is no measurement unit information in a metric
 
## Sources
- https://kamon.io/docs/latest/guides/frameworks/elementary-akka-setup/
- https://kamon.io/docs/latest/reporters/prometheus/
- https://kamon.io/docs/latest/reporters/zipkin/
- https://blog.kubernauts.io/cloud-native-monitoring-with-prometheus-and-grafana-9c8003ab9c7
- https://github.com/kamon-io/kamon-prometheus
- https://github.com/jaegertracing/jaeger-kubernetes/blob/master/README.md
- https://github.com/helm/charts/tree/master/stable/influxdb
- https://github.com/StephenKing/kamon-grafana-dashboard