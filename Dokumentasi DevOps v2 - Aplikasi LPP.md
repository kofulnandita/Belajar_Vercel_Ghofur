# Dokumentasi DevOps: Dokumen Technology Stack Aplikasi diLPPin

Oct 22, 2026 · @Ghofur

Aplikasi LPP membuat laporan pelaksanaan pekerjaan secara otomatis dan berjalan sebagai lima komponen (frontend, auth, backend, processor, database) di satu VM Linux internal lewat Docker Compose dan Nginx. Versi 2 ini memperbarui stack, environment, dan CI/CD berdasarkan dokumentasi terbaru repository. Bagian bertanda **\[ISI\]** menunggu konfirmasi tim.

## 1. Ringkasan Project

Aplikasi ini membantu karyawan menyusun laporan tanpa pekerjaan manual. Pengguna mengunggah data, sistem memprosesnya sebagai job, lalu menghasilkan laporan DOCX atau PDF berdasarkan template. Terdapat tiga peran pengguna: Superadmin, Admin, dan Operator.

| Item | Keterangan |
| --- | --- |
| Nama project | Aplikasi LPP (situs: lpp.pandora.my.id) |
| Versi aplikasi saat ini | \[ISI, dokumentasi menyebut v0.1 sampai v0.3\] |
| Repository | GitHub \[ISI: URL\] |
| Penanggung jawab DevOps | \[nama kamu\] |
| Pembimbing / PIC | \[ISI\] |

## 2. Technology Stack

### 2.1 Komponen dan teknologi

| Bagian | Folder | Stack | Versi |
| --- | --- | --- | --- |
| Frontend | frontend/ | React + TypeScript (Vite SPA), Tailwind + shadcn/ui, React Query | \[ISI\] |
| Auth | auth/ | Node + Better Auth: daftar, login, OTP email, lupa sandi, sesi, admin user; menerbitkan JWT ES256 | Node 24 |
| Backend | backend/ | Go (stdlib net/http, pgx, goose): API, upload, antrian job, verifikasi JWT dan izin dari DB | Go \[ISI\] |
| Processor | processor/ | Python: pandas/openpyxl, docxtpl, LibreOffice headless (PDF), AI agent (v0.3) | Python 3.12 |
| Template LPP | lpp\_templates/ | template.docx + manifest.yaml + processor.py per template | - |
| Database | - | PostgreSQL (termasuk PL/pgSQL untuk function dan migrasi) | 16 |
| Deploy | deploy/, docker-compose.yml | Docker Compose di VM Linux internal, Nginx sebagai reverse proxy | \[ISI\] |
| Script bantu | \[ISI\] | Shell (Bash) | - |

### 2.2 Komposisi bahasa repository

| Bahasa | Persentase | Peran |
| --- | --- | --- |
| TypeScript | 52,7% | Frontend dan kemungkinan auth |
| Python | 23,7% | Processor |
| Go | 19,4% | Backend |
| CSS | 1,5% | Styling |
| Shell | 1,1% | Script bantu |
| PLpgSQL | 0,9% | Function atau migrasi database |
| Lainnya | 0,7% | - |

### 2.3 Peta deployment per komponen

| Komponen | Image dasar (usulan) | Port internal | Bergantung pada | Health check | Tahap CI |
| --- | --- | --- | --- | --- | --- |
| nginx | nginx | 80 dan 443 | frontend, auth, backend | \[ISI\] | - |
| frontend | node (build) lalu nginx | \[ISI\] | - | \[ISI\] | ESLint, tsc, build |
| auth | node:24 | \[ISI\] | PostgreSQL, SMTP | \[ISI\] | lint, test \[ISI\] |
| backend | golang (build) lalu alpine | \[ISI\] | PostgreSQL, auth (kunci publik JWT) | \[ISI, mis. /health\] | go test |
| processor | python:3.12-slim + LibreOffice | tidak ada | PostgreSQL (tabel jobs), volume /data | \[ISI\] | pytest |
| db | postgres:16 | 5432 (internal saja) | volume data PostgreSQL | pg\_isready | - |

Ports, health check, dan tahap CI yang bertanda \[ISI\] perlu dikonfirmasi ke developer.

### 2.4 Hal khusus yang perlu diperhatikan

