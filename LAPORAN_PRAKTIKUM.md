# Laporan Praktikum Sistem Terdistribusi
## Arsitektur Microservice dengan Python, MySQL, PostgreSQL, dan Nginx API Gateway

---

## 1. Identitas dan Pendahuluan
- **Mata Kuliah**: Sistem Terdistribusi
- **Topik**: Implementasi Arsitektur Microservice, Database per Service, dan API Gateway
- **Teknologi**: Python 3.12 (Flask, SQLAlchemy), MySQL 8.0, PostgreSQL 16, Nginx Alpine, Docker Compose

---

## 2. Arsitektur Sistem dan Perancangan

### 2.1 Diagram Arsitektur
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

### 2.2 Komponen dan Prinsip Desain
1. **Pola Database-per-Service**:
   - `mahasiswa-service` mengelola database MySQL (`mahasiswa_db`) dengan tabel `mahasiswa` dan `krs`.
   - `matkul-service` mengelola database PostgreSQL (`matkul_db`) dengan tabel `mata_kuliah`.
   - Tidak ada koneksi database silang (cross-database query/join). Akses data antar-layanan murni dilakukan melalui protokol HTTP/REST API.
2. **API Gateway (Nginx)**:
   - Bertindak sebagai *single entry point* (port 8080) bagi client (browser/curl/dashboard).
   - Menerapkan *path-based routing*: me-reverse-proxy request `/mahasiswa` ke container `mahasiswa-service` dan request `/matkul` ke container `matkul-service`.
   - Menyajikan file antarmuka statis (*Dashboard Akademik* HTML/JS) pada root `/`.
3. **Komunikasi Antar-Layanan (Inter-service Communication)**:
   - Saat mahasiswa mengambil mata kuliah (`POST /mahasiswa/{nim}/matkul`), `mahasiswa-service` secara sinkron memanggil `GET http://matkul-service:5000/matkul/{kode_mk}` untuk memvalidasi keberadaan mata kuliah.
   - Saat melihat daftar KRS (`GET /mahasiswa/{nim}/matkul`), `mahasiswa-service` memanggil `matkul-service` untuk memperkaya (*enrich*) data dengan nama mata kuliah dan bobot SKS.

---

## 3. Pembagian Tugas dan Implementasi Modul

| Peran | Penanggung Jawab | Komponen & Tanggung Jawab | File Terkait |
|---|---|---|---|
| **Peserta A** | Service Mahasiswa | CRUD Mahasiswa, Pengambilan KRS, Validasi HTTP ke Service B, integrasi MySQL | `mahasiswa-service/Dockerfile`, `requirements.txt`, `app.py` |
| **Peserta B** | Service Mata Kuliah | CRUD Mata Kuliah, Seeding data awal (`IF301`, `IF302`, `IF303`), integrasi PostgreSQL | `matkul-service/Dockerfile`, `requirements.txt`, `app.py` |
| **Peserta C** | API Gateway & Orkestrasi | Konfigurasi reverse proxy Nginx, Web Dashboard SPA, orkestrasi `docker-compose.yml`, uji integrasi | `gateway/Dockerfile`, `default.conf`, `dashboard/index.html`, `docker-compose.yml` |

---

## 4. Pengujian Integrasi (Skenario 1 – 10)

Seluruh pengujian dilakukan melalui API Gateway pada URL tunggal `http://localhost:8080`.

### Skenario 1: Tambah Mahasiswa
- **Tujuan**: Memverifikasi pembuatan entitas mahasiswa baru.
- **Request**:
  ```bash
  curl -i -X POST http://localhost:8080/mahasiswa \
       -H "Content-Type: application/json" \
       -d '{"nim":"2210001","nama":"Budi","email":"budi@mail.com","angkatan":2022}'
  ```
- **Hasil yang Diharapkan**:
  - HTTP Status: `201 Created`
  - Body: `{"nim":"2210001","nama":"Budi","email":"budi@mail.com","angkatan":2022}`

### Skenario 2: NIM Duplikat
- **Tujuan**: Memastikan integritas data dan penanganan konflik NIM ganda.
- **Request**: Mengulangi perintah Skenario 1 dengan NIM yang sama.
- **Hasil yang Diharapkan**:
  - HTTP Status: `409 Conflict`
  - Body: `{"error":"NIM sudah terdaftar"}`

