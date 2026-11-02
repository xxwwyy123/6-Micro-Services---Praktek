# Praktikum Sistem Terdistribusi: Arsitektur Microservice

Proyek ini mengimplementasikan arsitektur microservice sederhana menggunakan **Python (Flask)**, **MySQL**, **PostgreSQL**, dan **Nginx API Gateway**, dijalankan dalam multi-container menggunakan **Docker Compose**.

---

## 🏛️ Arsitektur Sistem

```text
                    +----------------------+
   Browser/curl --> |  Nginx API Gateway   |  :8080
                    |  + Dashboard (/)     |
                    +----------+-----------+
                 /mahasiswa    |    /matkul
              +----------------+----------------+
              v                                 v
   +---------------------+   HTTP GET   +---------------------+
   | Mahasiswa Service   | -----------> | Matkul Service      |
   | Flask :5000         | /matkul/{kd} | Flask :5000         |
   +----------+----------+              +----------+----------+
              v                                    v
        MySQL (mahasiswa_db)             PostgreSQL (matkul_db)
```

### Komponen Sistem:
1. **API Gateway & Dashboard (Port 8080)**:
   - Nginx sebagai Reverse Proxy / API Gateway routing berbasis path.
   - Melayani Frontend Dashboard pada `/`.
   - Meneruskan `/mahasiswa` ke `mahasiswa-service:5000`.
   - Meneruskan `/matkul` ke `matkul-service:5000`.
2. **Mahasiswa Service (Port internal 5000, mapped 5001)**:
   - Python 3.12 + Flask + Flask-SQLAlchemy.
   - Database: MySQL 8.0 (`mahasiswa_db`).
   - Menyimpan data Mahasiswa dan relasi KRS (`nim`, `kode_mk`).
   - Komunikasi sinkron (HTTP REST) ke `matkul-service` saat validasi pengambilan KRS.
3. **Mata Kuliah Service (Port internal 5000, mapped 5002)**:
   - Python 3.12 + Flask + Flask-SQLAlchemy.
   - Database: PostgreSQL 16 (`matkul_db`).
   - Menyimpan master data Mata Kuliah dengan inisialisasi data awal default (`IF301`, `IF302`, `IF303`).

---

## 📁 Struktur Direktori

```text
.
├── docker-compose.yml        # Konfigurasi multi-container Docker
├── gateway/                  # Peserta C: Nginx Reverse Proxy & Dashboard UI
│   ├── Dockerfile
│   ├── default.conf
│   └── dashboard/
│       └── index.html
├── mahasiswa-service/        # Peserta A: Service Mahasiswa & KRS (MySQL)
│   ├── Dockerfile
│   ├── requirements.txt
│   └── app.py
├── matkul-service/           # Peserta B: Service Mata Kuliah (PostgreSQL)
│   ├── Dockerfile
│   ├── requirements.txt
│   └── app.py
├── LAPORAN_PRAKTIKUM.md      # Laporan lengkap, hasil pengujian, dan analisis
└── README.md
```

---

## 🚀 Panduan Menjalankan Sistem

### Prasyarat
- Docker & Docker Compose v2 terpasang dan berjalan (Docker Desktop).

### 1. Menjalankan Seluruh Sistem
Jalankan perintah berikut di root repositori:
```bash
docker compose up --build
```
> *Catatan*: Saat pertama kali dijalankan, container database MySQL dan PostgreSQL membutuhkan waktu sekitar 10–30 detik untuk inisialisasi database. Service Flask telah dilengkapi mekanisme *retry backoff* (30 kali percobaan) hingga database siap.

### 2. Mengakses Dashboard Web
Buka peramban (browser) dan akses:
```
http://localhost:8080
```
Dashboard menyediakan antarmuka visual untuk:
- Tambah, ubah, dan hapus Mahasiswa.
- Tambah, ubah, dan hapus Mata Kuliah.
- Pengambilan mata kuliah (KRS) dan pembatalan KRS.

### 3. Menghentikan Sistem
```bash
# Menghentikan container tetapi mempertahankan data
docker compose down

# Menghentikan dan menghapus seluruh volume database (reset data)
docker compose down -v
```

---

## 📋 Kontrak API

Semua request dan response berformat JSON.

### Mahasiswa Service (`/mahasiswa`)
| Method | Path | Deskripsi | Status Sukses |
|---|---|---|---|
| GET | `/mahasiswa` | Mendapatkan daftar seluruh mahasiswa | 200 OK |
| POST | `/mahasiswa` | Tambah mahasiswa (`nim`, `nama`, `email`, `angkatan`) | 201 Created |
| GET | `/mahasiswa/{nim}` | Detail satu mahasiswa | 200 OK |
| PUT | `/mahasiswa/{nim}` | Ubah data mahasiswa | 200 OK |
| DELETE | `/mahasiswa/{nim}` | Hapus data mahasiswa beserta KRS miliknya | 204 No Content |
| GET | `/mahasiswa/{nim}/matkul` | Daftar matkul yang diambil (termasuk detail nama & SKS) | 200 OK |
| POST | `/mahasiswa/{nim}/matkul` | Ambil matkul (`kode_mk`) | 201 Created |
| DELETE | `/mahasiswa/{nim}/matkul/{kode_mk}` | Batalkan pengambilan matkul | 204 No Content |

### Mata Kuliah Service (`/matkul`)
| Method | Path | Deskripsi | Status Sukses |
|---|---|---|---|
| GET | `/matkul` | Mendapatkan daftar seluruh mata kuliah | 200 OK |
| POST | `/matkul` | Tambah mata kuliah (`kode`, `nama`, `sks`, `semester`) | 201 Created |
| GET | `/matkul/{kode}` | Detail satu mata kuliah | 200 OK |
| PUT | `/matkul/{kode}` | Ubah data mata kuliah | 200 OK |
| DELETE | `/matkul/{kode}` | Hapus data mata kuliah | 204 No Content |