- **Processor tidak berkomunikasi lewat HTTP.** Backend dan processor hanya bertukar data lewat tabel `jobs` di PostgreSQL dan volume `/data`.
- **LibreOffice** membuat image processor lebih besar, boros memori, dan butuh font yang sesuai agar PDF rapi.
- **JWT ES256** memakai pasangan kunci: kunci privat hanya di auth, kunci publik dipakai backend untuk verifikasi.
- **OTP email** membutuhkan server SMTP yang dapat dijangkau dari VM.
- **AI agent (v0.3)** mungkin butuh akses keluar dan API key. \[ISI: konfirmasi layanan eksternal atau lokal\]
- **Template LPP** perlu diputuskan: masuk ke image atau di-mount sebagai volume. \[ISI\]

## 3. Arsitektur

```
Pengguna (browser)
      |
   [ Nginx ]  -- SSL, routing
      |-- /          -> frontend (React SPA)
      |-- /auth      -> auth (Node + Better Auth)   -> PostgreSQL
      |-- /api       -> backend (Go)                -> PostgreSQL

backend --(insert job)--> PostgreSQL (tabel jobs) <--(ambil job)-- processor (Python)
backend  <---- volume /data ---->  processor   (file upload dan hasil laporan)
```

Catatan: pemetaan path Nginx di atas adalah usulan. \[ISI: sesuaikan dengan konfigurasi sebenarnya\]

Alur laporan: pengguna login lewat auth dan menerima JWT, mengunggah berkas lewat API backend, backend menyimpan berkas ke `/data` dan membuat job, processor mengambil job lalu menghasilkan DOCX atau PDF ke `/data`, dan pengguna mengunduh hasilnya lewat backend.

## 4. Environment

| Environment | Tujuan | Hosting | URL | Deploy dipicu oleh |
| --- | --- | --- | --- | --- |
| Development | Pengembangan dan uji coba | Lokal developer / \[ISI\] | \[ISI\] | Manual atau push ke develop |
| Staging (opsional) | Uji sebelum production | \[ISI\] | \[ISI\] | Merge ke staging |
| Production | Dipakai pengguna sebenarnya | VM Linux internal | \[ISI, kemungkinan lpp.pandora.my.id\] | Merge ke main dengan approval |

### 4.1 Spesifikasi VM

| Environment | OS | CPU | RAM | Disk |
| --- | --- | --- | --- | --- |
| Development | \[ISI\] | \[ISI\] | \[ISI\] | \[ISI\] |
| Production | Linux \[ISI: distro dan versi\] | \[ISI\] | \[ISI\] | \[ISI\] |

RAM dan CPU perlu mempertimbangkan beban LibreOffice di processor.

### 4.2 Jaringan dan keamanan

- Port publik: 80 dan 443 lewat Nginx. Port PostgreSQL dan service lain hanya internal.
- SSL/TLS: \[ISI, siapa yang mengurus sertifikat\].
- Akses VM: \[ISI, SSH, VPN, atau jaringan kantor\].
- Firewall: \[ISI\].
- Batas upload: `client_max_body_size` di Nginx harus selaras dengan batas di backend.

### 4.3 Konfigurasi dan secret

| Variabel / secret | Dipakai oleh | Catatan |
| --- | --- | --- |
| SUPERADMIN\_EMAILS | auth / backend | Menentukan superadmin (lihat docs/AUTH.md di repository) |
| Kunci JWT ES256 | auth (privat), backend (publik) | Kunci privat jangan dibagikan ke service lain |
| Kredensial PostgreSQL | auth, backend, processor | Berbeda per environment |
| Konfigurasi SMTP | auth | Untuk OTP email |
| API key AI agent | processor | \[ISI jika memakai layanan eksternal\] |
| URL API (VITE\_\*) | frontend | Tertanam saat build, atur per environment |

File `.env` tidak boleh masuk Git. Sediakan `.env.example` tanpa nilai rahasia. Lokasi penyimpanan secret di CI/CD: \[ISI, GitHub Secrets atau lainnya\]. Pihak yang berhak mengakses: \[ISI\].

## 5. CI/CD

### 5.1 Strategi branching dan versi

| Branch | Fungsi | Memicu |
| --- | --- | --- |
| feature/\* | Pengembangan fitur atau change request | CI saat pull request |
| develop | Integrasi | CI dan deploy ke Development |
| main | Stabil | CI dan deploy ke Production dengan approval |

Tag image mengikuti nomor versi project (misalnya v0.3.0) dan commit SHA. Catat change request (CR-xxx) di changelog.

