# ADR-007 — Backend Lokal Langsung di Laptop

**Status:** ACCEPTED  
**Tanggal:** 2026-10-13  
**Pengambil Keputusan:** Tim Pengembang FinTrack  
**Konsultasi / Stakeholder:** Tim Pengembang Backend & DevOps  

---

## 1. Konteks

Pada tahap awal pengembangan FinTrack (Milestone 0 dan Milestone 1), prioritas utama adalah kecepatan iterasi, kemudahan debugging, serta kesederhanaan lingkungan pengembangan bagi para kontributor tim di laptop masing-masing.

Tantangan dan batasan lingkungan pengembangan:
1. **Kompleksitas Tooling:** Memaksakan orkestrasi container yang rumit (Docker, Kubernetes) atau penyewaan VPS cloud sejak hari pertama menambah beban setup, kebutuhan resource RAM/CPU laptop, dan risiko ketergantungan yang memperlambat pembangunan fondasi kode.
2. **Kebutuhan Debugging Cepat:** Pengembang perlu menjalankan Spring Boot dan MySQL dengan kemampuan live breakpoint di IDE, inspeksi query Flyway migrasi, serta koneksi instan dari emulator Android (`10.0.2.2:8080`) atau perangkat fisik Android di jaringan Wi-Fi lokal.
3. **Pemisahan Kredensial & Secret:** Konfigurasi database lokal (password root/fintrack), Google Client Secret pengujian, dan JWT secret lokal tidak boleh bocor atau ter-commit ke Git repository publik (PRD NFR-022 dan .gitignore FinTrack).

---

## 2. Opsi yang Dipertimbangkan

### Opsi 1: Wajib Menggunakan VPS Cloud / Remote Dev Server Sejak Awal
- **Deskripsi:** Semua pengembang menghubungkan aplikasi Android lokal ke server staging/development yang di-host di VPS cloud.
- **Kelebihan:**
  - Lingkungan terpusat dan mendekati kondisi produksi.
- **Kekurangan:**
  - Membutuhkan biaya infrastruktur sejak tahap nol.
  - Membutuhkan setup domain, reverse proxy HTTPS, sertifikat SSL publik, dan manajemen secret cloud.
  - Tidak dapat melakukan development dan debugging saat pengembang offline atau kuota internet terbatas.

### Opsi 2: Wajib Menggunakan Docker Compose untuk Semua Komponen
- **Deskripsi:** Backend Spring Boot dan MySQL di-containerize dan wajib dijalankan melalui `docker-compose up`.
- **Kelebihan:**
  - Reproduksibilitas environment seragam antar mesin OS berbeda (Windows/Linux/macOS).
- **Kekurangan:**
  - Mengharuskan instalasi Docker Desktop yang berat bagi laptop dengan spesifikasi terbatas.
  - Siklus compile-reload di container lebih lambat daripada menjalankan Maven daemon / Spring Boot langsung di JVM laptop.

### Opsi 3: Backend Spring Boot dan MySQL Dijalankan Langsung di Laptop (Dipilih)
- **Deskripsi:** 
  - Backend Java 21 / Spring Boot dijalankan langsung melalui Maven Wrapper (`./mvnw spring-boot:run` atau IDE runner).
  - Database MySQL (versi 8.x) dijalankan langsung sebagai service lokal di laptop (atau Docker satu kontainer bagi yang memilih, namun Docker bukan prasyarat wajib).
  - Konfigurasi lokal menggunakan profil Spring `local` (`application-local.properties` / `application-local.yml` atau environment variables) yang sudah masuk ke `.gitignore`.
  - Transisi ke Docker Compose dan deployment VPS dijadwalkan pada milestone berikutnya setelah fitur inti dan kontrak API stabil.
