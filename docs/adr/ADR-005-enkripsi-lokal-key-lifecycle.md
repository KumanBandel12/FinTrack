# ADR-005 — Enkripsi Lokal dan Key Lifecycle

**Status:** ACCEPTED  
**Tanggal:** 2026-10-13  
**Pengambil Keputusan:** Tim Pengembang FinTrack  
**Konsultasi / Stakeholder:** Arsitek Keamanan & Pengembang Android FinTrack  

---

## 1. Konteks

FinTrack menyimpan data finansial pengguna secara lokal di perangkat Android (buku kas, saldo, nominal pengeluaran, draft notifikasi bank). Sesuai prinsip *Secure by design* (PRD.md NFR-020 s.d. NFR-024 dan Architecture.md Bagian 12), data finansial di perangkat harus terlindung dari akses tidak sah apabila perangkat hilang, dicuri, atau di-root.

Tantangan utama keamanan lokal pada Android mencakup:
1. **Perlindungan Data at Rest:** File SQLite biasa di direktori aplikasi dapat diekstrak pada perangkat yang di-root atau melalui backup ADB jika tidak dilindungi.
2. **Manajemen Kunci Kriptografi:** Kunci enkripsi database dilarang keras di-hardcode di dalam source code APK karena mudah dibongkar lewat dekompilasi (reverse engineering). Kunci harus dibuat secara acak di perangkat dan diamankan oleh perangkat keras (hardware-backed Keystore).
3. **Pemisahan Peran App Lock vs Enkripsi:** PIN atau Biometrik (fingerprint/face unlock) sering disalahartikan sebagai enkripsi database. Padahal, biometric prompt adalah lapisan perlindungan UI/layar (app lock), bukan pengganti enkripsi database atau otentikasi API.
4. **Siklus Hidup Kunci (Key Lifecycle):** Perlu kejelasan prosedur saat pengguna melakukan sign-out, ganti perangkat, menghapus data akun, atau jika kunci Keystore rusak/hilang (misal setelah reset screen lock OS).

---

## 2. Opsi yang Dipertimbangkan

### Opsi 1: Mengandalkan Enkripsi Bawaan OS Android (Sandbox & FBE) Tanpa Enkripsi Database Tambahan
- **Deskripsi:** Menggunakan SQLite/Room standar tanpa library cipher, mengandalkan isolasi sandbox Linux Android dan File-Based Encryption (FBE) bawaan OS.
- **Kelebihan:**
  - Performa baca-tulis Room maksimal tanpa overhead kriptografi per-halaman.
  - Ukuran binary APK lebih kecil (tidak ada native `.so` library tambahan).
  - Tidak ada risiko korupsi database akibat kehilangan passphrase kunci.
- **Kekurangan:**
  - Tidak memenuhi standar kepatuhan aplikasi finansial pribadi tingkat tinggi.
  - Data tidak terlindungi jika perangkat di-root atau terdapat celah eksploitasi sandbox aplikasi.

### Opsi 2: Enkripsi Kolom Parsial (Field-Level Encryption)
- **Deskripsi:** Hanya mengenkripsi kolom tertentu (misal `amount_minor`, `description`) sebelum disimpan ke Room menggunakan AES-GCM.
- **Kelebihan:**
  - Tidak memerlukan library SQLite pihak ketiga.
- **Kekurangan:**
  - Mengacaukan kemampuan query SQL Room (misal `SUM(amount_minor)`, pencarian deskripsi, pengurutan nominal).
  - Metadata transaksi (tanggal, ID akun, relasi) tetap terbuka dalam plaintext.

### Opsi 3: Full Database Encryption (SQLCipher / AndroidX Security + Room) dengan Android Keystore (Dipilih)
- **Deskripsi:** Database Room dienkripsi penuh di level halaman SQLite menggunakan library yang kompatibel (seperti SQLCipher untuk Android via `SupportOpenHelperFactory`). Passphrase acak 256-bit dibuat saat inisialisasi pertama dan disimpan di Android Keystore / EncryptedSharedPreferences yang dilindungi hardware (TEE/StrongBox). PIN/BiometricPrompt digunakan sebagai gate pembuka sesi UI.
- **Kelebihan:**
  - Seluruh data at rest (tabel, index, skema, metadata) terenkripsi secara transparan.
  - Query Room (`SUM`, `ORDER BY`, relasi) tetap berjalan normal.
  - Kunci tidak pernah disimpan dalam plaintext dan tidak di-hardcode.
