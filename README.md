# Monitoring & Logging Solutions Summary for Cloud & On-Premise
## This article summarizes some of the monitoring & Logging tools options that work for AWS, On-Premise, and how to use them.
![Monitoring](/monitoring.png)
## Overview
Monitoring systems for Metrics and logs follow a common concept with multiple stages to collect the data - of either - from the destination - where data is stored - through a realtime data collector, to the visualization of this data so it can be read by humans.

The steps to configure each stage depends on the tool used from whether third-party tools - that can make a solution from a specific tool used for each stage, so they can integrate together - to a single platfrom that works out of the box as an ultimate soluti,on that manages these stages on its own in the backend.

The following options demonstrate in easy language the stages requierd for a monitoring system for logs and metrics, for Cloud - mainly AWS - & On-Premise infrastructures.

## Self-Hosted third-party examples of each stage
### Stage A: Data aggregation
This is where the collected data is stored - from both Metrics and logs. 
#### Logs
1. Loki
2. ElasticSearch
3. OpenSearch
#### Metrics
1. Prometheus
2. ElasticSearch
3. OpenSearch

### Stage B: Agent: Data Collector / Exporter
An agent is an app that is installed inside the environment from which we want to collect the logs and metrics, such as a linux/windows server, a Kuberenetes node, or a Cloud managed service. An agent will collect this data and send it to the Data aggregator (stage A).
#### Logs
1. Logstash - part of E(ElasticSearch)L(Logstash)K(Kibana - *explained below*) stack.
2. Promtail - used with Loki
3. Promtail-lambda - used with Loki for AWS integration
4. Data Prepper - part of Opensearch stack.
5. Fluentbit - used with lightweight infrastructures and containerized solutions like kubernetes.
6. Fluentd
#### Metrics
1. Logstash - part of E(ElasticSearch)L(Logstash)K(Kibana - *explained below*) stack.
2. Prometheus exporters - agents like addons used with Prometheus to collect metrics from different sources.
3. Data Prepper - part of Opensearch
4. Fluentbit - Not focused on metrics though
5. Fluentd - Not focused on metrics though

### Stage C: Visualization
This is where the collected data - from Stage A - can be interpretted in human friendly way.
1. Grafana - Universal; can integrate with tens of Data aggregation tools.
2. Kibana - Part of EKS
3. OpenSearch Dashboards - Part of Opensearch


## Third-party bundled solution examples
Here are a bundled tools or single managed tool that are used for complete monitoring solutions:
1. **E**(*elasticsearch*)**L**(*Logstash*)**K**(*Kibana*)
2. **Opensearch** - using its storage/collection/visualization tools.
3. (**Prometheus** - *used with agent: prometheues exporters*) + (**Loki** - *used with agent: promtail*) + **Grafana** for visualization.
4. **Datadog** - Universal managed service
5. **Splunk** - Universal managed service
6. **Sumo Logic** - Universal managed service
7. **New Relic** - Universal managed service

## Easy / Cost Effective / Lightwaight monitoring solutions for AWS and On Premise
### AWS cloud
#### Logging
 1. **CloudWatch** *(including cloudwatch agent for EC2 instances)*- fully managed AWS monitoring system.
 2. **Grafana Cloud** - Managed by Grafana; using Cloudwatch addon.
 3. **Grafana self-hosted** - based on Cloudwatch addon.
#### Metrics
 1. **CloudWatch** *(including cloudwatch agent for EC2 instances)*- fully managed AWS monitoring system.
 2. **Grafana Cloud** - Managed by Grafana; using on promethues for monitoring
 3. **Grafana Cloud** - Managed by Grafana; using Cloudwatch addon.
 4. **Grafana self-hosted** - using Cloudwatch addon.

### On Premise
#### Logging
 1. **Loki** (*Logging data storage*) + **Promtail/fluentbit** (*Logging collector tool/agent*) + **Grafana** (*Visualizing tool*)
#### Metrics
 1. **Prometheus** (*Logging data storage*) + **Prometheus exporters** (*Logging collector tool/agent*) + **Grafana** (*Visualizing tool*)

