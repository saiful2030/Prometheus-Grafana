# Prometheus + Grafana

## Arsitektur:
- Mini PC → Prometheus (mengumpulkan metric dari server).
- Grafana → menampilkan dashboard cantik (100% lokal).
- Node Exporter di server Proxmox → kirim metric ke Prometheus.

## Kelebihan:
- 100% lokal (tidak perlu internet).
- Banyak dashboard siap pakai untuk Proxmox (tinggal import di Grafana).
- Bisa custom alert rules (email, Telegram, dsb).
- Cocok untuk skala besar, bisa tambah server/container di masa depan.

## Kekurangan:
- Tidak sedetail Prometheus untuk metric container/VM.
- Lebih cocok untuk network monitoring (switch, router, server uptime).



### Setup Lengkap Prometheus + Grafana (Mini PC)

#### Install Prometheus
Download Prometheus versi 64-bit
```sh
cd /tmp
```
```sh
curl -LO https://github.com/prometheus/prometheus/releases/download/v3.5.0/prometheus-3.5.0.linux-amd64.tar.gz
```
Extract
```sh
tar xvf prometheus-3.5.0.linux-amd64.tar.gz
```
```sh
cd prometheus-3.5.0.linux-amd64
```
Install binary
```sh
sudo cp prometheus /usr/local/bin/
```
```sh
sudo cp promtool /usr/local/bin/
```
Buat folder konfigurasi & data
```sh
sudo mkdir -p /etc/prometheus /var/lib/prometheus
```
Copy file konfigurasi
```sh
sudo cp prometheus.yml /etc/prometheus/prometheus.yml
```
Buat user prometheus
```sh
sudo useradd --no-create-home --shell /bin/false prometheus
```
Atur permission
```sh
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
```
```sh
sudo chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
```
Buat systemd service
```sh
sudo nano /etc/systemd/system/prometheus.service
```
Isi systemd service
```sh
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/

[Install]
WantedBy=multi-user.target
```
Aktifkan & jalankan
```sh
sudo systemctl daemon-reload
```
```sh
sudo systemctl enable prometheus
```
```sh
sudo systemctl start prometheus
```
Cek di browser
```sh
http://localhost:9090
```

#### Install Grafana
Update Repository
```sh
sudo apt-get update
```
Install Dependency
```sh
sudo apt-get install -y adduser libfontconfig1 musl
```
Download Grafana
```sh
wget https://dl.grafana.com/grafana/release/12.1.1/grafana_12.1.1_16903967602_linux_amd64.deb
```
Install Paket Grafana
```sh
sudo dpkg -i grafana_12.1.1_16903967602_linux_amd64.deb
```
Aktifkan & jalankan
```sh
sudo systemctl enable grafana-server
```
```sh
sudo systemctl start grafana-server
```
Cek di browser
```sh
http://localhost:3000
```
Login awal:
- Username: **admin**
- Password: **admin**

#### Hubungkan Grafana ke Prometheus
- Di Grafana -> **Connections** -> **Data Sources** -> **Add data source**
- Pilih **Prometheus**
- Isi URL: `http://localhost:9090`
- Klik **Save & Test**

#### Import Dashboard
- Buka **Dashboards** -> **New** -> **Import**
- Masukkan ID: `1860`
- Klik **Load**
- Pilih data source `Prometheus`
- Klik **Import**

### Setup Node Exporter di Server Proxmox

#### Install Node Exporter
`Login ke masing-masing server Proxmox`
Pindah ke Folder `/tmp`
```sh
cd /tmp
```
Unduh Node Exporter
```sh
curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
```
Ekstrak File
```sh
tar xvf node_exporter-1.8.2.linux-amd64.tar.gz
```
Masuk ke Folder Hasil Ekstraksi
```sh
cd node_exporter-1.8.2.linux-amd64
```
Salin Binary ke `/usr/local/bin/`
```sh
sudo cp node_exporter /usr/local/bin/
```
#### Buat User & Service
Buat user khusus agar lebih aman 
```sh
sudo useradd --no-create-home --shell /bin/false node_exporter
```
Buat file systemd service
```sh
sudo nano /etc/systemd/system/node_exporter.service
```
Isi
```sh
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```
Pastikan firewall Proxmox mengizinkan port 9100
```sh
sudo ufw allow 9100/tcp
```
Aktifkan & jalankan
```sh
sudo systemctl daemon-reload
```
```sh
sudo systemctl enable node_exporter
```
```sh
sudo systemctl start node_exporter
```
Tes Akses
Coba dari mini PC:
```sh
curl http://IP_SERVER_PROXMOX:9100/metrics
```
Harus keluar banyak text seperti:
```sh
# HELP node_cpu_seconds_total Seconds the CPUs spent in each mode.
node_cpu_seconds_total{cpu="0",mode="system"} 12345
...
```
#### Tambahkan ke Prometheus
Di mini PC, edit file `/etc/prometheus/prometheus.yml`
```sh
sudo nano /etc/prometheus/prometheus.yml
```
Tambahkan job seperti ini (ganti IP sesuai Proxmox kamu)
```sh
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "proxmox-nodes"
    static_configs:
      - targets: ["IP_SERVER_PROXMOX:9100", "IP_SERVER_PROXMOX:9100"]
```

Restart Prometheus
```sh
sudo systemctl restart prometheus
```
Buka browser:
```sh
http://IP_MINI_PC:9090/targets
```












