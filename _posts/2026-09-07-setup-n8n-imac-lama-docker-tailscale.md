---
layout: post
title: "Setup n8n Self-Hosted di Komputer Lama (4GB RAM): Panduan dari Nol"
date: 2026-09-07
---

Kalau kamu punya komputer lama yang nganggur — misalnya iMac 2011 dengan RAM cuma 4GB seperti punya saya — dan koneksi internet yang stabil dan unlimited, kamu bisa manfaatkan untuk otomatisasi kerja sehari-hari menggunakan AI tanpa perlu langganan cloud berbayar dulu. Semua langkah di bawah sudah saya coba sendiri, lengkap dengan error-error yang saya temui dan cara memperbaikinya.

Target pembaca: sudah ngerti apa itu n8n, sudah pernah pakai terminal dasar, sudah install Docker, tapi belum pernah pakai n8n atau Docker Compose. Apa itu n8n, docker, kamu bisa search di internet atau baca di artikel-artikel saya sebelumnya.

## Yang Kamu Butuhkan

- Komputer/server dengan Linux (di tulisan ini: Linux Mint, tapi distro apa saja mirip)
- Docker sudah terinstal (**bukan** Docker Desktop — kita pakai Docker CLI murni lewat terminal)
- Koneksi internet
- Akun Tailscale gratis (untuk akses publik nanti — opsional tapi direkomendasikan)

## Kenapa Docker, Bukan Virtual Machine

Docker container berbagi kernel OS host, beda dengan VM yang menjalankan OS tamu penuh. Karena itu container jauh lebih ringan dan cepat dinyalakan — penting banget di mesin dengan RAM terbatas seperti iMac saya. Bonus lain: Docker Engine di Linux itu open source penuh (lisensi Apache-2.0), jadi tidak ada masalah lisensi meskipun dipakai untuk keperluan komersial.

## Kenapa RAM Kecil Bukan Masalah Besar

n8n sebenarnya cukup ringan kalau dikonfigurasi dengan benar. Kunci utamanya:

- **Pakai SQLite, bukan PostgreSQL** — SQLite sudah jadi default n8n dan jauh lebih hemat resource. Untuk belajar dan pemakaian personal, ini lebih dari cukup.
- **Batasi memory container** lewat Docker Compose supaya n8n tidak "makan" semua RAM sistem.

## Langkah 1 — Verifikasi Docker

Sebelum mulai, pastikan Docker benar-benar jalan:

```bash
docker --version
docker compose version
sudo docker run hello-world
```

Kalau `hello-world` berhasil menampilkan pesan sukses, Docker sudah siap. Supaya tidak perlu ketik `sudo` tiap kali menjalankan perintah Docker:

```bash
sudo usermod -aG docker $USER
```

Lalu logout/login (atau restart) supaya perubahan grup berlaku.

## Langkah 2 — Buat File `docker-compose.yml`

Buat folder khusus untuk project ini:

```bash
mkdir -p ~/n8n-local && cd ~/n8n-local
```

**Catatan penting dari pengalaman saya:** kalau kamu paste konfigurasi YAML ke editor `nano`, ada kemungkinan muncul error aneh seperti:

```
go-yaml load error in scanner at L2.C9: mapping values are not allowed in this context
```

Ini biasanya terjadi karena karakter tersembunyi atau indentasi yang berantakan waktu proses copy-paste ke `nano`. Solusi paling aman: skip `nano`, langsung tulis file lewat `heredoc` di terminal — ini menulis teks apa adanya tanpa campur tangan editor:

```bash
cat > docker-compose.yml << 'EOF'
services:
  n8n:
    image: n8nio/n8n:latest
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - GENERIC_TIMEZONE=Asia/Jakarta
      - TZ=Asia/Jakarta
      - N8N_HOST=localhost
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
    volumes:
      - n8n_data:/home/node/.n8n
    deploy:
      resources:
        limits:
          memory: 1.5g

volumes:
  n8n_data:
EOF
```

