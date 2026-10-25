# Dokumentasi DevOps: Aplikasi Laporan Pelaksanaan Pekerjaan

*Disusun oleh: Abdul Ghofur, DevOps*
* Versi 1.0, 21 Oktober 2026.

## 1. Ringkasan Project

Aplikasi ini membantu karyawan membuat laporan pelaksanaan pekerjaan secara otomatis sehingga tidak perlu menyusun laporan secara manual. Dokumen ini menjelaskan technology stack, arsitektur environment, dan alur CI/CD dari sisi DevOps.

## 2. Technology Stack

| Layer | Teknologi | Versi | Fungsi di project ini | Alasan pemilihan |
| --- | --- | --- | --- | --- |
| Frontend | React | \[ISI, mis. 18.x\] | Antarmuka pengguna untuk input dan melihat laporan | Ekosistem luas, komponen reusable |
| Backend API | Go | \[ISI, mis. 1.22\] | REST API, autentikasi, logika bisnis | Performa tinggi, binary ringan, cocok untuk container |
| Service otomasi | Python | \[ISI, mis. 3.12\] | \[ISI: generate laporan PDF/Excel, scheduler, pengolahan data\] | Library pengolahan data dan dokumen lengkap |
| Database | PostgreSQL | \[ISI, mis. 16\] | Penyimpanan data pengguna, pekerjaan, dan laporan | Stabil, open source, mendukung relasi kompleks |
| Containerization | Docker, Docker Compose | \[ISI\] | Membungkus semua service agar konsisten di semua environment | Environment seragam dari dev sampai production |
| Version control | Git (\[GitHub/GitLab\]) | - | Manajemen kode dan kolaborasi | Standar industri |
| CI/CD | \[GitHub Actions / GitLab CI\] | - | Build, test, dan deploy otomatis | Terintegrasi dengan repository |
| Reverse proxy | \[Nginx / Traefik\] | \[ISI\] | Routing, SSL, serving frontend | \[ISI\] |
| Monitoring dan logging | \[ISI, mis. Prometheus + Grafana, Loki\] | \[ISI\] | Memantau kesehatan aplikasi dan log | \[ISI\] |

| Bagian | Folder | Stack | Versi | Alasan pemilihan |
| --- | --- | --- | --- | --- |
| Frontend | frontend/ | React + TypeScript (Vite SPA), Tailwind + shadcn/ui, React Query | \[ISI\] |Hasil build berupa file statis yang disajikan Nginx, sehingga tidak butuh server Node saat berjalan dan hemat resource VM. TypeScript menangkap kesalahan tipe sebelum rilis (dicek `tsc` di pipeline). React Query memudahkan pengambilan data dan pemeriksaan status job; shadcn/ui + Tailwind mempercepat pembuatan tampilan yang konsisten. 
| Auth | auth/ | Node + Better Auth: daftar, login, OTP email, lupa sandi, sesi, admin user; menerbitkan JWT ES256 | Node 24 |Fitur login, OTP email, lupa sandi, dan sesi sudah tersedia sehingga tidak dibuat dari nol, yang mengurangi risiko celah keamanan buatan sendiri. Mengikuti pola Pandora. JWT ES256 bersifat asimetris: backend cukup memegang kunci publik untuk memverifikasi, tanpa bisa menerbitkan token. 
| Backend | backend/ | Go (stdlib net/http, pgx, goose): API, upload, antrian job, verifikasi JWT dan izin dari DB | Go \[ISI\] |Go menghasilkan satu binary ringan dengan startup cepat dan memori kecil, cocok untuk VM yang juga menjalankan LibreOffice. Pustaka standar `net/http` meminimalkan dependensi. pgx adalah driver PostgreSQL yang efisien, dan goose membuat perubahan skema database terversi dan dapat dilacak. 
| Processor | processor/ | Python: pandas/openpyxl, docxtpl, LibreOffice headless (PDF), AI agent (v0.3) | Python 3.12 | Python punya ekosistem pengolahan data dan dokumen paling lengkap (pandas, openpyxl, docxtpl) dan mudah memanggil LibreOffice untuk PDF. Dipisah dari backend agar proses berat tidak mengganggu API dan resource-nya bisa dibatasi sendiri. Ekosistem AI juga kuat di Python (AI agent v0.3).
| Template LPP | lpp\_templates/ | template.docx + manifest.yaml + processor.py per template | - |Format laporan berupa file Word, sehingga bentuk laporan bisa disesuaikan tanpa mengubah kode inti. Logika tiap jenis laporan terpisah (manifest dan processor per template), jadi menambah jenis laporan baru tidak berisiko merusak yang lama. 
| Database | - | PostgreSQL (termasuk PL/pgSQL untuk function dan migrasi) | 16 |Data aplikasi (pengguna, peran, pekerjaan, laporan) bersifat relasional dan butuh transaksi yang andal. PostgreSQL juga dipakai sebagai antrian job, sehingga tidak perlu menambah komponen seperti Redis atau RabbitMQ. Open source tanpa biaya lisensi, dan PL/pgSQL tersedia untuk function dan migrasi. 
| Deploy | deploy/, docker-compose.yml | Docker Compose di VM Linux internal, Nginx sebagai reverse proxy | \[ISI\] | Aplikasi internal di satu VM, jadi Docker Compose cukup dan jauh lebih sederhana daripada Kubernetes. Docker membuat environment dev, staging, dan production seragam. Nginx menyajikan SPA, meneruskan API, dan menjadi satu-satunya pintu masuk dari luar. 
| Script bantu | \[ISI\] | Shell (Bash) | - |Shell tersedia di semua server Linux tanpa dependensi tambahan, cocok untuk tugas sederhana seperti backup terjadwal dan setup. 