More tools can be viewed in the [CNCF landscape](https://landscape.cncf.io/guide#observability-and-analysis--observability)

# Lab: Setting up (Prometheus + Loki + Grafana)
We'll prepare a template that runs (Prometheus + Loki + Grafana) bundle - Using docker-compose.yml
## A. Metrics
   * [Doc: Prometheus installation](https://prometheus.io/docs/prometheus/latest/getting_started/)
   * [Doc: Prometheus Exporters list](https://prometheus.io/docs/instrumenting/exporters/)
### Concept
Prometheus is used as a centerlized database that stores the metrics of different applications from different machiens. 

To scrape the metrics from each machine of a certain app, we will setup an agent inside the machine that the Prometheus server will use to scrape the machine's data using the agent's endpoint.

The agent is refered to as an **exporter**, each app uses its own exporter, and each exporter has a method of installation.

One installation is done, there will be and endpoint with a port that will be used in the prometheus.yml` configuration file as a target.

### Steps:
Here's an example:

 1. Prepare the `prometheus.yml` configuration
 2. Create persistent volume for prometheues on the local machine.
 3. Define Prometheus image, ports, config file - where exporters are configured in `prometheues.yml`, and connect the persistent prometheus volume with the container's local volume.
 4. Define the exporter required as a seperate image with its configuration.
 5. Configure the `prometheues.yml` file to contain the exporter's container name - it should be defined in the docker-compose.yml to use it as endpoint under the same network - and endpoint.
 6. run the docker-compose.yml


`docker-compose.yml`
```
version: '3'

# Define persistent data volumes for each container
volumes:
  prometheus-data:
    driver: local
    
services:
# Image - Prometheus
  Prometheus:
    image: ubuntu/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    command: "--config.file=/etc/prometheus/prometheus.yml"
    volumes:
      - ./data/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    restart: unless-stopped

#Exporter images: Each exporter has its documentation of setup and configuration, including the docker container's setup
  node_exporter: #Node exporter
    image: quay.io/prometheus/node-exporter:latest
    container_name: node_exporter
    command:
      - '--path.rootfs=/host'
    network_mode: host
    pid: host
    restart: unless-stopped
    volumes:
      - '/:/host:ro,rslave'

```


`./data/prometheus.yml` exporter template:
```
scrape_configs:
    ...

    # Exporter
  - job_name: "NAME" 
    scrape_interval: 5s
    static_configs:
      - targets: ["CONTAINER_NAME_DEFINED_IN_DOCKER-COMPOSE.YML:CONTAINER_HOST_PORT"]


```

Once the docker-compose.yml is up, the prometheus image will be running, we can make sure that the exporter is running by checking the **Targets** tab in prometheus's dashboard, or by running localhost:port/metrics

## B. Logs
### Concept
Similarly to Metrics, Logs with Loki use a similar method of data scrapping.

Loki is considered as a centerlized database, that will store the logs data collected from each machine using an agent. This agent - promtail - will be installed in each machine so it collects the logs and push them to the Loki's server.

Loki uses a configuration file called loki-config.yml which sets how Loki should behave. Similarly, Promtail uses a config file called promtail-config.yml which is similar to the prometheus.yml file where we define the job's name and target and path, it also assigns the client's endpoint - Loki - where the scrapped data is pushed.

In brief:
Promtail - with the promtail-config.yml - will acts as the agent inside the machine which will control the logs data scrapping and pushing.

Loki - with the configuration file - will be running on as centerlized server that will recieve the scrapped data from all the promtail agents installed in different machines.

### Steps:
This example consider's scrapping logs from the same machine where Loki server is running, so we'll run both the agent and the database storage in the same network in docker-compose.yml file.
1. Get the loki-config.yml, promtail-config.yml, and the docker-compose.yml files from the main documentation by [Grafana Loki](https://grafana.com/docs/loki/latest/get-started/).
2. In the docker-compose.yml, connect the volume where loki-config.yml is located in loki's image to /mnt/config path in the container where configuration is done, and set the config.file's path to the loki-config.yml file.
3. In the promtail's image, we will connect the host's /var/log volume to the container's /var/log with read-only mode - to avoid overwriting to the host machine. We'll also connect the volume where promtail-config.yaml is located to the /mnt/config directroy in the container.


`docker-compose.yml`
```
version: '3'

networks:
  loki:

services:
# Loki 
  loki:
    image: grafana/loki:2.9.4
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - ./loki:/mnt/config 
    command: -config.file=/mnt/config/loki-config.yaml
    networks:
      - loki

# Protmail
  promtail:
    image: grafana/promtail:2.9.4
    container_name: promtail # required so containers can connect with each other in the network using the name
    volumes:
      - /var/log:/var/log:ro
      - ./promtail:/mnt/config
    command: -config.file=/mnt/config/promtail-config.yaml
    networks:
      - loki

```

`./promtail/promtail-config.yml`  template:
```
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: system
    static_configs:
      # Here we can create targets for different log sources
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*log

```
Once the docker-compose.yml is up, the loki image will be running, we can make sure that the exporter is running by running localhost:port/metrics

## C. Visualization

Once Logging and Metrics data are scrapped we need a user-friendly tool that help us get use this data in a productive and readable way. This is where visualizing tools come.

We'll use Grafana for this, as it is a popular self-hosted and managed universal tool used for visualization from many different sources, mainly, prometheus and Loki.

### Concept
Grafana is a tool that depends on data sources, these data sources could be any logging/metrics database. Once connected to the data source, we can create dashboard templates based on these logs and metrics, as well as alerts.

### Steps:
1. For persistant data volumes, we'll create local voluems for Grafana.
2. In case of connecting Grafana to AWS cloudwatch, we need to use the AWS credentials, So we'll connect the .aws/credentials host file to /usr/share/grafana/.aws/credentials:ro in grafana's container's directory in read-only.
3. Once created, Open the Grafana's dashboard with the default credentials - admin/admin - and update the new credentials.
4. From the data connections tab, choose the data source required; Prometheus/Loki/CloudWatch, and modify them with the endpoints and credentials if any.

`docker-compose.yml`
```
image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana-data-lib:/var/lib/grafana
      - grafana-data-etc:/etc/grafana
      # For AWS credintials
      - ~/.aws/credentials:/usr/share/grafana/.aws/credentials:ro
    networks:
      - loki
```

Finally, We'll compile the previous docker scripts in a single [docker-compose.yml](/docker-compose.yml) script to run over a single machine.