### Skenario 3: Tambah Mata Kuliah
- **Tujuan**: Memverifikasi pembuatan mata kuliah baru pada Service B lewat Gateway.
- **Request**:
  ```bash
  curl -i -X POST http://localhost:8080/matkul \
       -H "Content-Type: application/json" \
       -d '{"kode":"IF304","nama":"Kecerdasan Buatan","sks":3,"semester":5}'
  ```
- **Hasil yang Diharapkan**:
  - HTTP Status: `201 Created`
  - Body: `{"kode":"IF304","nama":"Kecerdasan Buatan","sks":3,"semester":5}`

### Skenario 4: Pengambilan Mata Kuliah (KRS Berhasil)
- **Tujuan**: Menguji komunikasi antar-layanan (Mahasiswa Service memverifikasi `IF301` ke Matkul Service).
- **Request**:
  ```bash
  curl -i -X POST http://localhost:8080/mahasiswa/2210001/matkul \
       -H "Content-Type: application/json" \
       -d '{"kode_mk":"IF301"}'
  ```
- **Hasil yang Diharapkan**:
  - HTTP Status: `201 Created`
  - Body: `{"nim":"2210001","kode_mk":"IF301","pesan":"berhasil"}`

### Skenario 5: Pengambilan Mata Kuliah yang Tidak Terdaftar
- **Tujuan**: Menguji validasi keberadaan mata kuliah saat kode tidak ditemukan di Matkul Service.
- **Request**:
  ```bash
  curl -i -X POST http://localhost:8080/mahasiswa/2210001/matkul \
       -H "Content-Type: application/json" \
       -d '{"kode_mk":"XX999"}'
  ```
- **Hasil yang Diharapkan**:
  - HTTP Status: `404 Not Found`
  - Body: `{"error":"mata kuliah tidak ditemukan"}`

### Skenario 6: Pengambilan Mata Kuliah Ganda (Duplikat KRS)
- **Tujuan**: Memastikan mahasiswa tidak dapat mengambil kode mata kuliah yang sama lebih dari sekali.
- **Request**: Mengulangi perintah Skenario 4 (`IF301`).
- **Hasil yang Diharapkan**:
  - HTTP Status: `409 Conflict`
  - Body: `{"error":"mata kuliah sudah diambil"}`

### Skenario 7: Melihat KRS Mahasiswa (Data Enrichment)
- **Tujuan**: Memverifikasi penggabungan data KRS dari MySQL dengan detail mata kuliah dari PostgreSQL via API call.
- **Request**:
  ```bash
  curl -i http://localhost:8080/mahasiswa/2210001/matkul
  ```
- **Hasil yang Diharapkan**:
  - HTTP Status: `200 OK`
  - Body:
    ```json
    [
      {
        "kode_mk": "IF301",
        "nama": "Sistem Terdistribusi",
        "sks": 3,
        "semester": 5
      }
    ]
    ```

### Skenario 8: Hapus Mahasiswa (Cascade KRS)
- **Tujuan**: Memastikan penghapusan entitas mahasiswa secara otomatis membersihkan relasi KRS terkait di database MySQL.
- **Request**:
  ```bash
  curl -i -X DELETE http://localhost:8080/mahasiswa/2210001
  ```
- **Hasil yang Diharapkan**:
  - HTTP Status: `204 No Content`
  - Verifikasi: Request `GET http://localhost:8080/mahasiswa/2210001` menghasilkan `404 Not Found`.

### Skenario 9: Uji Ketahanan (*Fault Tolerance*)
- **Tujuan**: Mengetahui respons sistem ketika dependent service mengalami gangguan (downtime).
- **Langkah & Perintah**:
  1. Hentikan container Matkul Service:
     ```bash
     docker compose stop matkul-service
     ```
  2. Coba daftarkan mahasiswa baru dan ambil matkul `IF301`:
     ```bash
     curl -i -X POST http://localhost:8080/mahasiswa/2210001/matkul \
          -H "Content-Type: application/json" \
          -d '{"kode_mk":"IF301"}'
     ```
