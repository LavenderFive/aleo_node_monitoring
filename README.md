# aleo-node-monitoring

A monitoring solution for Aleo node runners and validators utilizing docker containers with [Prometheus](https://prometheus.io/), [Grafana](http://grafana.org/), [cAdvisor](https://github.com/google/cadvisor), [NodeExporter](https://github.com/prometheus/node_exporter), 
and alerting with [AlertManager](https://github.com/prometheus/alertmanager). 

This is intended to be a single-stop solution for monitoring your Aleo node. It does not (yet!) calculate Aleo signing rate.

## Install

Clone this repository on your Docker host, cd into Aleo-node-monitoring directory and run compose up:

```bash
git clone https://github.com/LavenderFive/aleo-node-monitoring
cd aleo-node-monitoring

ADMIN_USER=admin ADMIN_PASSWORD=admin ADMIN_PASSWORD_HASH=JDJhJDE0JE91S1FrN0Z0VEsyWmhrQVpON1VzdHVLSDkyWHdsN0xNbEZYdnNIZm1pb2d1blg4Y09mL0ZP docker-compose up -d
```

**Caddy v2 does not accept plaintext passwords. It MUST be provided as a hash value. The above password hash corresponds to ADMIN_PASSWORD 'admin'. To know how to generate hash password, refer [Updating Caddy to v2](#Updating-Caddy-to-v2)**

Prerequisites:

* Docker Engine >= 1.13
* Docker Compose >= 1.11

Containers:

* Prometheus (metrics database) `http://<host-ip>:9090`
* Prometheus-Pushgateway (push acceptor for ephemeral and batch jobs) `http://<host-ip>:9091`
* AlertManager (alerts management) `http://<host-ip>:9093`
* Alertmanager-discord (disabled by default) `http://<host-ip>:9094`
* Grafana (visualize metrics) `http://<host-ip>:3000`
  * Infinity Plugin
* NodeExporter (host metrics collector)
* cAdvisor (containers metrics collector)
* Caddy (reverse proxy and basic auth provider for prometheus and alertmanager)
* Aleo Prometheus Exporter (Aleo monitoring solution)

## TL;DR: Steps
```
1. cp .env.sample .env
----- Caddy ------
1. under caddy/Caddyfile:
1. replace YOUR_WEBSITE.COM with your website
1. replace YOUR_EMAIL@EMAIL.COM with your email
1. point your dns to your monitoring server
-----------------
1. cd ~/aleo-node-monitoring
1. docker compose up -d
```

## Setup Grafana

### Aleo Grafana Dashboard
This monitoring solution comes built in with a *very basic* Peggo Monitoring dashboard, 
which works out of the box. Grafana, Prometheus, and Infinity are installed 
automatically.

#### 1. Create Persistent Storage
To support persistent storage, you'll first need to create the volume:
```
docker volume create grafana-storage
```

---

Navigate to `http://<host-ip>:3000` and login with user ***admin*** password ***admin***. You can change the credentials in the compose file or by supplying the `ADMIN_USER` and `ADMIN_PASSWORD` environment variables on compose up. The config file can be added directly in grafana part like this

```yaml
grafana:
  image: grafana/grafana:7.2.0
  env_file:
    - .env
```

and the config file format should have this content

```yaml
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=changeme
GF_USERS_ALLOW_SIGN_UP=false
```

If you want to change the password, you have to remove this entry, otherwise the change will not take effect

```yaml
- grafana_data:/var/lib/grafana
```

Grafana is preconfigured with dashboards and Prometheus as the default data source:

* Name: Prometheus
* Type: Prometheus
* Url: [http://prometheus:9090](http://prometheus:9090)
* Access: proxy

***Monitor Services Dashboard***

![Monitor Services](https://raw.githubusercontent.com/LavenderFive/aleo-node-monitoring/master/screens/Grafana_Prometheus.png)

The Monitor Services Dashboard shows key metrics for monitoring the containers that make up the monitoring stack:

* Prometheus container uptime, monitoring stack total memory usage, Prometheus local storage memory chunks and series
* Container CPU usage graph
* Container memory usage graph
* Prometheus chunks to persist and persistence urgency graphs
* Prometheus chunks ops and checkpoint duration graphs
* Prometheus samples ingested rate, target scrapes and scrape duration graphs
* Prometheus HTTP requests graph
* Prometheus alerts graph

## Define alerts

Two alert groups have been setup within the [alert.rules](https://github.com/LavenderFive/aleo-node-monitoring/blob/master/prometheus/alert.rules) configuration file:

* Monitoring services alerts [targets](https://github.com/LavenderFive/aleo-node-monitoring/blob/master/prometheus/alert.rules#L13-L22)
* Peggo alerts [peggo](https://github.com/LavenderFive/aleo-node-monitoring/blob/master/prometheus/alert.rules#L2-L11)

You can modify the alert rules and reload them by making a HTTP POST call to Prometheus:

```bash
curl -X POST http://admin:admin@<host-ip>:9090/-/reload
```

***Monitoring services alerts***

Trigger an alert if any of the monitoring targets (node-exporter and cAdvisor) are down for more than 30 seconds:

```yaml
- alert: monitor_service_down
    expr: up == 0
    for: 30s
    labels:
      severity: critical
    annotations:
      summary: "Monitor service non-operational"
      description: "Service {{ $labels.instance }} is down."
```

***Aleo alerts***

Trigger an alert if Peggo isn't catching up, AND it's more than 5 nonce behind the network

```yaml
- name: aleo_alerts
  rules:
  - alert: HighNonceDifference
    expr: abs(peggo_network_nonce - peggo_orchestrator_nonce) > 5 and increase(peggo_orchestrator_nonce[1h]) <= 0
    for: 15m
    labels:
      severity: critical
    annotations:
      summary: "High difference between peggo_orchestrator_nonce and peggo_network_nonce"
      description: "The difference between peggo_orchestrator_nonce and peggo_network_nonce has been greater than 5 for more than 15 minutes."
```


## Setup alerting

The AlertManager service is responsible for handling alerts sent by Prometheus server.
AlertManager can send notifications via email, Pushover, Slack, HipChat or any other system that exposes a webhook interface.
A complete list of integrations can be found [here](https://prometheus.io/docs/alerting/configuration).

You can view and silence notifications by accessing `http://<host-ip>:9093`.

The notification receivers can be configured in [alertmanager/config.yml](https://github.com/LavenderFive/aleo-node-monitoring/blob/master/alertmanager/config.yml) file.

To receive alerts via Slack you need to make a custom integration by choose ***incoming web hooks*** in your Slack team app page.
You can find more details on setting up Slack integration [here](http://www.robustperception.io/using-slack-with-the-alertmanager/).

Copy the Slack Webhook URL into the ***api_url*** field and specify a Slack ***channel***.

```yaml
route:
    receiver: 'slack'

receivers:
    - name: 'slack'
      slack_configs:
          - send_resolved: true
            text: "{{ .CommonAnnotations.description }}"
            username: 'Prometheus'
            channel: '#<channel>'
            api_url: 'https://hooks.slack.com/services/<webhook-id>'
```

![Slack Notifications](https://raw.githubusercontent.com/LavenderFive/aleo-node-monitoring/master/screens/Slack_Notifications.png)
