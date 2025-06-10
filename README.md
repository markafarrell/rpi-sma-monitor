# rpi-sma-monitor

## Procedure

1. Install Raspberry Pi OS Lite
  * Enable SSH with:
    * Username: pi
    * Password: raspberry
  * Hostname: sma-monitor
2. SSH into sma-monitor
```bash
ssh pi@sma-monitor.lan
```
3. Upgrade
```bash
  sudo apt update && \
  sudo apt dist-upgrade && \
  sudo reboot
```
4. Install opkssh
```bash
wget -qO- "https://raw.githubusercontent.com/openpubkey/opkssh/main/scripts/install-linux.sh" | sudo bash
sudo opkssh add pi mark.andrew.farrell@gmail.com google
```

On Laptop:
```bash
opkssh login google
ssh-add
ssh pi@sma-monitor.lan
# Disable ssh login using password
cat <<EOF | sudo tee /etc/ssh/sshd_config.d/99-no-passwd.conf
PasswordAuthentication no
EOF
sudo systemctl restart sshd
sudo passwd -d pi # Disable password for pi user
```

5. Install tailscale (https://tailscale.com/kb/1174/install-debian-bookworm)
```bash
curl -fsSL https://pkgs.tailscale.com/stable/debian/bookworm.noarmor.gpg | sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null
curl -fsSL https://pkgs.tailscale.com/stable/debian/bookworm.tailscale-keyring.list | sudo tee /etc/apt/sources.list.d/tailscale.list
sudo apt-get update
sudo apt-get install tailscale
sudo tailscale up
```
NOTE: Disable key expiry in tailscale console

6. Install victoria metrics
```bash
sudo apt install victoria-metrics
sudo systemctl enable victoria-metrics
sudo systemctl start victoria-metrics

sudo mkdir -p /etc/victoria-metrics
sudo chown _victoria-metrics:_victoria-metrics /etc/victoria-metrics

sudo mkdir /lib/systemd/system/victoria-metrics.service.d

cat <<EOF | sudo tee /etc/default/victoria-metrics
# Set the command-line arguments to pass to the server.
# Due to shell escaping, to pass backslashes for regexes, you need to double
# them (\\d for \d). If running under systemd, you need to double them again
# (\\\\d to mean \d), and escape newlines too.
ARGS="-envflag.enable"
EOF

cat <<EOF | sudo tee /lib/systemd/system/victoria-metrics.service.d/storage.conf
[Service]
Environment=storageDataPath=/var/lib/victoria-metrics
EOF

cat <<EOF | sudo tee /lib/systemd/system/victoria-metrics.service.d/scrape.conf
[Service]
Environment=promscrape_config=/etc/victoria-metrics/scrape.yml
EOF

sudo mkdir -p /etc/victoria-metrics
sudo chown _victoria-metrics:_victoria-metrics /etc/victoria-metrics

cat <<EOF | sudo -u _victoria-metrics tee /etc/victoria-metrics/scrape.yml
scrape_configs:
- job_name: sma-inverter-exporter
  static_configs:
  - targets:
    - localhost:9745
- job_name: node
  static_configs:
  - targets:
    - localhost:9100
EOF

sudo systemctl daemon-reload
sudo systemctl restart victoria-metrics
```

7. Install grafana (https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/)
```bash
sudo apt-get install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com beta main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
# Updates the list of available packages
sudo apt-get update
# Installs the latest OSS release:
sudo apt-get install grafana
sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server

grafana-cli plugins install victoriametrics-metrics-datasource

cat <<EOF | sudo tee /etc/grafana/provisioning/datasources/victora-metrics.yaml
apiVersion: 1
datasources:
  - name: VictoriaMetrics
    type: victoriametrics-metrics-datasource
    access: proxy
    url: http://127.0.0.1:8428
    isDefault: true
EOF

sudo mkdir -p /var/lib/grafana/dashboards

cat <<EOF | sudo tee /etc/grafana/provisioning/dashboards/default.yml
apiVersion: 1

providers:
  - name: 'default'
    orgId: 1
    folder: ''
    folderUid: ''
    type: file
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards
EOF

rsync --rsync-path="sudo rsync" dashboards pi@sma-monitor.lan:/var/lib/grafana/dashboards


sudo systemctl restart grafana-server
```

8. Install sma_inverter_exporter
```bash
curl -L https://github.com/markafarrell/sma_inverter_exporter/releases/download/v0.4/sma_inverter_exporter-Linux-musl-arm64.tar.gz | tar xz
sudo cp sma_inverter_exporter /usr/local/bin
sudo chown root:root /usr/local/bin/sma_inverter_exporter

sudo useradd --system sma-inverter-exporter

cat <<EOF | sudo tee /lib/systemd/system/sma-inverter-exporter.service
[Unit]
Description=sma-inverter-exporter
Documentation=https://github.com/dr0ps/sma_inverter_exporter
Wants=network-online.target
After=network-online.target

[Service]
User=sma-inverter-exporter
Group=sma-inverter-exporter
Type=simple
Restart=on-failure
ExecStart=/usr/local/bin/sma_inverter_exporter

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable sma-inverter-exporter
sudo systemctl start sma-inverter-exporter
```

9. Configure wifi
```bash

sudo nmcli radio wifi on

SSID=TelstraD4A58F
PASSWORD=**********

sudo nmcli connection add type wifi con-name $SSID ifname wlan0 ssid $SSID wifi-sec.key-mgmt wpa-psk wifi-sec.auth-alg open wifi-sec.psk $PASSWORD connection.autoconnect yes

SSID=OPTUS_BC32B3
PASSWORD=**********

sudo nmcli connection add type wifi con-name $SSID ifname wlan0 ssid $SSID wifi-sec.key-mgmt wpa-psk wifi-sec.auth-alg open wifi-sec.psk $PASSWORD connection.autoconnect yes

```