- **Hasil yang Diharapkan**:
  - HTTP Status: `503 Service Unavailable`
  - Body: `{"error":"matkul-service tidak dapat dihubungi"}`
  - Layanan `mahasiswa-service` **tidak crash** berkat adanya blok `try-except requests.RequestException` dengan timeout.

### Skenario 10: Uji Persistensi Data
- **Tujuan**: Memastikan data tidak hilang saat container dimatikan dan dinyalakan kembali.
- **Langkah & Perintah**:
  ```bash
  docker compose down
  docker compose up -d
  curl -i http://localhost:8080/matkul
  ```
- **Hasil yang Diharapkan**:
  - Data mata kuliah dan mahasiswa tetap utuh karena tersimpan dalam persistent named volume Docker (`mysql_data` dan `pg_data`).

---

## 5. Jawaban Pertanyaan Analisis

### Pertanyaan 1
> **Mengapa tabel `krs` hanya menyimpan `kode_mk` tanpa foreign key ke tabel mata kuliah? Apa konsekuensinya bila mata kuliah dihapus setelah diambil mahasiswa? Usulkan solusinya.**

**Jawaban:**
1. **Alasan Tanpa Foreign Key**:
   Dalam arsitektur microservice dengan pola *Database-per-Service*, tabel `krs` berada di database MySQL (`mahasiswa_db`), sedangkan tabel `mata_kuliah` berada di database PostgreSQL (`matkul_db`). Secara teknis, DBMS relational (RDBMS) tidak mendukung pembuatan Foreign Key (FK) fisik antar-mesin/antar-database yang terpisah (*cross-database foreign key*). Selain itu, batasan arsitektur microservice melarang ketergantungan skema secara ketat (*loose coupling*) demi independensi deployment dan skalabilitas masing-masing database.
2. **Konsekuensi Jika Mata Kuliah Dihapus**:
   Jika mata kuliah dihapus di `matkul-service`, data `kode_mk` di tabel `krs` pada `mahasiswa-service` akan menjadi *orphan data* (mengacu pada mata kuliah fiktif). Saat mahasiswa melihat KRS miliknya, panggilan HTTP ke `matkul-service` akan menghasilkan status 404 atau tidak lengkap.
3. **Usulan Solusi**:
   - **Penerapan Event-Driven Architecture (Asinkron)**: Ketika mata kuliah dihapus, `matkul-service` mempublikasikan pesan event `CourseDeleted(kode_mk)` ke message broker (misal RabbitMQ/Kafka). `mahasiswa-service` sebagai subscriber menangkap event tersebut dan menandai atau menghapus data KRS terkait.
   - **Soft Delete / Archival**: Pada `matkul-service`, alih-alih menghapus data secara permanen (`DELETE`), gunakan kolom `is_active` atau `deleted_at` (*soft delete*). Dengan demikian mata kuliah nonaktif tidak dapat diambil oleh mahasiswa baru, tetapi riwayat mahasiswa yang telah mengambilnya tetap valid.
   - **Kompensasi Saga**: Menggunakan pola transaksi terdistribusi *Saga* untuk koordinasi rollback atau pembatalan antar-service.

---

### Pertanyaan 2
> **Pada skenario 9, apa yang terjadi saat Matkul Service mati? Jelaskan istilah *coupling* dan *single point of failure* (SPOF) terkait hal ini.**

**Jawaban:**
1. **Perilaku Sistem Saat Matkul Service Mati**:
   Ketika `matkul-service` mati, pemanggilan `GET http://matkul-service:5000/matkul/{kode}` oleh `mahasiswa-service` mengalami *timeout* atau *connection refused*. Karena implementasi telah dilengkapi penanganan pengecualian (`try-except requests.RequestException`), service mahasiswa tidak mengalami *crash*, melainkan merespons klien dengan status HTTP `503 Service Unavailable` beserta pesan `'matkul-service tidak dapat dihubungi'`.
2. **Coupling (Keterikatan)**:
   Kejadian ini menunjukkan adanya **Runtime Temporal Coupling** (keterikatan waktu eksekusi). Meskipun kedua service terpisah secara repositori dan database, alur pengambilan KRS membutuhkan ketersediaan `matkul-service` secara *real-time* saat itu juga. Jika satu service mati, fungsi di service lain ikut terhenti.
