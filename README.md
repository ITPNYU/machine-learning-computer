# machine-learning-computer
Setup for ITP/IMA/LowRes machine learning workstation computer in South Faculty Office.


## Energy Monitoring
The Energy monitoring setup consists of the following applications:
- [Scaphandre](https://github.com/hubblo-org/scaphandre) for collecting metrics about energy usage
- [DCGM-Exporter](https://docs.nvidia.com/datacenter/cloud-native/gpu-telemetry/latest/dcgm-exporter.html) for collecting metrics about GPU usage
- [Prometheus](https://prometheus.io/) for storing time-series data
- [Grafana](https://grafana.com/) for building visualizations based on that data

These applications are run using `docker-compose` based on the example found [here in the Scaphandre git repo](https://github.com/hubblo-org/scaphandre/tree/main/docker-compose).  Below is the docker-compose.yaml file, modified to include DCGM exporter:
```yaml
version: '3.2'

services:

  grafana:
    build:
      context: ./grafana
    environment:
      GF_SECURITY_ADMIN_PASSWORD: "graphadmin12123232"
      GF_DASHBOARDS_DEFAULT_HOME_DASHBOARD_PATH: "/var/lib/grafana/dashboards/sample/sample-dashboard.json"
    depends_on:
      - prometheus
    ports:
      - "4000:3000"
    networks:
      - scaphandre-network
    volumes:
      - grafana-data:/var/lib/grafana
      - type: bind
        source: "./dashboards/sample-dashboard.json"
        target: "/var/lib/grafana/dashboards/sample/sample-dashboard.json"

  scaphandre:
    image: hubblo/scaphandre
    privileged: true
    ports: 
      - "8080:8080"
    volumes:
      - type: bind
        source: /proc
        target: /proc
      - type: bind
        source: /sys/class/powercap
        target: /sys/class/powercap
    command: ["prometheus"]
    networks:
      - scaphandre-network

  prometheus:
    build:
      context: ./prom
    ports: 
      - "9090:9090"
    volumes: 
      - promdata-scaphandre:/prometheus 
    networks:
      - scaphandre-network

  dcgm-exporter:
    image: nvcr.io/nvidia/k8s/dcgm-exporter:3.3.9-3.6.1-ubuntu22.04
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    cap_add:
      - SYS_ADMIN
    ports:
      - "9400:9400"
    networks:
      - scaphandre-network

volumes:
  promdata-scaphandre:
  grafana-data:

networks:
  scaphandre-network:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.33.0/24
```

The `prometheus.yml` file is modified to include GPU metrics from the DCGM exporter:

```yaml


# my global config
global:
  scrape_interval:     10s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
      monitor: 'scaphandre-monitor'

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first.rules"
  # - "second.rules"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'scaphandre'
         # metrics_path defaults to '/metrics'
         # scheme defaults to 'http'.

    static_configs:
      - targets: ['scaphandre:8080']

  - job_name: 'gpu-metrics'

    static_configs:
      - targets: ['dcgm-exporter:9400']
```
