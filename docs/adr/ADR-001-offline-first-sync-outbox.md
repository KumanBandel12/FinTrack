# ADR-001 — Offline-first dan Sync Outbox

**Status:** ACCEPTED  
**Tanggal:** 2026-10-13  
**Pengambil Keputusan:** Tim Pengembang FinTrack  
**Konsultasi / Stakeholder:** Arsitek & Pengembang Android/Backend  

---

## 1. Konteks

FinTrack adalah aplikasi pencatatan keuangan pribadi harian yang digunakan pengguna dalam berbagai situasi jaringan, termasuk saat sinyal seluler lemah, di dalam ruangan tanpa Wi-Fi, atau tanpa akses internet sama sekali (offline). Pengguna menuntut pencatatan transaksi berlangsung instan tanpa terhambat latensi jaringan atau kegagalan koneksi.

Selain itu, privasi dan keandalan data keuangan adalah prioritas utama. Pengguna harus tetap dapat melihat saldo, riwayat transaksi, dan menginput pengeluaran kapan pun. Sinkronisasi ke server berfungsi sebagai backup awan dan agregasi lintas perangkat, bukan sebagai penghalang akses lokal.

Oleh karena itu, diperlukan arsitektur data lokal dan mekanisme sinkronisasi yang menjamin:
1. UI selalu responsif dan langsung membaca/menulis data lokal.
2. Tidak ada kehilangan data akibat kegagalan transmisi jaringan.
3. Aliran sinkronisasi terisolasi dari domain lokal sehingga kegagalan sync tidak membatalkan transaksi yang telah berhasil dicatat oleh pengguna.

---

## 2. Opsi yang Dipertimbangkan

### Opsi 1: Online-first dengan Caching Lokal
- **Deskripsi:** Semua operasi tulis (write) dikirim ke REST API server terlebih dahulu. Room hanya berfungsi sebagai cache baca (read cache) yang diperbarui setelah server merespons sukses.
- **Kelebihan:**
  - Konflik data di sisi klien minimal karena server menjadi single authority penentu state.
  - Implementasi Android lebih sederhana tanpa perlu antrean sinkronisasi kompleks.
- **Kekurangan:**
  - Aplikasi tidak dapat digunakan saat offline atau koneksi lambat.
  - Latensi terasa pada setiap input transaksi (menunggu round-trip network).
  - Melanggar prinsip dasar FinTrack (PRD.md FR-030).

### Opsi 2: Offline-first dengan Sync Langsung pada Entity Domain (Flag `is_synced`)
- **Deskripsi:** Transaksi disimpan di Room dengan kolom `is_synced: Boolean` atau `sync_status`. Worker mencari record dengan `is_synced = false`, mengirimnya ke server, dan mengubah flag menjadi `true`.
- **Kelebihan:**
  - Relatif sederhana di tahap awal tanpa tabel antrean terpisah.
- **Kekurangan:**
  - Menggabungkan concern domain data dengan concern operasional sinkronisasi.
  - Sulit menangani urutan operasi kompleks (misal update berulang, soft delete, atau retry spesifik per mutasi).
  - Berpotensi menimbulkan race condition jika pengguna mengubah transaksi saat payload sedang in-flight.

### Opsi 3: Offline-first dengan Pola Outbox Terpisah (Outbox Pattern) via WorkManager (Dipilih)
- **Deskripsi:** Room menjadi single source of truth untuk UI. Operasi mutasi domain (transaksi, rekening, transfer) ditulis secara atomik (dalam satu transaksi database Room) bersama sebuah entri operasi sinkronisasi ke tabel antrean `sync_operations` (Outbox). Background worker (WorkManager) memproses entri outbox secara berkala atau saat koneksi internet pulih.
- **Kelebihan:**
  - UI membaca data lokal seketika tanpa latensi jaringan.
  - Pemisahan bersih antara entity domain dan metadata sinkronisasi (retry count, status in-flight, idempotency key).
  - Transaksi lokal tetap tersimpan permanen di Room setelah sinkronisasi berhasil; hanya record outbox yang ditandai `SYNCED` atau di-purge sesuai kebijakan.
  - WorkManager mengelola batasan jaringan (NetworkType.CONNECTED), pengulangan dengan exponential backoff, dan ketahanan terhadap proses yang dimatikan sistem Android.
- **Kekurangan:**
  - Membutuhkan struktur tabel antrean outbox tambahan dan logika state machine (`PENDING`, `IN_FLIGHT`, `SYNCED`, `RETRY`, `CONFLICT`, `FAILED`).
  - Memerlukan penanganan resolusi konflik dan penjaminan idempotensi di backend.

---

## 3. Keputusan

FinTrack mengadopsi **Opsi 3: Offline-first dengan Pola Outbox Terpisah menggunakan Room dan WorkManager**.

Aturan pelaksanaan arsitektur ini:
1. **Penyimpanan Lokal:** Room Database adalah sumber pembacaan tunggal bagi ViewModel dan Jetpack Compose UI.
2. **Penulisan Atomik:** Setiap operasi buat/ubah/hapus pada domain (rekening, transaksi, transfer) wajib menyertakan pembuatan entri `SyncOperation` dalam satu transaksi Room `@Transaction`.
3. **Eksekusi Sync:** Sinkronisasi dijalankan oleh `WorkManager` dengan constraint jaringan (`Connected`). Operasi outbox diproses secara FIFO per entitas.
4. **Idempotensi:** Setiap operasi outbox mengusung `idempotency_key` dan UUID client (`client_id`) yang stabil agar transmisi ulang tidak memicu duplikasi di server.
5. **Retensi Data Lokal:** Setelah server merespons sukses (HTTP 200/201), data transaksi lokal **tidak boleh dihapus**; hanya entri outbox yang statusnya diperbarui menjadi `SYNCED`.
6. **Kegagalan Jaringan:** Kegagalan jaringan atau token kedaluwarsa hanya mempengaruhi status antrean outbox (pindah ke `RETRY`), dan tidak pernah menghapus atau membatalkan data finansial lokal pengguna.

---

## 4. Konsekuensi

### Dampak Positif
- Pengguna dapat mencatat keuangan secara instan dalam kondisi jaringan apa pun.
- Zero data loss pada level klien: mutasi tersimpan permanen di penyimpanan persisten sebelum mencoba transmisi jaringan.
- Logika antrean sinkronisasi terisolasi dan mudah diuji secara modular dengan unit test dan DAO test.

### Dampak Negatif & Trade-off
- Kompleksitas kode bertambah karena memerlukan entitas `SyncOperation`, worker outbox, serta penanganan error jaringan terperinci.
- Penyimpanan lokal membutuhkan ruang sedikit lebih besar untuk mengelola tabel antrean outbox sebelum di-cleanup.

### Mitigasi & Tindak Lanjut
- Menerapkan periodic cleanup untuk entri outbox berstatus `SYNCED` yang sudah melewati periode retensi tertentu (misalnya 7 hari).
- Menyediakan pengujian integrasi Room DAO dan unit test worker untuk memastikan invariant atomisitas penulisan terpenuhi.

---

## 5. Referensi & Dokumen Terkait

- [PRD.md FR-030 s.d. FR-035](../../PRD.md)
- [Architecture.md Bagian 1, 3, 7, dan 10](../../Architecture.md)
- [ADR-006: Idempotensi dan Penanganan Konflik Sinkronisasi](ADR-006-idempotensi-sync-conflict.md)
