# ADR-006 — Idempotensi dan Konflik Sinkronisasi

**Status:** ACCEPTED  
**Tanggal:** 2026-10-13  
**Pengambil Keputusan:** Tim Pengembang FinTrack  
**Konsultasi / Stakeholder:** Tim Arsitektur Sistem Android & Backend  

---

## 1. Konteks

FinTrack mengimplementasikan model sinkronisasi offline-first dua arah (client-to-server push dan server-to-client pull). Dalam kondisi jaringan seluler yang fluktuatif, request pengiriman data sering kali mengalami putus koneksi di tengah jalan (misal timeout setelah server berhasil menulis ke database, tetapi sebelum acknowledgment sampai ke aplikasi klien).

Risiko-risiko yang harus dicegah secara sistemik:
1. **Duplikasi Transaksi (Duplicate Posting):** Jika klien mengulangi pengiriman (retry) tanpa mekanisme penanda unik, server berpotensi membuat record transaksi baru untuk transaksi yang sebenarnya sama.
2. **Konflik Pembaruan Bersamaan (Concurrent Modification Conflict):** Transaksi atau saldo rekening yang sama bisa saja diperbarui di perangkat lain atau diedit ulang secara lokal sebelum perubahan sebelumnya selesai disinkronkan.
3. **Kehilangan Data Lokal (Silent Overwrite / Data Loss):** Menimpa data lokal secara diam-diam (silent overwrite) atau menghapus mutasi lokal pengguna ketika terjadi konflik server melanggar integritas aplikasi finansial (PRD FR-035).
4. **Penebalan Log Error Sensitif:** Pesan kegagalan sinkronisasi yang dikirim balik ke klien tidak boleh membocorkan informasi internal server (seperti stack trace database atau query SQL).

---

## 2. Opsi yang Dipertimbangkan

### Opsi 1: Last-Write-Wins (LWW) Berbasis Timestamp Klien
- **Deskripsi:** Server membandingkan `updated_at` dari klien dengan server; mutasi dengan timestamp paling baru selalu menimpa mutasi yang lebih lama tanpa mencatat konflik.
- **Kelebihan:**
  - Sangat mudah diimplementasikan.
- **Kekurangan:**
  - Jam perangkat klien (client clock) tidak dapat dipercaya (rentan drift, manipulasi waktu pengguna, atau zona waktu berbeda).
  - Menyebabkan *silent data loss*: perubahan penting pengguna dapat terhapus tanpa ada pemberitahuan atau audit log.

### Opsi 2: Sinkronisasi Terpusat Penuh dengan Locking Sesi (Pessimistic)
- **Deskripsi:** Klien harus mengunci akun di server sebelum dapat menulis mutasi apa pun.
- **Kelebihan:**
  - Bebas konflik konkuren.
- **Kekurangan:**
  - Tidak kompatibel dengan konsep offline-first; aplikasi tidak bisa digunakan saat offline.

### Opsi 3: Idempotensi Berbasis UUID Stabil + Optimistic Versioning + Resolusi Eksplisit (Dipilih)
- **Deskripsi:** 
  1. **Identitas Stabil:** Setiap entitas domain dibuat di klien dengan `client_id` (UUID v4) yang tidak pernah berubah seumur hidup record tersebut.
  2. **Idempotensi Tulis:** Setiap mutasi outbox membawa `idempotency_key` unik. Backend menyimpan dan memeriksa key ini dalam batas retensi waktu (misal 24–48 jam). Jika request dengan key yang sama diterima lagi, backend mengembalikan status keberhasilan yang sama tanpa membuat record baru.
  3. **Optimistic Versioning:** Setiap entitas memiliki atribut integer `version`. Saat update dikirim, versi yang diharapkan disertakan. Jika versi di server sudah lebih tinggi, backend menolak pembaruan dengan respons status `409 Conflict`.
  4. **Penanganan Klien:** Klien tidak pernah membuang perubahan lokal. Entri outbox diberi status `CONFLICT`. Logika sync melakukan rekonsiliasi eksplisit (misal membuat penyesuaian saldo terpisah atau mempertahankan revisi lokal yang lebih baru).
  5. **Bounded Retry:** Kegagalan jaringan transient di-retry dengan exponential backoff dan batas maksimum percobaan (`max_attempts`).
- **Kelebihan:**
  - Mencegah duplikasi data 100% pada saat jaringan putus-sambung dan retry otomatis.
  - Aman terhadap manipulasi jam perangkat pengguna karena menggunakan integer versi monotonik.
  - Zero silent data loss: data lokal pengguna tetap utuh.