3. **Single Point of Failure (SPOF)**:
   Untuk proses pengambilan KRS, `matkul-service` menjadi *Single Point of Failure*: kegagalan pada satu komponen menyebabkan kegagalan sistem pada alur fitur tersebut secara keseluruhan. Jika API Gateway mati, API Gateway juga bertindak sebagai SPOF bagi seluruh akses aplikasi.

---

### Pertanyaan 3
> **Apa keuntungan dan kerugian memakai MySQL dan PostgreSQL pada dua service berbeda dibanding satu database bersama?**

**Jawaban:**
1. **Keuntungan (Polyglot Persistence & Loose Coupling)**:
   - **Pemilihan Alat yang Tepat (*Right Tool for the Job*)**: Setiap tim dapat memilih mesin database yang paling sesuai dengan kebutuhan domain layanannya (misalnya PostgreSQL untuk data berstruktur kompleks/JSON/GIS, atau database NoSQL untuk caching/log).
   - **Independensi Skema & Deployment**: Tim Mahasiswa dapat mengubah struktur tabel, migrasi database, atau melakukan upgrade versi DBMS tanpa perlu berkoordinasi atau mengambil risiko mengganggu Service Mata Kuliah.
   - **Isolasi Kegagalan & Resource**: Beban query berat pada database mahasiswa tidak akan menghabiskan CPU, memory, atau koneksi pada database mata kuliah.
2. **Kerugian**:
   - **Kompleksitas Operasional**: Tim DevOps harus memelihara, memonitor, mem-backup, dan mem-patch dua jenis DBMS berbeda.
   - **Kehilangan Integritasi Relasional Otomatis (ACID Cross-Database)**: Tidak ada transaksi atomik instan (`BEGIN ... COMMIT`) yang mencakup kedua database. Konsistensi data harus dikelola pada level aplikasi atau menggunakan pola *eventual consistency*.
   - **Overhead Infrastruktur**: Menjalankan dua instance container database yang berbeda mengonsumsi sumber daya memori dan storage lebih besar dibanding satu instance bersama.

---

### Pertanyaan 4
> **Apa peran API Gateway dalam sistem ini? Sebutkan minimal tiga fungsi lain yang dapat ditambahkan pada gateway (misalnya autentikasi, rate limiting, load balancing).**

**Jawaban:**
1. **Peran API Gateway dalam Sistem**:
   - **Pintu Masuk Tunggal (*Single Entry Point*)**: Klien hanya perlu berinteraksi dengan satu host dan port (`localhost:8080`), tanpa perlu mengetahui topologi jaringan internal, IP, atau port masing-masing microservice (`5001`, `5002`).
   - **Reverse Proxy & Path-based Routing**: Meneruskan request secara cerdas berdasarkan URI path (`/mahasiswa` -> `mahasiswa-service`, `/matkul` -> `matkul-service`).
   - **Pencegahan Masalah CORS**: Karena dashboard web statis dan API dilayani dari origin yang sama (`http://localhost:8080`), browser tidak memicu pembatasan *Cross-Origin Resource Sharing* (CORS).
   - **Enkapsulasi dan Keamanan Jaringan**: Port internal service tidak perlu diekspos ke publik internet, sehingga meminimalkan *attack surface*.
2. **Tiga Fungsi Tambahan yang Dapat Diintegrasikan**:
   - **Autentikasi & Otorisasi Terpusat**: Memeriksa token (seperti JWT atau API Key) di level gateway sebelum meneruskan request ke backend service.
   - **Rate Limiting**: Membatasi frekuensi request dari satu IP/klien (misal dengan `limit_req_zone` di Nginx) untuk mencegah *Brute Force* atau serangan DoS/DDoS.
   - **Load Balancing**: Mendistribusikan lalu lintas secara merata (round-robin / least connections) ke beberapa replika instance service jika beban sistem meningkat.

---

### Pertanyaan 5
> **Jika jumlah mahasiswa sangat besar, service mana yang perlu di-scale lebih dulu dan bagaimana caranya? Bagaimana peran Nginx dalam skenario tersebut?**

