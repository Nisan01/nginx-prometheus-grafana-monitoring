AWS EC2 Nginx & System Observability PipelineAn end-to-end monitoring and observability pipeline deployed on AWS EC2 to collect, scrape, and visualize real-time OS hardware metrics and HTTP request performance using Prometheus, Node Exporter, Nginx Exporter, and Grafana.Includes simulated traffic and hardware stress testing to validate live metrics under load.🏗️ Architecture Overview ┌──────────────────────────────────────────────────────────┐
 │                      AWS EC2 Server                      │
 │                                                          │
 │  ┌──────────────┐     ┌───────────────────────────────┐  │
 │  │ Linux Kernel │ ──> │ Node Exporter (:9100/metrics) │ ─┼─┐
 │  └──────────────┘     └───────────────────────────────┘  │ │
 │                                                          │ │
 │  ┌──────────────┐     ┌───────────────────────────────┐  │ │
 │  │ Nginx Web Server   │ Nginx Exporter (:9113/metrics)│ ─┼──┤ (Pull / Scrape)
 │  │ (:8080 status)     └───────────────────────────────┘  │ │  Every 15s
 │  └──────────────┘                                        │ │
 └──────────────────────────────────────────────────────────┘ │
                                                              ▼
 ┌─────────────────┐       (PromQL Queries)       ┌──────────────────────┐
 │ Grafana (:3000) │ <─────────────────────────── │ Prometheus TSDB      │
 │ (Dashboards)    │                              │ (:9090)              │
 └─────────────────┘                              └──────────────────────┘
🔑 Key FeaturesSystem-Level Observability: Real-time tracking of CPU usage across multi-core systems, RAM allocation, Disk I/O, network traffic, and Linux PSI (Pressure Stall Information).Application-Level Metrics: Monitoring active TCP connections, request processing rates (accepted vs. handled), and HTTP connection states (reading, writing, waiting).Production-Ready Systemd Integration: All exporters and databases configured as background systemd services with auto-restart policies.Synthetic Load Verification: Validated via apache2-utils (Apache Bench) for HTTP traffic spikes and stress for multi-core CPU bottleneck analysis.🛠️ Tech Stack & Port MatrixComponentRoleSecurity Group PortNginxPrimary Web Server & Status Provider80 (HTTP), 8080 (Internal Status)Node ExporterOS / Hardware Metrics Exporter9100Nginx ExporterApplication Metrics Exporter9113PrometheusTime-Series Database (Metrics Collector)9090GrafanaVisualization & Dashboard Engine3000🚀 Deployment & Installation1. Web Server & Status ConfigurationInstall Nginx and expose its internal status module locally:Bashsudo apt update && sudo apt install nginx -y
sudo systemctl enable --now nginx

# Configure status endpoint for the local exporter
sudo nano /etc/nginx/conf.d/stub_status.conf
Add configuration:Nginxserver {
    listen 127.0.0.1:8080;
    location /nginx_status {
        stub_status;
        allow 127.0.0.1;
        deny all;
    }
}
Reload Nginx: sudo nginx -t && sudo systemctl reload nginx2. Install Hardware Exporters (Systemd Managed)Node Exporter SetupBashcd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz
tar xvf node_exporter-1.10.2.linux-amd64.tar.gz
sudo mv node_exporter-1.10.2.linux-amd64/node_exporter /usr/local/bin/
Create /etc/systemd/system/node_exporter.service:Ini, TOML[Unit]
Description=Node Exporter
[Service]
User=nobody
ExecStart=/usr/local/bin/node_exporter
Restart=always
[Install]
WantedBy=multi-user.target
Nginx Exporter SetupBashcd /tmp
wget https://github.com/nginxinc/nginx-prometheus-exporter/releases/download/v1.3.0/nginx-prometheus-exporter_1.3.0_linux_amd64.tar.gz
tar xvf nginx-prometheus-exporter_1.3.0_linux_amd64.tar.gz
sudo mv nginx-prometheus-exporter /usr/local/bin/
Create /etc/systemd/system/nginx_exporter.service:Ini, TOML[Unit]
Description=Nginx Exporter
[Service]
User=nobody
ExecStart=/usr/local/bin/nginx-prometheus-exporter --nginx.scrape-uri=http://127.0.0.1:8080/nginx_status
Restart=always
[Install]
WantedBy=multi-user.target
Enable and start services:Bashsudo systemctl daemon-reload
sudo systemctl enable --now node_exporter nginx_exporter
3. Prometheus SetupBashcd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v2.53.0/prometheus-2.53.0.linux-amd64.tar.gz
tar xvf prometheus-2.53.0.linux-amd64.tar.gz
sudo mv prometheus-2.53.0.linux-amd64/prometheus /usr/local/bin/
sudo mv prometheus-2.53.0.linux-amd64/promtool /usr/local/bin/

sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo chown -R nobody:nogroup /var/lib/prometheus
Create /etc/prometheus/prometheus.yml:YAMLglobal:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'nginx'
    static_configs:
      - targets: ['localhost:9113']
Create /etc/systemd/system/prometheus.service:Ini, TOML[Unit]
Description=Prometheus
[Service]
User=nobody
ExecStart=/usr/local/bin/prometheus --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/var/lib/prometheus
Restart=always
[Install]
WantedBy=multi-user.target
Enable and start Prometheus:Bashsudo systemctl daemon-reload
sudo systemctl enable --now prometheus
4. Grafana Integration & Dashboard SetupBashsudo apt install -y apt-transport-https software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update && sudo apt install grafana -y
sudo systemctl enable --now grafana-server
Navigate to http://<EC2-PUBLIC-IP>:3000 (Default Credentials: admin / admin).Add Prometheus as Data Source: Set URL to http://localhost:9090.Import Community Dashboards:Node Exporter Full: Dashboard ID 1860Nginx Exporter: Dashboard ID 12708🧪 Load Testing & Metric VerificationTo verify that metrics reflect system utilization, run synthetic benchmarks directly on the EC2 host:HTTP Request Load SimulationGenerate high-frequency concurrent traffic using Apache Bench (ab):Bashsudo apt install apache2-utils -y

# Send continuous requests with 20 concurrent connections over 5 minutes
ab -n 1000000 -t 300 -c 20 http://localhost/
Expected Result: Processed Connections panel in Grafana shows spikes up to ~3.3k processed/handled connections per minute.CPU Hardware Stress TestingSimulate multi-core processor load using stress:Bashsudo apt install stress -y

# Burn 2 CPU cores for 60 seconds
stress --cpu 2 --timeout 60s
Expected Result: CPU Busy gauge in Node Exporter dashboard jumps to 100% load.🧠 Key Learnings & Troubleshooting NotesMetrics vs. Logs: Learned the architecture distinction between Prometheus (pull-based numeric time-series metrics for aggregate system state) and Loki/Promtail (log collection for string filtering and root-cause analysis).Dynamic EC2 IPs: Configured Grafana to communicate with Prometheus internally via http://localhost:9090 rather than external public IP addresses to withstand EC2 stop/start cycles.PromQL Filtering: Fixed $instance regex variable mismatches by standardizing target labels inside community dashboard panel queries (label_values(instance)).