- **Kekurangan:**
  - Backend harus memelihara tabel/kolom idempotensi dan validasi nomor versi.
  - Logika klien harus membedakan error yang dapat di-retry (transient 5xx, network error) vs non-retryable (400 validation, 401 unauthenticated, 403 forbidden, 409 conflict).

---

## 3. Keputusan

FinTrack mengadopsi **Opsi 3: Idempotensi Berbasis UUID Klien Stabil, Optimistic Versioning, dan Resolusi Konflik Eksplisit**.

Aturan pelaksanaan keputusan ini:
1. **Identitas Entitas Klien:**
   - Semua entitas domain (`accounts`, `categories`, `transactions`, `transfers`) wajib memiliki kolom `client_id` bertipe UUID v4 yang digenerate oleh klien saat pembuatan lokal.
   - Kolom `client_id` menjadi kunci relasi lokal dan kunci penjaminan idempoten di backend (unik per pengguna).
2. **Kunci Idempotensi (`idempotency_key`):**
   - Setiap operasi di tabel `sync_operations` memiliki `idempotency_key` (berisi kombinasi tipe operasi, client_id entitas, dan versi mutasi).
   - Endpoint `POST /api/v1/sync/push` di backend memvalidasi `idempotency_key`. Jika mutasi sudah pernah diproses, server mengembalikan status sukses (idempotent reply) dengan status DTO terkini tanpa memproses ulang ledger.
3. **Pendeteksian Konflik Versi:**
   - Pembaruan entitas menyertakan `version`. Backend menolak operasi jika `client_version != server_version` dengan HTTP 409 Conflict.
4. **Prinsip Larangan Silent Overwrite:**
   - Data lokal di Room **dilarang keras ditimpa secara diam-diam atau dihapus** ketika terjadi konflik atau error sinkronisasi.
   - Mutasi lokal yang mengalami konflik ditandai dengan status `CONFLICT` di antrean outbox, dan ditangani melalui rekonsiliasi yang transparan bagi pengguna.
5. **Kebijakan Retry Terukur (Bounded Retry):**
   - Retry otomatis WorkManager hanya dijalankan untuk error jaringan transient (koneksi terputus, timeout, HTTP 503/504).
   - Retry menggunakan exponential backoff (`initialDelay = 10s`, `maxDelay = 1 hour`) dengan batas maksimum upaya (misal 5 kali) sebelum ditandai `FAILED` untuk review manual.
   - Error otorisasi (401/403) atau error validasi skema (400) **tidak boleh di-retry secara buta**, melainkan memicu re-autentikasi atau sanitasi data.
6. **Sanitasi Pesan Error:**
   - Kolom `last_error_code` pada outbox hanya menyimpan kode error terstandarisasi yang disanitasi (misal `NETWORK_TIMEOUT`, `SERVER_UNAVAILABLE`, `VERSION_CONFLICT`), bukan raw SQL trace atau payload sensitif.

---

## 4. Konsekuensi

### Dampak Positif
- Integritas data keuangan terjaga mutlak: tidak ada posting ganda atau saldo terpotong dua kali akibat retry koneksi.
- Ketahanan tinggi terhadap ketidakstabilan jaringan seluler khas penggunaan mobile di Indonesia.
- Auditabilitas sinkronisasi jelas dan dapat ditelusuri.

### Dampak Negatif & Trade-off
- Membutuhkan penanganan logika branching error yang teliti di lapisan Android Repository/WorkManager.
- Skema database di server harus memiliki constraint unique pada `(user_id, client_id)` dan tabel idempotensi.

### Mitigasi & Tindak Lanjut
- Membuat unit test khusus untuk simulasi skenario: retry request yang sama, transmisi dengan versi usang (stale version), dan error transient vs fatal.
- Menyediakan status sinkronisasi yang jelas di layar Settings (misal "Semua data tersinkron", "3 perubahan menunggu jaringan", atau "1 konflik perlu perhatian").

---

## 5. Referensi & Dokumen Terkait

- [PRD.md FR-031 s.d. FR-035](../../PRD.md)
- [Architecture.md Bagian 7 (`SyncOperation`), Bagian 10 (`Sinkronisasi Offline-first`), dan Bagian 11 (API usulan)](../../Architecture.md)
- [ADR-001: Offline-first dan Sync Outbox](ADR-001-offline-first-sync-outbox.md)