Bagian `deploy.resources.limits.memory` membatasi container maksimal pakai 1.5GB RAM — penting kalau sistem kamu terbatas seperti punya saya. Kalau nanti versi Docker Compose kamu tidak membaca setting ini, ganti dengan `mem_limit: 1.5g` sejajar dengan baris `image:`.

Validasi dulu sebelum dijalankan:

```bash
docker compose config
```

Kalau tidak ada error, lanjut.

## Langkah 3 — Jalankan Container

```bash
docker compose up -d
docker compose ps
docker compose logs -f
```

Kalau di log muncul baris:

```
Editor is now accessible via:
http://localhost:5678
```

Berarti n8n sudah jalan. Buka `http://localhost:5678` di browser, dan ikuti wizard untuk membuat akun owner (email + password) — ini menggantikan basic auth di versi-versi lama n8n.

Kamu mungkin juga melihat beberapa warning deprecation di log (misalnya soal `WEBHOOK_URL` vs `N8N_WEBHOOK_URL`, atau soal Python task runner yang tidak tersedia). Selama tidak ada kata "failed to start" untuk n8n itu sendiri dan baris "Editor is now accessible" muncul, itu semua aman diabaikan untuk pemakaian dasar.

## Langkah 4 — Akses Publik Lewat Tailscale Funnel

Kalau kamu ingin n8n bisa menerima webhook dari internet (misalnya dari WhatsApp API, Telegram, Google Forms), n8n perlu diakses lewat domain publik dengan HTTPS — bukan cuma `localhost`. Di sinilah Tailscale Funnel berguna: gratis, otomatis dapat sertifikat HTTPS, tanpa perlu setup Nginx/Certbot manual.

### 4.1 Install Tailscale di host (bukan di dalam container)

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Ikuti link login yang muncul, approve device-nya dari browser.

### 4.2 Aktifkan HTTPS Certificates