**Catatan:** konfirmasi ke developer fungsi Python dan Go masing-masing, karena pembagian tugasnya mempengaruhi desain container dan pipeline.

## 3. Arsitektur Aplikasi

&#91;Sisipkan diagram dari draw.io atau Excalidraw\]

Alur umum:

1. User membuka aplikasi lewat browser dan diarahkan oleh reverse proxy.
2. Frontend (React) memanggil API Go.
3. API Go membaca dan menulis data ke PostgreSQL.
4. Untuk pembuatan laporan otomatis, API atau scheduler memanggil service Python \[ISI: lewat HTTP internal / message queue / cron\].
5. Hasil laporan disimpan di \[ISI: database / folder storage / object storage\].

### Daftar container

| Container | Image dasar | Port | Keterangan |
| --- | --- | --- | --- |
| frontend | node (build) lalu nginx | 80 | Hasil build React disajikan oleh Nginx |
| backend | golang (build) lalu alpine | \[ISI, mis. 8080\] | API Go |
| report-service | python:3.x-slim | \[ISI, mis. 8000\] | Service otomasi laporan |
| db | postgres | 5432 (internal saja) | Tidak dibuka ke publik |

## 4. Environment

| Environment | Tujuan | Server / Hosting | Domain / URL | Deploy dipicu oleh |
| --- | --- | --- | --- | --- |
| Development | Pengembangan dan uji coba developer | \[ISI: lokal / server dev\] | \[ISI\] | Push ke branch develop |
| Staging (opsional) | Uji sebelum production | \[ISI\] | \[ISI\] | Merge ke branch staging |
| Production | Dipakai pengguna sebenarnya | \[ISI\] | \[ISI\] | Merge ke main, dengan approval |

### Spesifikasi server

| Environment | OS | CPU | RAM | Storage |
| --- | --- | --- | --- | --- |
| Development | \[ISI\] | \[ISI\] | \[ISI\] | \[ISI\] |
| Production | \[ISI\] | \[ISI\] | \[ISI\] | \[ISI\] |

### Jaringan dan keamanan