- **Kekurangan:**
  - Memerlukan integrasi native binary (`.so`), menambah ukuran APK sekitar 3–5 MB.
  - Wajib melewati tahap validasi kompatibilitas (M0.4 POC) untuk memastikan kestabilan pada minimum API 26 dan performa migrasi Room.

---

## 3. Keputusan

FinTrack mengadopsi **Opsi 3: Full Database Encryption menggunakan SQLCipher / Solusi Kompatibel Room dengan Manajemen Kunci Berbasis Android Keystore**.

Aturan pelaksanaan arsitektur keamanan ini:
1. **Penyimpanan Kunci Kriptografi:**
   - Kunci enkripsi database (passphrase 256-bit) dihasilkan secara kriptografis (`SecureRandom`) pada saat pertama kali aplikasi diinisialisasi.
   - Kunci disimpan menggunakan `EncryptedSharedPreferences` yang memanfaatkan `MasterKey` dari **Android Keystore System** (dengan flag hardware-backed jika didukung).
   - **Dilarang keras melakukan hardcode kunci** di resource, BuildConfig, atau konstanta kode.
2. **Lapisan Enkripsi Database:**
   - Room dihubungkan dengan enkripsi full-database melalui `SupportOpenHelperFactory` (SQLCipher atau paket keamanan resmi yang divalidasi pada M0.4).
   - Seluruh tabel domain (`transactions`, `transfers`, `accounts`, `transaction_drafts`, `sync_operations`) terenkripsi secara transparan.
3. **Peran App Lock (Biometrik / PIN):**
   - Fitur PIN / BiometricPrompt adalah mekanisme otentikasi lokal untuk **melindungi akses antarmuka (UI lock)** saat aplikasi resume dari background.
   - Biometrik **bukan pengganti enkripsi database** dan bukan pengganti token otentikasi API backend.
4. **Siklus Hidup Kunci & Penanganan Data (Lifecycle):**
   - **Sign-Out:** Saat pengguna keluar akun (sign out), token otentikasi backend dihapus dari Keystore. Pengguna diberi opsi untuk menghapus data lokal secara aman (purge local database) atau menyimpannya dalam keadaan terkunci.
   - **Penghapusan Data / Reset Akun:** File database `.db` ditutup dan dihapus secara fisik (`context.deleteDatabase()`), dan entri kunci di Keystore di-invalidate.
   - **Kehilangan / Kerusakan Kunci Keystore:** Jika kunci Keystore tidak dapat didekripsi (misalnya karena perubahan biometrik drastis di OS atau korupsi hardware), aplikasi mendeteksi kegagalan pembukaan DB secara aman, tidak crash liar, dan memandu pengguna untuk memulihkan data dari sinkronisasi cloud backend setelah verifikasi ulang.

---

## 4. Konsekuensi

### Dampak Positif
- Perlindungan data privasi finansial maksimal (defense in depth).
- Memenuhi spesifikasi PRD FR-004, NFR-020, NFR-021, dan Architecture.md Bagian 12.
- Mengurangi risiko paparan data jika perangkat fisik berpindah tangan tanpa izin.

### Dampak Negatif & Trade-off
- Sedikit penambahan waktu startup awal saat pembukaan koneksi database terenkripsi (sekitar 50–150 ms).
- Risiko data lokal tidak terbaca jika master key Android Keystore mengalami invalidasi OS.

### Mitigasi & Tindak Lanjut
- Melakukan verifikasi proof-of-concept pada M0.4 terkait penanganan startup database terenkripsi, migrasi skema Room, dan uji coba pembukaan DB tanpa kunci valid untuk memastikan penolakan akses bekerja benar.
- Menyiapkan mekanisme graceful error recovery: bila DB lokal rusak/terkunci tanpa kunci, arahkan pengguna untuk re-autentikasi dan sync ulang data dari server.

---

## 5. Referensi & Dokumen Terkait

- [PRD.md FR-004, NFR-020 s.d. NFR-024](../../PRD.md)
- [Architecture.md Bagian 12 (`Autentikasi dan Keamanan`)](../../Architecture.md)
- [ADR-001: Offline-first dan Sync Outbox](ADR-001-offline-first-sync-outbox.md)
- [ADR-004: Google Verification dan Siklus JWT](ADR-004-google-verification-jwt.md)