Buka [admin console Tailscale](https://login.tailscale.com/admin/dns), aktifkan toggle **HTTPS Certificates**. Tanpa ini, Funnel tidak bisa menerbitkan sertifikat otomatis.

### 4.3 Jalankan Funnel

```bash
sudo tailscale funnel --bg 5678
tailscale funnel status
```

Perintah kedua menampilkan domain publik kamu, formatnya **`nama-device.nama-tailnet.ts.net`** — misalnya `yulian-imac.banteng-fahrenheit.ts.net`. Catat domain lengkapnya (dua bagian sebelum `.ts.net`, jangan sampai kepotong).

> **Jebakan umum:** kalau kamu cuma pakai bagian belakang domainnya (nama tailnet saja, tanpa nama device), browser akan menampilkan error `DNS_PROBE_FINISHED_NXDOMAIN`. Selalu pakai domain lengkap dari hasil `tailscale funnel status`.

### 4.4 Update Konfigurasi n8n

Edit `docker-compose.yml`, ganti bagian `environment` dengan domain funnel kamu:

```yaml
    environment:
      - GENERIC_TIMEZONE=Asia/Jakarta
      - TZ=Asia/Jakarta
      - N8N_HOST=nama-device.nama-tailnet.ts.net
      - N8N_PROTOCOL=https
      - N8N_PORT=5678
      - N8N_WEBHOOK_URL=https://nama-device.nama-tailnet.ts.net.ts.net/
      - N8N_PROXY_HOPS=1
```

Dua hal wajib menurut dokumentasi resmi n8n saat berjalan di belakang reverse proxy: `N8N_WEBHOOK_URL` (bukan `WEBHOOK_URL` yang sudah deprecated) dan `N8N_PROXY_HOPS=1`.

Restart container supaya env baru terbaca:

```bash
docker compose down
docker compose up -d
```

### 4.5 Test dari Luar

Buka domain funnel-nya dari HP pakai koneksi internet lain (bukan koneksi internet yang kamu pakai saat ini), untuk memastikan benar-benar bisa diakses publik.

> **Kalau muncul "Cannot GET /":** jangan panik dulu. Dalam pengalaman saya, ini hilang sendiri setelah beberapa saat — kemungkinan sertifikat HTTPS-nya masih dalam proses terbit/propagasi.

### 4.6 Satu Hal yang Sering Terlewat: Jangan Pakai `localhost` Lagi

Setelah `N8N_PROTOCOL=https` diaktifkan, n8n otomatis mengaktifkan **secure cookie**, yang cuma bisa dikirim lewat koneksi HTTPS. Kalau kamu masih coba akses `http://localhost:5678` (plain http), kemungkinan besar akan muncul error:

> "Your n8n server is configured to use a secure cookie, however, you are visiting this via an insecure URL"

Solusinya simpel: **selalu akses lewat domain Tailscale**, walaupun kamu sedang bekerja langsung dari komputer yang sama. Karena Tailscale jalan di host, domain ini tetap bisa diakses normal dari komputer itu sendiri dengan HTTPS yang valid.

## Kenalan dengan Konsep Dasar n8n

Kalau kamu sudah familiar dengan format JSON, bagian ini akan terasa mudah — karena di balik layar, n8n cuma "mengoper" data antar node dalam bentuk JSON.

**Istilah kunci:**

- **Workflow** — alur kerja otomatis, kumpulan node yang disambung jadi satu alur.
- **Node** — satu "kotak" instruksi. Tiap kotak melakukan satu hal spesifik.
- **Trigger** — node khusus yang memicu seluruh alur untuk mulai berjalan. Ibarat bel di pintu toko: begitu bunyi, alur mulai bergerak.
- **Webhook trigger** — jenis trigger yang menunggu "pesan" datang dari aplikasi lain lewat internet (misalnya notifikasi WhatsApp).

Setiap node menghasilkan output berbentuk **array of items**, dan tiap item punya properti `json`:

```json
[
  {
    "json": {
      "pesan": "Halo dari brownies!"
    }
  }
]
```

## Latihan Pertama (Aman, Tanpa Internet)

1. Buka n8n, klik **"+ Add workflow"**.
2. Di canvas kosong, klik **"+"**, cari dan pilih **"Manual Trigger"** — trigger paling sederhana, cuma jalan kalau kamu klik tombol Execute sendiri.
3. Klik **"+"** lagi di sebelah kanan node itu, tambahkan node **"Edit Fields (Set)"** — untuk mengisi data sederhana.
4. Klik **"Execute workflow"**, lihat node menyala hijau kalau berhasil. Klik tiap node untuk lihat data JSON yang lewat di dalamnya.

## Langkah Lanjutan

Setelah nyaman dengan latihan dasar:

1. **Cek tab output tiap node** — kunci membaca n8n, selalu lihat data apa yang tersedia dari node sebelumnya.
2. **Pakai expression** — ketik `{{ $json.pesan }}` di kolom input node mana pun untuk mengambil data dari node sebelumnya.
3. **Coba HTTP Request node** dengan API publik gratis, misalnya `https://jsonplaceholder.typicode.com/todos/1` (API khusus untuk latihan, stabil) atau `https://catfact.ninja/fact`.

   > Catatan: API `api.quotable.io` yang dulu populer untuk latihan sekarang sering down/tidak stabil — hindari untuk latihan baru.

4. **Kenalan dengan IF node** — node ini mengecek kondisi dan membelah alur jadi dua jalur (true/false), dasar dari logika otomatis.
5. **Ganti Manual Trigger dengan Schedule Trigger** — supaya workflow jalan otomatis di jadwal tertentu (misalnya tiap jam 8 pagi), bukan cuma saat diklik manual.

## Penutup

Itu dia perjalanan setup n8n self-hosted saya dari nol di komputer lama — mulai dari Docker, Tailscale Funnel untuk akses publik, sampai kenalan konsep dasarnya. Kalau kamu juga sedang mencoba di hardware terbatas, semoga error-error di atas menghemat waktu debugging kamu.

Langkah saya selanjutnya: mulai bangun workflow yang benar-benar dipakai sehari-hari — entah untuk otomasi kerja toko, atau bantu proses konten. Akan saya tulis lagi kalau ada progres.
