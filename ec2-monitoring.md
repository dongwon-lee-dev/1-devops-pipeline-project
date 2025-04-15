# EC2 version Monitoring

[To Home](README.md)

---

## Table of Contents

- [Configure Prometheus](#configure-prometheus)  
- [Configure Grafana](#configure-grafana)  
- [Connect Jenkins to Prometheus](#connect-jenkins-to-prometheus)  

---

## Configure Prometheus
1. Add to prometheus-2.53.2.linux-amd64/prometheus.yml

\* In case of Raspberry Pi, edit /etc/prometheus/prometheus.yml

```yaml
scrape_configs:
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]  # Look for a HTTP 200 response.
    static_configs:
      - targets:
        - http://prometheus.io    # Target to probe with http.
        - http://[Kubernetes worker node ip address]:30898
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: [Monitoring Instance ip address]:9115  # The blackbox exporter's real hostname:port.
```

2. Restart Promethius

```bash
pgrep prometheus
kill [pid]
./prometheus &
```

```bash
# Raspberry Pi
sudo systemctl restart prometheus
```

## Configure Grafana

1. Create Grafana Dashboard: admin admin
2. Home > Connections > Data sources > prometheus - Connections - Data sources - Connection Prometheus server URL: http://[ip address]:9090
   - Change whenever the monitor instance ip address changes
3. Click on the top right + Import Dashboard 7587 Load, signcl-prometheus: prometheus - Import

## Connect Jenkins to Prometheus

1.Jenkins plugin: prometheus metrics 
2. Jenkins server: install Node Exporter (9100)

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
tar -xvf node_exporter-1.8.2.linux-amd64.tar.gz
cd node_exporter-1.8.2.linux-amd64.tar.gz
./node_exporter &
```

```bash
# Raspberry pi
sudo apt install prometheus-node-exporter
sudo ufw allow 9100
```

3. Jenkins - System - Prometheus configuration default

4. Monitoring server - prometheus.yml add

```yaml
scrape_configs:
  - job_name: "node_exporter"
    static_configs:
      - targets: ["[jenkins server ip]:9100"]
  - job_name: "jenkins"
    metrics_path: "/prometheus"
    static_configs:
      - targets: ["[jenkins server ip]:8080"]
```

5. Restart Promethius

```bash
pgrep prometheus
kill [pid]
./prometheus &
```

6. Create a Grafana dashboard: Click on the top right + Import Dashboard 1860 & 9964 Load, signcl-prometheus: prometheus - Import

---

[To Home](README.md)
