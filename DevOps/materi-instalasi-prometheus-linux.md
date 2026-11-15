# Instalasi dan Konfigurasi Prometheus di Server Linux

Materi ini disusun dari artikel DevOpsCube berjudul [How To Install and Configure Prometheus On a Linux Server](https://devopscube.com/install-configure-prometheus-linux/) (20 Oktober 2020), diterjemahkan dan disesuaikan untuk dipakai di lab internal.

Prometheus adalah sistem monitoring open source yang ringan dan memiliki mekanisme alerting yang baik. Panduan ini memasang Prometheus dari binary resmi di satu server Linux (VM atau bare metal), menjalankannya sebagai service `systemd`, lalu membuka Web UI-nya.

> [!IMPORTANT]
> Artikel asli memakai Prometheus **v2.22.0**. Sejak **Prometheus 3.0**, folder contoh `consoles` dan `console_libraries` tidak lagi disertakan di dalam paket rilis. Karena itu langkah penyalinan kedua folder tersebut dihilangkan, termasuk flag `--web.console.templates` dan `--web.console.libraries` pada service. Contoh di bawah memakai **v3.16.0**. Bila memakai versi 2.x, langkah consoles pada artikel aslinya masih berlaku.

## Sebelum Mulai

1. Pastikan memiliki akses `sudo` ke server Linux, karena seluruh perintah memerlukan hak istimewa.
2. Server memiliki akses internet untuk mengunduh binary Prometheus.
3. Port `9090` dibuka pada firewall agar Web UI bisa diakses.

| Kebutuhan | Nilai |
|---|---|
| Port Prometheus | `9090` |
| Folder konfigurasi | `/etc/prometheus` |
| Folder data (time series) | `/var/lib/prometheus` |
| Binary | `/usr/local/bin/prometheus`, `/usr/local/bin/promtool` |
| Service systemd | `prometheus.service` |

## Ringkasan Langkah

| Langkah | Isi |
|---|---|
| [1](#1-unduh-dan-ekstrak-prometheus) | Unduh dan ekstrak Prometheus |
| [2](#2-buat-user-dan-direktori) | Buat user `prometheus` dan direktori |
| [3](#3-salin-binary-ke-usrlocalbin) | Salin binary ke `/usr/local/bin` |
| [4](#4-buat-konfigurasi-prometheusyml) | Buat `prometheus.yml` |
| [5](#5-buat-service-systemd) | Buat service `systemd` |
| [6](#6-akses-web-ui-prometheus) | Akses Web UI Prometheus |
| [7](#7-langkah-berikutnya-node-exporter) | Langkah berikutnya: node-exporter |

---

## 1. Unduh dan ekstrak Prometheus

Perbarui dulu daftar paket sistem.

RHEL/CentOS/Rocky/AlmaLinux:

```bash
sudo yum update -y
```

Ubuntu/Debian:

```bash
sudo apt update -y
```

Ambil tautan unduhan Linux binary terbaru dari halaman resmi [prometheus.io/download](https://prometheus.io/download/). Setelah itu unduh, ekstrak, dan ganti nama foldernya:

```bash
curl -LO https://github.com/prometheus/prometheus/releases/download/v3.16.0/prometheus-3.16.0.linux-amd64.tar.gz
tar -xvf prometheus-3.16.0.linux-amd64.tar.gz
mv prometheus-3.16.0.linux-amd64 prometheus-files
```

Sesuaikan nomor versi dengan yang tertera di halaman unduhan. Untuk arsitektur ARM64, pilih berkas `linux-arm64`, bukan `linux-amd64`.

Isi folder hasil ekstrak:

```bash
ls -1 prometheus-files
```

Hasil yang diharapkan: `prometheus`, `promtool`, `prometheus.yml`, `LICENSE`, `NOTICE`.

> [!NOTE]
> Pada Prometheus 2.x, hasil ekstrak juga berisi folder `consoles` dan `console_libraries`. Pada 3.x keduanya tidak ada, dan itu normal.

## 2. Buat user dan direktori

Prometheus sebaiknya tidak berjalan sebagai `root`. Buat user sistem khusus dan dua direktori yang dibutuhkan:

```bash
sudo useradd --no-create-home --shell /bin/false prometheus
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
sudo chown prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
```

| Perintah | Arti |
|---|---|
| `useradd --no-create-home` | User tanpa home directory, karena tidak perlu login |
| `--shell /bin/false` | Login shell dinonaktifkan |
| `/etc/prometheus` | Tempat konfigurasi |
| `/var/lib/prometheus` | Tempat Prometheus menyimpan data time series |

Periksa hasilnya:

```bash
id prometheus
ls -ld /etc/prometheus /var/lib/prometheus
```

## 3. Salin binary ke `/usr/local/bin`

```bash
sudo cp prometheus-files/prometheus /usr/local/bin/
sudo cp prometheus-files/promtool /usr/local/bin/
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool
```

`prometheus` adalah server-nya, sedangkan `promtool` adalah utilitas untuk memeriksa konfigurasi dan aturan alert.

Periksa instalasi:

```bash
prometheus --version
promtool --version
```

Hasil yang diharapkan: keterangan versi, build, dan branch Prometheus tercetak.

## 4. Buat konfigurasi `prometheus.yml`

Seluruh konfigurasi Prometheus disimpan di `/etc/prometheus/prometheus.yml`.

**Langkah 1:** Buat berkasnya.

```bash
sudo vi /etc/prometheus/prometheus.yml
```

**Langkah 2:** Isi dengan konfigurasi berikut.

```yaml
global:
  scrape_interval: 10s

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
```

Penjelasan:

| Bagian | Arti |
|---|---|
| `global.scrape_interval: 10s` | Interval pengambilan metrik untuk semua job |
| `job_name: 'prometheus'` | Nama job; muncul sebagai label `job` pada metrik |
| `scrape_interval: 5s` | Interval khusus job ini, menimpa nilai global |
| `targets: ['localhost:9090']` | Prometheus memantau dirinya sendiri |

**Langkah 3:** Ubah kepemilikan berkas agar bisa dibaca service.

```bash
sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

**Langkah 4 (opsional, disarankan):** Periksa konfigurasi sebelum service dijalankan.

```bash
promtool check config /etc/prometheus/prometheus.yml
```

Hasil yang diharapkan: `SUCCESS: /etc/prometheus/prometheus.yml is valid prometheus config file syntax`.

## 5. Buat service systemd

Menjalankan Prometheus lewat `systemd` membuat prosesnya hidup kembali setelah server reboot dan log-nya terpusat di journal.

**Langkah 1:** Buat berkas service.

```bash
sudo vi /etc/systemd/system/prometheus.service
```

**Langkah 2:** Isi dengan konfigurasi berikut.

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.listen-address=:9090

[Install]
WantedBy=multi-user.target
```

Arti tiap opsi:

| Opsi | Arti |
|---|---|
| `User` / `Group` | Proses berjalan sebagai `prometheus`, bukan `root` |
| `Type=simple` | Prometheus berjalan di foreground, `systemd` yang mengawasi |
| `--config.file` | Lokasi konfigurasi utama |
| `--storage.tsdb.path` | Lokasi penyimpanan data time series |
| `--web.listen-address=:9090` | Alamat dan port Web UI |
| `WantedBy=multi-user.target` | Service ikut start saat boot jika di-`enable` |

**Langkah 3:** Daftarkan service, lalu jalankan.

```bash
sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl enable prometheus
```

`daemon-reload` diperlukan agar `systemd` membaca berkas unit yang baru dibuat. `enable` membuat Prometheus otomatis start saat server menyala.

**Langkah 4:** Periksa status service.

```bash
sudo systemctl status prometheus
```

Hasil yang diharapkan: baris `Active: active (running)` beserta beberapa baris log startup, misalnya `Server is ready to receive web requests.`

Bila status `failed`, baca log-nya:

```bash
sudo journalctl -u prometheus -n 50 --no-pager
```

## 6. Akses Web UI Prometheus

Buka dari browser:

```text
http://<prometheus-ip>:9090/graph
```

Ganti `<prometheus-ip>` dengan IP atau hostname server. Bila Prometheus dijalankan di mesin yang sama dengan browser, gunakan `http://localhost:9090`.

Halaman yang perlu dikenal:

| Halaman | URL | Isi |
|---|---|---|
| Graph | `/graph` | Menjalankan query PromQL dan melihat grafik |
| Targets | `/targets` | Status tiap target yang di-scrape |
| Configuration | `/config` | Konfigurasi yang sedang dipakai |
| Rules | `/rules` | Aturan alert dan recording |
| Health | `/-/healthy` | Cek cepat, membalas `Prometheus Server is Healthy.` |

Coba jalankan query berikut pada tab **Graph**:

```promql
up
```

Hasil yang diharapkan: satu deret dengan label `job="prometheus"` dan `instance="localhost:9090"` bernilai `1`. Nilai `1` berarti target berhasil di-scrape.

Query lain untuk mengenal metrik internal Prometheus:

```promql
prometheus_http_requests_total
rate(prometheus_tsdb_head_samples_appended_total[5m])
```

Bila port tidak bisa dibuka dari mesin lain, periksa firewall.

firewalld (RHEL/CentOS):

```bash
sudo firewall-cmd --permanent --add-port=9090/tcp
sudo firewall-cmd --reload
```

ufw (Ubuntu/Debian):

```bash
sudo ufw allow 9090/tcp
sudo ufw status
```

## 7. Langkah berikutnya: node-exporter

Sampai di sini Prometheus baru memantau dirinya sendiri. Untuk memantau server lain, alamat target harus didaftarkan pada `scrape_configs` di `prometheus.yml`, lalu Prometheus me-restart atau me-reload konfigurasinya.

Metrik CPU, RAM, disk, dan network tidak disediakan Prometheus, melainkan oleh **node-exporter** yang dipasang di tiap server yang ingin dipantau. Contoh konfigurasi target setelah node-exporter terpasang di server `192.168.1.20`:

```yaml
scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['192.168.1.20:9100']
```

Terapkan perubahan konfigurasi tanpa mematikan service:

```bash
promtool check config /etc/prometheus/prometheus.yml
sudo systemctl reload prometheus
```

Reload hanya berhasil bila konfigurasi valid; bila gagal, Prometheus tetap memakai konfigurasi lama dan pesan errornya muncul di `journalctl`.

Untuk lab berbasis Docker yang sudah menyertakan node-exporter dan Grafana, lihat [Panduan Lab 03](README.md).

---

## Bila Ada Masalah

| Gejala | Penyebab umum | Solusi |
|---|---|---|
| `Active: failed (Result: exit-code)` | Konfigurasi salah atau direktori tidak bisa ditulis | `journalctl -u prometheus -n 50 --no-pager`, lalu perbaiki dan `systemctl restart prometheus` |
| `unknown long flag '--web.console.templates'` | Flag disalin dari panduan Prometheus 2.x | Hapus flag consoles dari berkas service |
| `failed to open .../consoles/...: no such file or directory` | Folder `consoles` tidak ada di rilis 3.x | Hapus `--web.console.templates` dan `--web.console.libraries` |
| `permission denied` pada `/var/lib/prometheus` | Kepemilikan direktori salah | `sudo chown -R prometheus:prometheus /var/lib/prometheus` |
| Service jalan, tetapi `localhost:9090` tidak terbuka | Bind ke alamat lain, atau port dipakai proses lain | Cek `ss -tlnp \| grep 9090`, lalu sesuaikan `--web.listen-address` |
| Halaman terbuka di server, tetapi tidak dari laptop | Firewall atau security group | Buka port `9090/tcp` pada firewall dan security group cloud |
| Target `DOWN` di `/targets` | Target mati, port salah, atau exporter belum jalan | Uji `curl http://<ip-target>:9100/metrics` dari server Prometheus |
| Data hilang setelah restart | `--storage.tsdb.path` menunjuk direktori sementara | Pastikan menunjuk `/var/lib/prometheus` dan direktori itu persisten |

## Perintah Penting

| Kebutuhan | Perintah |
|---|---|
| Status service | `sudo systemctl status prometheus` |
| Start / stop / restart | `sudo systemctl start\|stop\|restart prometheus` |
| Muat ulang konfigurasi | `sudo systemctl reload prometheus` |
| Ikuti log | `sudo journalctl -u prometheus -f` |
| Validasi konfigurasi | `promtool check config /etc/prometheus/prometheus.yml` |
| Validasi aturan alert | `promtool check rules /etc/prometheus/rules.yml` |
| Cek kesehatan HTTP | `curl -s http://localhost:9090/-/healthy` |
| Cek port terpakai | `sudo ss -tlnp \| grep 9090` |
| Cek versi | `prometheus --version` |

## Referensi

- Artikel sumber: <https://devopscube.com/install-configure-prometheus-linux/>
- Unduhan Prometheus: <https://prometheus.io/download/>
- Dokumentasi Prometheus: <https://prometheus.io/docs/introduction/overview/>
- Catatan rilis 3.0 (penghapusan contoh consoles): <https://github.com/prometheus/prometheus/blob/main/CHANGELOG.md>
- Konsol templat Prometheus: <https://prometheus.io/docs/visualization/consoles/>