### 5.2 Alur pipeline

**Push code → Deteksi folder yang berubah → Lint → Test → Build image → Push ke registry → Deploy ke VM → Migrasi database → Health check → Notifikasi**

Pipeline berbasis path: hanya komponen yang foldernya berubah yang di-build dan di-test.

| Komponen | Pemicu path | Lint | Test | Build |
| --- | --- | --- | --- | --- |
| frontend | frontend/\*\* | ESLint, tsc | \[ISI\] | npm run build, image |
| auth | auth/\*\* | \[ISI\] | \[ISI\] | image |
| backend | backend/\*\* | golangci-lint | go test ./... | go build, image |
| processor | processor/**, lpp\_templates/** | ruff / flake8 | pytest | image |
| deploy | deploy/\*\*, docker-compose.yml | validasi compose | - | - |

### 5.3 Contoh konfigurasi (GitHub Actions)

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [develop, main]

jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      backend: ${{ steps.filter.outputs.backend }}
      processor: ${{ steps.filter.outputs.processor }}
      frontend: ${{ steps.filter.outputs.frontend }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            backend:
              - 'backend/**'
            processor:
              - 'processor/**'
              - 'lpp_templates/**'
            frontend:
              - 'frontend/**'

  backend:
    needs: changes
    if: needs.changes.outputs.backend == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'   # [ISI: sesuaikan versi]
      - run: cd backend && go test ./...

  processor:
    needs: changes
    if: needs.changes.outputs.processor == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: cd processor && pip install -r requirements.txt && pytest

  frontend:
    needs: changes
    if: needs.changes.outputs.frontend == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '24'
      - run: cd frontend && npm ci && npm run build
```

Contoh ini hanya tahap CI. Tahap deploy ke VM internal ditambahkan setelah ada keputusan soal runner dan registry. Versi action dan versi Go perlu disesuaikan dengan kondisi sebenarnya.

### 5.4 Keputusan yang masih terbuka untuk CI/CD

| Topik | Pilihan | Status |
| --- | --- | --- |
| Runner | Runner GitHub bawaan atau self-hosted (VM internal mungkin tidak terjangkau dari luar) | \[ISI\] |
| Registry image | GHCR, Docker Hub, atau registry internal | \[ISI\] |
| Cara deploy ke VM | SSH dari runner, runner self-hosted di VM, atau tarik image dari VM | \[ISI\] |
| Approval production | Siapa yang menyetujui | \[ISI\] |
| Notifikasi | Telegram, Slack, atau email | \[ISI\] |

### 5.5 Jika pipeline gagal

1. Buka tab Actions dan temukan job yang gagal.
2. Baca log pada langkah yang gagal.
3. Reproduksi di lokal.
4. Perbaiki di branch feature, push ulang.
5. Jika deploy production gagal, lakukan rollback (bagian 6.3).

## 6. Deployment, Backup, dan Rollback

### 6.1 Deployment manual (cadangan jika CI/CD bermasalah)

1. Masuk ke VM lewat SSH dengan akun yang berwenang.
2. Pastikan Docker dan Docker Compose terpasang.
3. Ambil kode atau image versi yang akan dirilis.
4. Siapkan file `.env` dari `.env.example` dan isi nilainya.
5. Backup database sebelum migrasi.
6. Jalankan `docker compose up -d` lalu migrasi goose \[ISI: apakah otomatis saat start atau langkah terpisah\].
7. Cek `docker compose ps` dan health check.

### 6.2 Backup

| Yang dibackup | Metode | Frekuensi | Lokasi | Retensi |
| --- | --- | --- | --- | --- |
| Database PostgreSQL | pg\_dump terjadwal | \[ISI\] | \[ISI\] | \[ISI\] |
| Folder /data (upload dan hasil laporan) | Salin atau snapshot | \[ISI\] | \[ISI\] | \[ISI\] |
| Konfigurasi (.env, nginx, compose) | Salin terenkripsi | Saat berubah | \[ISI\] | \[ISI\] |

Uji restore minimal sekali dan catat hasilnya.

### 6.3 Rollback

- Setiap image diberi tag versi, bukan hanya `latest`.
- Rollback aplikasi: ubah tag image ke versi sebelumnya lalu jalankan `docker compose up -d`.
- Rollback migrasi database: gunakan migrasi goose down bila tersedia, atau restore dari backup. \[ISI: konfirmasi kebijakan\]

## 7. Monitoring dan Logging

| Yang dipantau | Cara | Alert |
| --- | --- | --- |
| Status container | docker compose ps, Uptime Kuma | Container berhenti atau restart berulang |
| CPU, RAM, disk VM | docker stats, \[ISI: Prometheus + Grafana\] | Di atas 80% |
| Memori processor | docker stats | Mendekati batas memori container |
| Antrian job | Query tabel jobs | Job pending atau running terlalu lama, jumlah job gagal naik |
| Health check API | Endpoint /health | Tidak merespons |
| Disk /data | df -h | Sisa ruang di bawah \[ISI\]% |
| Log aplikasi | docker compose logs, \[ISI: Loki\] | Lonjakan error |

Pedoman logging: tulis ke stdout, gunakan level (DEBUG, INFO, WARNING, ERROR), jangan mencatat password, token, atau data pribadi karyawan, dan atur rotasi log Docker (`max-size`, `max-file`).

## 8. Troubleshooting Umum

| Gejala | Kemungkinan penyebab | Cara cek dan solusi |
| --- | --- | --- |
| Container terus restart | Environment variable salah atau database belum siap | `docker compose logs`, cek `.env`, atur healthcheck dan `depends_on` |
| Backend tidak bisa konek database | Host atau kredensial salah | Gunakan nama service sebagai host, cek status db |
| Login gagal atau token ditolak | Kunci JWT tidak cocok antara auth dan backend, jam server tidak sinkron | Periksa kunci publik di backend, cek waktu server |
| OTP email tidak terkirim | SMTP tidak terjangkau atau kredensial salah | Cek log auth dan konektivitas ke SMTP |
| Job laporan tidak diproses | Processor mati, job macet, atau koneksi DB bermasalah | Cek log processor dan status job di tabel jobs |
| Laporan PDF gagal atau berantakan | LibreOffice tidak terpasang, RAM kurang, font hilang | Cek log processor, pasang font di Dockerfile, naikkan batas memori |
| Upload gagal | Batas ukuran Nginx atau backend, disk /data penuh | Cek `client_max_body_size` dan `df -h` |
| Halaman 404 saat di-refresh | Nginx belum mengarahkan route SPA | Tambahkan `try_files` ke index.html |
| Frontend tampil tapi data kosong | URL API salah atau CORS | Cek tab Network dan variabel VITE\_ saat build |
| Deploy sukses tapi aplikasi tidak berubah | Image lama masih dipakai | Pastikan pull dan up dijalankan, cek tag image |

Catat setiap insiden: tanggal, gejala, penyebab, perbaikan, dan pencegahan.

## 9. Pertanyaan Terbuka untuk Konsultasi

| No | Pertanyaan | Ditujukan ke | Jawaban |
| --- | --- | --- | --- |
| 1 | Berapa environment yang dibutuhkan (Dev, Staging, Production)? | Pembimbing |  |
| 2 | Siapa yang mengelola VM, patching, dan backup VM? | Tim IT |  |
| 3 | Bagaimana runner CI menjangkau VM internal (self-hosted, VPN)? | Tim IT |  |
| 4 | Registry image apa yang dipakai? | Pembimbing |  |
| 5 | Siapa yang menyetujui rilis production? | Pembimbing |  |
| 6 | Siapa menyediakan SMTP untuk OTP email? | Tim IT |  |
| 7 | Bagaimana penyimpanan dan rotasi kunci JWT? | Developer auth |  |
| 8 | Kapan migrasi goose dijalankan dan bagaimana rollback-nya? | Developer backend |  |
| 9 | Berapa batas job paralel dan memori processor? | Developer processor |  |
| 10 | AI agent memakai layanan eksternal atau lokal, dan bolehkah data keluar? | Developer processor |  |
| 11 | Template LPP masuk image atau di-mount sebagai volume? | Developer |  |
| 12 | Apakah ada standar deployment dari project Pandora yang harus diikuti? | Pembimbing |  |

## 10. Riwayat Perubahan Dokumen

| Versi | Tanggal | Perubahan | Penulis |
| --- | --- | --- | --- |
| 1.0 | \[ISI\] | Dokumen awal dengan template umum | \[nama\] |
| 2.0 | 2026-10-22 | Stack diperbarui dari dokumentasi repository (service auth, processor, VM internal), pipeline berbasis path, backup /data, pertanyaan konsultasi | \[nama\] |
