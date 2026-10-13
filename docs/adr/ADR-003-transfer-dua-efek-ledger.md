# ADR-003 — Transfer dengan Dua Efek Ledger

**Status:** ACCEPTED  
**Tanggal:** 2026-10-13  
**Pengambil Keputusan:** Tim Pengembang FinTrack  
**Konsultasi / Stakeholder:** Tim Arsitektur & Domain Keuangan FinTrack  

---

## 1. Konteks

Dalam aplikasi pencatatan keuangan, perpindahan dana antar rekening pengguna (misalnya memindahkan saldo dari Bank BCA ke Gopay, atau tarik tunai dari ATM ke Dompet Fisik) adalah transaksi harian yang sangat umum.

Terdapat beberapa tantangan fundamental dalam pemodelan data transfer:
1. **Konsistensi Saldo:** Transfer harus mengubah saldo dua rekening berbeda secara atomik dan simetris (rekening asal berkurang, rekening tujuan bertambah).
2. **Biaya Transfer (Fee):** Transaksi transfer sering kali mengenakan biaya admin (fee). Biaya ini merupakan pengeluaran murni, bukan perpindahan modal.
3. **Analitik Keuangan:** Transfer bukanlah pemasukan (income) dan bukan pula pengeluaran (expense) dari kekayaan bersih (net worth) pengguna. Jika transfer dihitung sebagai expense di rekening asal dan income di rekening tujuan, laporan arus kas bulanan akan terdistorsi secara drastis.
4. **Validasi Domain:** Rekening asal dan rekening tujuan tidak boleh sama. Rekening yang diarsipkan tidak boleh menjadi tujuan transfer aktif baru.
5. **Koreksi dan Rekonsiliasi:** Jika pengguna mengedit atau membatalkan transfer, efek kedua sisi rekening harus diperbarui secara konsisten.

---

## 2. Opsi yang Dipertimbangkan

### Opsi 1: Dua Transaksi Terpisah yang Tidak Terhubung
- **Deskripsi:** Pengguna mencatat satu transaksi `EXPENSE` pada rekening asal dan satu transaksi `INCOME` pada rekening tujuan secara manual tanpa relasi sistem.
- **Kelebihan:**
  - Tidak membutuhkan model data baru; cukup menggunakan tabel `transactions`.
- **Kekurangan:**
  - Analitik pendapatan dan pengeluaran menjadi salah total (double counting arus kas).
  - Tidak ada jaminan atomisitas: pengguna bisa saja mengedit atau menghapus salah satu sisi sehingga saldo total menjadi tidak seimbang.
  - Pengguna harus mengetik dua kali untuk satu peristiwa finansial.

### Opsi 2: Entitas Transfer Logis Tunggal dengan Dua Efek Ledger Tertaut (Dipilih)
- **Deskripsi:** Dibuat model domain `Transfer` sebagai satu operasi logis yang mengikat `source_account_id` dan `destination_account_id`. Operasi ini mengeksekusi dua efek ledger secara atomik dalam transaksi database:
  1. Pengurangan pokok pada rekening asal (`-amount_minor`).
  2. Penambahan pokok pada rekening tujuan (`+amount_minor`).
  3. Jika terdapat `fee_minor > 0`, fee dicatat sebagai efek pengeluaran terpisah (`EXPENSE`) yang melekat pada rekening asal.
  Pada lapisan kalkulasi analitik, pokok transfer dikecualikan dari total pemasukan/pengeluaran pengguna.
- **Kelebihan:**
  - Menjaga invariant saldo: total kekayaan bersih tidak berubah oleh pokok transfer.
  - Perubahan atau pembatalan (void) transfer langsung mengembalikan saldo kedua rekening secara konsisten dan atomik.
  - Biaya admin terlacak transparan sebagai pengeluaran riil pengguna.
  - Mencegah kekeliruan pemilihan rekening asal = rekening tujuan di level validasi domain.
- **Kekurangan:**
  - Membutuhkan penanganan khusus dalam query kalkulasi saldo rekening dan query analitik laporan bulanan.
  - Sinkronisasi payload transfer melibatkan validasi integritas kedua rekening di backend.