**Jawaban:**
1. **Service yang Perlu Di-scale Terlebih Dahulu**:
   `mahasiswa-service` perlu di-scale terlebih dahulu. Alasannya:
   - Volume operasi tulis (*write operations*) jauh lebih tinggi saat periode pengisian KRS (ribuan mahasiswa mengakses dan submit secara simultan).
   - Data mahasiswa dan transaksi KRS jauh lebih dinamis dan bertumbuh pesat dibanding master data mata kuliah yang cenderung statis dan jarang berubah.
2. **Cara Melakukan Skalabilitas (*Scaling Out*)**:
   - Menggunakan Docker Compose / Kubernetes untuk menjalankan beberapa replika container:
     ```bash
     docker compose up --scale mahasiswa-service=3 -d
     ```
3. **Peran Nginx API Gateway dalam Skenario Skala**:
   - Nginx dikonfigurasi dengan blok `upstream` untuk melakukan *Load Balancing* ke seluruh instans replika `mahasiswa-service`:
     ```nginx
     upstream mahasiswa_cluster {
         server mahasiswa-service-1:5000;
         server mahasiswa-service-2:5000;
         server mahasiswa-service-3:5000;
     }

     location /mahasiswa {
         proxy_pass http://mahasiswa_cluster;
     }
     ```
   - Nginx juga dapat menerapkan *caching* respons GET untuk query data yang sering dibaca berulang kali, sehingga mengurangi beban database.

---

### Pertanyaan 6
> **Bandingkan komunikasi sinkron (REST) dengan asinkron (message broker seperti RabbitMQ). Kapan masing-masing lebih cocok?**

**Jawaban:**

| Aspek | Komunikasi Sinkron (REST/HTTP) | Komunikasi Asinkron (Message Broker - RabbitMQ/Kafka) |
|---|---|---|
| **Mekanisme** | Klien mengirim request dan **menunggu** (*blocking*) sampai respons dikembalikan oleh server tujuan. | Pengirim mengirim pesan ke antrean (*queue/topic*) lalu langsung melanjutkan proses (*non-blocking* / *fire-and-forget*). |
| **Keterikatan (*Coupling*)** | Erat (*tight temporal coupling*): kedua service harus aktif dan merespons saat itu juga. | Longgar (*loose coupling*): penerima pesan bisa sedang offline atau memproses antrean sesuai kapasitas (*load leveling*). |
| **Kompleksitas** | Rendah: protokol standar, mudah di-debug dengan curl/browser/Postman. | Lebih tinggi: membutuhkan infrastruktur broker, penanganan *dead-letter queue*, *idempotency*, dan mekanisme *eventual consistency*. |
| **Latency Klien** | Tergantung pada rantai pemanggilan service terlama (*cumulative latency*). | Sangat cepat bagi klien karena request selesai segera setelah pesan masuk antrean. |

**Kapan Masing-Masing Lebih Cocok?**:
- **Gunakan REST (Sinkron)** saat:
  1. Klien membutuhkan data seketika itu juga (*immediate feedback* / query *read*), misalnya pengecekan detail profil mahasiswa, validasi login, atau pencarian mata kuliah.
  2. Alur transaksi sederhana yang tidak memiliki rantai dependensi panjang.
- **Gunakan Message Broker (Asinkron)** saat:
  1. Operasi membutuhkan waktu pemrosesan lama (*background tasks*), seperti pengiriman email notifikasi KRS, pembuatan kartu ujian dalam bentuk PDF, atau pemrosesan batch.
  2. Integrasi antar-domain event: misalnya event `MataKuliahDihapus` atau `MahasiswaLulus` yang perlu didengar oleh banyak service berbeda (pola *Publish-Subscribe*).
  3. Menghadapi lonjakan trafik ekstrem (*traffic spike* / war KRS) agar server tidak langsung *crash*, melainkan pesan diantrekan secara teratur.

---

## 6. Kesimpulan
Praktikum ini berhasil membuktikan bahwa arsitektur microservice memungkinkan dekomposisi aplikasi menjadi layanan-layanan modular dengan database terisolasi (*Database-per-Service*). Penggunaan Nginx sebagai API Gateway mempermudah akses klien melalui satu gerbang terpadu dan menyembunyikan kompleksitas infrastruktur backend. Mekanisme penanganan error yang baik terbukti krusial untuk mencegah kegagalan berantai (*cascading failure*) pada sistem terdistribusi.
