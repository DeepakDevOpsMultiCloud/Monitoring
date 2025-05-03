# Prometheus + Grafana + Blackbox Exporter + Static Site Monitoring Setup

This guide outlines the step-by-step installation and configuration of Prometheus, Grafana, a static site, and the Blackbox Exporter for monitoring site availability.

## 🛠️ Prerequisites for EC2 Instance

- **Instance Type**: `t2.micro` (for testing) or `t2.medium` / `t3.medium` for better performance.
- **Operating System**: Ubuntu 20.04 or 22.04 (recommended).
- **Security Group Rules**:
  - Allow **inbound access** on:
    - TCP `9090` – for Prometheus
    - TCP `3000` – for Grafana
    - TCP `22` – for SSH
- **Storage**: At least **10 GB**.
- **User**: Use the default `ubuntu` user (Ubuntu AMI).

![](../Monitoring/Images/1.jpg)

### 🔄 Update the System

```bash
sudo apt update && sudo apt upgrade -y
 
```

## ✅ Step 1: Create an Instance and Install Prometheus

```bash
# Download Prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.46.0/prometheus-2.46.0.linux-amd64.tar.gz

# Extract the archive
tar -xvf prometheus-2.46.0.linux-amd64.tar.gz

# Move it to /opt directory
sudo mv prometheus-2.46.0.linux-amd64 /opt/prometheus

# Create Prometheus user
sudo useradd --no-create-home --shell /bin/false prometheus

# Change ownership
sudo chown -R prometheus:prometheus /opt/prometheus
```

---

## ✅ Step 2: Create Prometheus systemd Service

```bash
sudo nano /etc/systemd/system/prometheus.service
```

Paste the following:

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
ExecStart=/opt/prometheus/prometheus \
  --config.file=/opt/prometheus/prometheus.yml \
  --storage.tsdb.path=/opt/prometheus/data

[Install]
WantedBy=default.target
```

Then reload and start:

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl enable prometheus
sudo systemctl status prometheus
```

![](../Monitoring/Images/2.jpg)

Check Whether Prometheus is installed or not 
  <http://Ipaddress:9000>

![](../Monitoring/Images/3.jpg)

---

## ✅ Step 3: Install Grafana

```bash
# Install dependencies
sudo apt-get install -y adduser libfontconfig1 musl

# Download and install Grafana
wget https://dl.grafana.com/enterprise/release/grafana-enterprise_11.6.1_amd64.deb
sudo dpkg -i grafana-enterprise_11.6.1_amd64.deb

# Enable and start Grafana
sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```
![](../Monitoring/Images/4.jpg)

Now Access it from Browser
* Username & Password is "admin"
After login change Password(Custom Password)

![](../Monitoring/Images/5.jpg)
---

## ✅ Step 4: Add Prometheus as a Grafana Data Source

1. Open Grafana in your browser: `http://<your-ip>:3000`
2. Login (default: `admin/admin`)
3. Navigate to: Search **Gear Icon (⚙️) > Data Sources**
4. Click **Add data source**
5. Select **Prometheus**
6. Set URL: `http://localhost:9090`
7. Click **Save & Test**

---
![](../Monitoring/Images/6.jpg)
![](../Monitoring/Images/7.jpg)
![](../Monitoring/Images/8.jpg)
![](../Monitoring/Images/9.jpg)

## ✅ Step 5: Serve a Static Website with Nginx

```bash
# Install Nginx
sudo apt install nginx -y

# Check status
sudo systemctl status nginx

# Download a sample static site template
wget https://www.free-css.com/assets/files/free-css-templates/download/page2/prestigious.zip

# Install unzip and extract
sudo apt install zip -y
sudo unzip prestigious.zip

# Move index.html to Nginx root
cd prestigious
sudo mv index.html /var/www/html/

# Access your static site
http://<your-ip>:80
```
![](../Monitoring/Images/10.jpg)
---
- We accessed the Website full picture of website isn't possible it is application side issue not our's

To see the metrics in Prometheus the application should have /metrics folder, but for our CSS-Template we need to use some Exporters 
Common Exporters like 
- Exporter	Purpose	Default Port
- node_exporter	OS metrics (CPU, memory)	9100
- blackbox_exporter	ICMP/HTTP/TCP checks	9115
- mysqld_exporter	MySQL metrics	9104
- redis_exporter	Redis metrics	9121
- cadvisor	Docker container metrics	8080

## ✅ Step 6: Install and Configure Blackbox Exporter

```bash
# Download and extract
sudo wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.22.0/blackbox_exporter-0.22.0.linux-amd64.tar.gz
sudo tar -xvf blackbox_exporter-0.22.0.linux-amd64.tar.gz
cd blackbox_exporter-0.22.0.linux-amd64

# Run Blackbox Exporter
./blackbox_exporter &

![](../Monitoring/Images/11.jpg)

# Metrics will be available at:
http://localhost:9115/metrics
```
NOTE: Open Port 9115 in security Group
---

## ✅ Step 7: Configure Prometheus to Use Blackbox Exporter

Edit the Prometheus config:

```bash
sudo nano /opt/prometheus/prometheus.yml
```

Add the following scrape config:

```yaml
scrape_configs:
  - job_name: 'blackbox_static_site_check'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - http://localhost:80  # Replace with your site URL
                              # http://3.110.169.228:80  
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115  # Blackbox Exporter
```

Restart Prometheus to apply the changes:

```bash
sudo systemctl restart prometheus
```

---

## ✅ Step 8: Verify in Prometheus

Open Prometheus web UI:

```http
http://<your-ip>:9090/targets

```
![](../Monitoring/Images/12.jpg)
![](../Monitoring/Images/13.jpg)

You should see the target listed under `blackbox_static_site_check` and its status should be **UP**.

---

## ✅ Optional: Create Dashboard in Grafana

1. Go to **Create > Dashboard**
2. Add a panel
3. Select metric: `probe_success`
4. Apply filters and visualize uptime checks

![](../Monitoring/Images/14.jpg)


---

## 🧼 Cleanup Notes

* Ensure all services are running:

  ```bash
  sudo systemctl status prometheus
  sudo systemctl status grafana-server
  sudo systemctl status nginx
  ```

* Make sure Blackbox Exporter is always running or set it as a systemd service (optional advanced).

---

## 📁 Directory Summary

| Path                                     | Description                           |
| ---------------------------------------- | ------------------------------------- |
| `/opt/prometheus/`                       | Prometheus binary and config          |
| `/etc/systemd/system/prometheus.service` | Systemd service for Prometheus        |
| `/var/www/html/`                         | Static website content                |
| `/blackbox_exporter-*/`                  | Blackbox exporter binaries and config |

---

## ✅ Final URLs to Test

* Prometheus UI: `http://<your-ip>:9090`
* Grafana UI: `http://<your-ip>:3000`
* Static Site: `http://<your-ip>:80`
* Blackbox Exporter: `http://<your-ip>:9115/metrics`
* Prometheus Targets: `http://<your-ip>:9090/targets`

---

**Done! 🎉 monitoring setup is ready.**