- Port yang dibuka ke publik: \[ISI, biasanya 80 dan 443\].
- Port yang hanya internal: 5432 (PostgreSQL), port API dan service Python.
- SSL/TLS: \[ISI, mis. Let's Encrypt\].
- Firewall: \[ISI, mis. UFW dengan aturan apa saja\].

### Manajemen konfigurasi dan secret

- Konfigurasi dibaca dari environment variable, bukan ditulis di kode.
- File `.env` tidak boleh masuk Git. Sediakan `.env.example` berisi nama variabel tanpa nilai rahasia.
- Secret untuk CI/CD disimpan di \[GitHub Secrets / GitLab CI Variables\].
- Variabel penting: `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`, \[ISI variabel lain\].
- Pihak yang punya akses ke secret: \[ISI\].

## 5. Struktur Repository

```
project/
├── frontend/            # React
│   └── Dockerfile
├── backend/             # Go
│   └── Dockerfile
├── report-service/      # Python
│   └── Dockerfile
├── docker-compose.yml
├── .env.example
├── .github/workflows/   # pipeline CI/CD
└── docs/                # dokumentasi ini
```

&#91;Sesuaikan dengan struktur repo yang sebenarnya.\]

## 6. CI/CD

### Strategi branching

| Branch | Fungsi | Memicu |
| --- | --- | --- |
| feature/\* | Pengembangan fitur | CI (build dan test) saat pull request |
| develop | Integrasi fitur | CI dan deploy ke Development |
| main | Kode stabil | CI dan deploy ke Production (dengan approval) |

### Alur pipeline

**Push code → Lint → Test → Build Docker image → Push ke registry → Deploy ke server → Health check → Notifikasi**

### Detail tiap stage

| Stage | Frontend (React) | Backend (Go) | Report service (Python) |
| --- | --- | --- | --- |
| Lint | ESLint | golangci-lint | flake8 / ruff |
| Test | npm test | go test ./... | pytest |
| Build | npm run build | go build | docker build |
| Image | Docker image | Docker image | Docker image |

### Contoh konfigurasi pipeline (GitHub Actions)

```yaml
name: CI-CD

on:
  push:
    branches: [develop, main]
  pull_request:
    branches: [develop, main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
      - run: cd backend && go test ./...
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: cd frontend && npm ci && npm test
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: cd report-service && pip install -r requirements.txt && pytest

  build-and-deploy:
    needs: test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker compose build
      # [ISI: login ke registry, push image, lalu SSH ke server dan jalankan docker compose pull && docker compose up -d]
```

&#91;Sesuaikan versi, nama registry, dan langkah deploy dengan kondisi sebenarnya.\]

### Notifikasi

Hasil pipeline dikirim ke \[ISI: Telegram / Slack / email\].

### Jika pipeline gagal

1. Buka tab Actions dan cari stage yang gagal.
2. Baca log error pada stage tersebut.
3. Perbaiki di branch feature, lalu push ulang.
4. Jika deploy production gagal, lakukan rollback (lihat bagian 7).

## 7. Deployment Manual, Backup, dan Rollback

### Deployment manual (cadangan jika CI/CD bermasalah)

1. Login ke server lewat SSH.
2. Install Docker dan Docker Compose \[ISI perintah sesuai OS\].
3. Clone repository dan salin `.env.example` menjadi `.env`, lalu isi nilainya.
4. Jalankan `docker compose up -d --build`.
5. Cek status dengan `docker compose ps` dan `docker compose logs -f`.

### Backup database

- Metode: `pg_dump` terjadwal lewat cron.
- Frekuensi: \[ISI, mis. harian pukul 02.00\].
- Lokasi penyimpanan: \[ISI\].
- Retensi: \[ISI, mis. 7 hari terakhir\].
- Cara restore: \[ISI, mis. `psql` atau `pg_restore`\].

### Rollback

- Setiap image diberi tag versi (mis. commit SHA atau v1.0.0), bukan hanya `latest`.
- Untuk rollback, ubah tag image ke versi sebelumnya lalu jalankan `docker compose up -d`.

## 8. Monitoring dan Logging

| Yang dipantau | Tools | Alert |
| --- | --- | --- |
| Status container | \[ISI\] | \[ISI\] |
| Penggunaan CPU, RAM, disk | \[ISI\] | \[ISI, mis. di atas 80%\] |
| Log aplikasi | `docker compose logs` / \[ISI\] | \[ISI\] |
| Health check endpoint | \[ISI, mis. /health\] | \[ISI\] |

## 9. Troubleshooting Umum

| Masalah | Kemungkinan penyebab | Solusi |
| --- | --- | --- |
| Container terus restart | Environment variable salah atau database belum siap | Cek log dengan `docker compose logs [service]` |
| API tidak bisa konek ke database | Host atau kredensial salah, container db belum jalan | Periksa `.env` dan status container db |
| Frontend tampil tapi data kosong | URL API salah atau CORS | Periksa konfigurasi URL API dan reverse proxy |
| Pipeline gagal di tahap test | Test error atau dependency berubah | Jalankan test secara lokal untuk mereproduksi |

## 10. Riwayat Perubahan Dokumen

| Versi | Tanggal | Perubahan | Penulis |
| --- | --- | --- | --- |
| 1.0 | 21 Oktober 2026 | Dokumen awal | Abdul Ghofur |