- **Kelebihan:**
  - Setup paling cepat, ringan, dan langsung dapat di-debug dari IntelliJ IDEA / VS Code.
  - Tidak membebani resource mesin pengembang.
  - Mendukung pekerjaan penuh secara offline di laptop.
  - Sesuai dengan Architecture.md Bagian 2 ("Tahap pertama menjalankan Spring Boot dan MySQL langsung di laptop").
- **Kekurangan:**
  - Pengembang perlu menginstal JDK 21 dan instance MySQL lokal di mesin masing-masing.
  - Port (misal 3306 dan 8080) harus dikelola agar tidak bentrok dengan software lokal lain.

---

## 3. Keputusan

FinTrack mengadopsi **Opsi 3: Backend Lokal Dijalankan Langsung di Laptop (Bare-Metal JVM & MySQL Service)** untuk tahap Milestone 0 hingga Milestone 2.

Aturan pelaksanaan keputusan ini:
1. **Stack Runtime Lokal:**
   - Java Development Kit (JDK) 21 LTS.
   - MySQL 8.x berjalan di localhost port `3306`.
   - Spring Boot dijalankan di port `8080` melalui Maven Wrapper (`./mvnw` / `mvnw.cmd`).
2. **Konektivitas Klien Android:**
   - Emulator Android mengakses backend lokal via URL default: `http://10.0.2.2:8080/api/v1/`.
   - Perangkat fisik Android mengakses via IP LAN Wi-Fi laptop lokal (misal `http://192.168.x.x:8080/api/v1/`).
3. **Manajemen Konfigurasi dan Secret:**
   - Konfigurasi default di Git (`application.yml`) hanya memuat nilai placeholder netral atau profil dasar.
   - Konfigurasi kredensial lokal disimpan di file profil lokal (misal `application-local.yml` atau via environment variable) yang **wajib dikecualikan oleh `.gitignore`**.
   - Dilarang keras melakukan commit kredensial database riil atau JWT secret ke branch Git mana pun.
4. **Migrasi Database Terstandar:**
   - Inisialisasi skema dan evolusi tabel MySQL dikelola secara terpusat oleh **Flyway migration scripts** di `backend/src/main/resources/db/migration/`. Setiap migrasi wajib dapat dijalankan otomatis saat aplikasi Spring Boot start.
5. **Jadwal Evaluasi VPS / Docker:**
   - Docker Compose opsional dan dapat disediakan sebagai kemudahan tambahan di M0.5, namun tidak menjadi penghalang atau dependensi wajib.
   - Deployment VPS baru dieksekusi setelah HTTPS, verifikasi secret produksi, dan pengujian otorisasi siap (Architecture.md Bagian 2).

---

## 4. Konsekuensi

### Dampak Positif
- Waktu startup, build test, dan debugging di laptop menjadi sangat cepat.
- Zero cost hosting di fase awal pembuatan scaffold dan unit/integration testing.
- Proses onboarding pengembang menjadi transparan dengan instruksi setup lokal yang jelas (M0.5).

### Dampak Negatif & Trade-off
- Versi patch MySQL pada laptop pengembang bisa sedikit berbeda jika tidak dikunci (misal MySQL 8.0 vs 8.4).
- Pengembang bertanggung jawab menjaga instance MySQL lokal tetap aktif.

### Mitigasi & Tindak Lanjut
- Menyediakan dokumentasi prasyarat, perintah inisialisasi database `fintrack_db`, serta skrip start/stop yang jelas pada task M0.5.
- Memanfaatkan Flyway untuk memastikan integritas skema database selalu identik di seluruh mesin pengembang.

---

## 5. Referensi & Dokumen Terkait

- [Architecture.md Bagian 2 (`Stack - Lingkungan`)](../../Architecture.md)
- [.gitignore (Pola application-*.yml/properties)](../../.gitignore)
- [TODO.md Milestone 0.5: Dokumentasikan cara kerja lokal](../../TODO.md)
- [ADR-004: Google Verification dan Siklus JWT](ADR-004-google-verification-jwt.md)
