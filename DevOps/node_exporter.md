** Node Exporter can be downloaded directly from its official GitHub repository.**

First, navigate to the temporary directory:
```
cd /tmp
```
Download Node Exporter:
```
wget https://github.com/prometheus/node_exporter/releases/download/v1.12.1/node_exporter-1.12.1.linux-amd64.tar.gz
```
extract
```
tar -xzf node_exporter-1.12.1.linux-amd64.tar.gz
```
config node_exporter
```
sudo install -m 0755 node_exporter-1.12.1.linux-amd64/node_exporter /usr/local/bin/node_exporter
```
Check the installed version:
```
node_exporter --version
```
### 2.1 Create a dedicated user
For security reasons, we will create a separate system user for Node Exporter instead of running it as root.
```
sudo useradd --system --no-create-home --user-group --shell /usr/sbin/nologin node_exporter
```
### 2.2 Create a systemd service
Create a new service file:
```
sudo nano /etc/systemd/system/node_exporter.service
 ```

Add the following configuration:
```
[Unit]
Description=Prometheus Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter --web.listen-address=127.0.0.1:9100
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
### 3.1 Start Node Exporter
After creating the service file, reload systemd:
```
sudo systemctl daemon-reload
 ```
Start Node Exporter:
```
sudo systemctl start node_exporter
```
### 3.2 Enable Node Exporter on boot
To make Node Exporter start automatically when the server reboots, run:
```
sudo systemctl enable node_exporter
```
### 3.3 Check service status
Check whether Node Exporter is running:
```
sudo systemctl status node_exporter
```