### Opsi 3: Entitas Transaksi dengan Flag Pasangan (`paired_transaction_id`)
- **Deskripsi:** Menggunakan dua baris di tabel `transactions`, tetapi masing-masing memiliki foreign key yang merujuk ke ID transaksi pasangannya.
- **Kelebihan:**
  - Tetap menggunakan tabel `transactions` yang sama.
- **Kekurangan:**
  - Rawan inkonsistensi saat pembaruan parsial jika query update lupa mengunci kedua baris.
  - Validasi constraint foreign key sirkular (`A.paired_id = B.id` dan `B.paired_id = A.id`) rumit di SQLite/Room dan MySQL.

---

## 3. Keputusan

FinTrack mengadopsi **Opsi 2: Entitas Transfer Logis Tunggal dengan Dua Efek Ledger Tertaut**.

Aturan pelaksanaan keputusan ini:
1. **Model Domain Eksplisit:** Entitas `Transfer` didefinisikan dengan atribut: `id`, `client_id`, `user_id`, `source_account_id`, `destination_account_id`, `amount_minor`, `fee_minor`, `currency_code`, `occurred_at`, `description`, `status` (`POSTED`, `VOIDED`), dan `version`.
2. **Validasi Rekening:** `source_account_id` **tidak boleh sama** dengan `destination_account_id`. Kedua rekening harus berada di bawah kepemilikan `user_id` yang sama dan menggunakan mata uang yang kompatibel.
3. **Penghitungan Saldo Rekening:**
   - Saldo Rekening Asal: berkurang sebesar `amount_minor + fee_minor`.
   - Saldo Rekening Tujuan: bertambah sebesar `amount_minor`.
4. **Isolasi Biaya Admin (Fee):** `fee_minor` diakui sebagai beban pengeluaran (expense) dari rekening asal dan masuk ke dalam kalkulasi total pengeluaran pada periode terkait.
5. **Eksklusi dari Analitik Arus Kas:** Pokok perpindahan (`amount_minor`) **dilarang keras** dihitung sebagai Income maupun Expense dalam grafik pengeluaran, ringkasan bulanan, atau perhitungan cash flow bersih pengguna.
6. **Atomisitas Operasi:** Pembuatan, pembaruan nominal/rekening, atau pembatalan transfer di Room maupun MySQL wajib dieksekusi dalam satu transaksi database (`@Transaction` di Room / `@Transactional` di Spring Boot).
7. **Bukan Rekonsiliasi:** Koreksi saldo karena selisih fisik yang tidak diketahui tujuannya tidak boleh memakai transfer palsu, melainkan harus menggunakan mekanisme `Adjustment` bertanggal dengan alasan eksplisit.

---

## 4. Konsekuensi

### Dampak Positif
- Integritas data buku besar ganda terjaga secara matematis.
- Laporan keuangan, dashboard, dan grafik arus kas akurat tanpa distorsi double counting.
- Pengalaman pengguna jauh lebih baik: cukup satu form input untuk transfer antar-akun.

### Dampak Negatif & Trade-off
- Perlu membuat tabel tersendiri (`transfers`) atau view ledger yang menggabungkan entri transaksi dan transfer saat menampilkan riwayat per rekening.
- Sinkronisasi transfer membutuhkan penanganan jika salah satu rekening belum tersinkronkan ke backend.

### Mitigasi & Tindak Lanjut
- Gunakan DAO / Repository method yang memfasilitasi query feed riwayat gabungan (`transactions` UNION `transfers`) secara efisien.
- Menulis unit test komprehensif untuk memastikan invariant saldo kedua rekening dan eksklusi analitik teruji.

---

## 5. Referensi & Dokumen Terkait

- [PRD.md FR-015 s.d. FR-019](../../PRD.md)
- [Architecture.md Bagian 7 (`Transfer`) dan Bagian 8 (`Invariant Saldo`)](../../Architecture.md)
- [ADR-001: Offline-first dan Sync Outbox](ADR-001-offline-first-sync-outbox.md)
