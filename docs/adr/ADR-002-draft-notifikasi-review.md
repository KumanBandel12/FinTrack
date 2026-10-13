# ADR-002 — Draft Notifikasi Wajib Direview

**Status:** ACCEPTED  
**Tanggal:** 2026-10-13  
**Pengambil Keputusan:** Tim Pengembang FinTrack  
**Konsultasi / Stakeholder:** Tim Desain Produk & Pengembang Android  

---

## 1. Konteks

FinTrack memiliki fitur unggulan untuk menangkap notifikasi transaksi dari aplikasi perbankan dan dompet digital (e-wallet) melalui Android `NotificationListenerService`. Tujuannya adalah mempermudah pengguna mencatat transaksi tanpa perlu mengetik manual dari nol.

Namun, mengotomatisasi pencatatan keuangan dari notifikasi memiliki risiko kritis:
1. Format notifikasi bank dan e-wallet sering berubah tanpa pemberitahuan resmi.
2. Teks notifikasi sering kali ambigu (misalnya membedakan transfer keluar antar rekening sendiri vs pembayaran merchant vs biaya admin).
3. Notifikasi bisa terkirim berulang kali oleh sistem perbankan.
4. Auto-posting transaksi yang salah langsung ke buku besar (ledger) akan merusak saldo akun dan merusak kepercayaan pengguna terhadap akurasi aplikasi.
5. Menyimpan isi mentah (raw body) notifikasi berpotensi membocorkan data sensitif pengguna (seperti nomor rekening lengkap, saldo rekening bank, atau informasi pribadi lain) yang terbawa dalam payload.

Oleh karena itu, diperlukan keputusan tegas mengenai lifecycle notifikasi dan batas interaksi sistem sebelum data masuk ke saldo rekening.

---

## 2. Opsi yang Dipertimbangkan

### Opsi 1: Auto-Posting Otomatis ke Ledger
- **Deskripsi:** Ketika notifikasi masuk dan berhasil di-parse, sistem langsung membuat record `Transaction` dan memperbarui saldo rekening secara otomatis tanpa konfirmasi pengguna.
- **Kelebihan:**
  - Friction pengguna sangat rendah (zero click).
- **Kekurangan:**
  - Jika parser salah mendeteksi nominal atau tipe transaksi (income vs expense), saldo rekening langsung salah.
  - Sangat rentan terhadap duplicate posting jika notifikasi berulang diterima.
  - Pengguna tidak memiliki kesempatan melengkapi kategori atau memo transaksi saat itu juga.

### Opsi 2: Draft Notifikasi Wajib Direview (Review-Required Inbox) (Dipilih)
- **Deskripsi:** Setiap notifikasi dari aplikasi yang masuk allowlist diproses oleh parser ber-versi dan disimpan sebagai entitas sementara `TransactionDraft` dengan status `NEEDS_REVIEW`. Draft ini masuk ke kotak masuk (review inbox). Pengguna wajib memeriksa, melengkapi (misal memilih rekening tujuan atau kategori), lalu menekan konfirmasi (POST) atau menolak (REJECT).
- **Kelebihan:**
  - Menjaga integritas ledger dan invariant saldo 100% di bawah kendali pengguna.
  - Parser yang tidak yakin dapat memberi status `UNPARSED` atau menandai field yang belum lengkap tanpa memecahkan saldo.
  - Jika terdeteksi transfer satu sisi (misal notifikasi debet bank tanpa tahu rekening tujuan), draft tidak memotong saldo sebagai transfer lengkap sampai pengguna menentukan rekening penerima.
  - Memungkinkan minimisasi data: isi notifikasi mentah (raw body) tidak disimpan secara permanen di database lokal, hanya field transaksi yang ter-ekstraksi.
- **Kekurangan:**
  - Memerlukan tindakan manual tambahan dari pengguna di layar Inbox Review.
  - Membutuhkan entitas dan layar UI khusus untuk mengelola lifecycle draft.

### Opsi 3: Auto-Posting Bersyarat dengan Ambang Kepercayaan (Confidence Threshold)
- **Deskripsi:** Jika confidence parser 100%, transaksi di-post otomatis. Jika di bawah 100%, masuk ke draft review.
- **Kelebihan:**
  - Mengurangi jumlah notifikasi yang perlu direview manual untuk transaksi berulang standar.
- **Kekurangan:**
  - Sangat sulit mengukur skor confidence parser teks secara deterministik tanpa machine learning yang kompleks di perangkat.
  - Risiko false positive tetap tinggi dan inkonsistensi perilaku aplikasi membingungkan pengguna.

---

## 3. Keputusan

FinTrack mengadopsi **Opsi 2: Draft Notifikasi Wajib Direview (Review-Required)**.

Aturan pelaksanaan keputusan ini:
1. **Tidak Ada Auto-Posting:** Tidak ada notifikasi yang langsung mengubah saldo rekening atau masuk ke tabel `transactions` tanpa aksi konfirmasi eksplisit dari pengguna.
2. **Entitas Terpisah:** Notifikasi masuk disimpan ke tabel `transaction_drafts` dengan status awal `NEEDS_REVIEW` atau `UNPARSED`.
3. **Minimisasi Data Sensitif:** Teks mentah notifikasi (raw notification body) **dilarang disimpan secara default** di database lokal. Hanya field terstruktur (kandidat nominal, tanggal, petunjuk sumber, deduplication hash) yang disimpan.
4. **Penanganan Transfer Satu Sisi:** Jika notifikasi hanya mendeteksi mutasi keluar (debet) tanpa informasi rekening tujuan, sistem dilarang membukukan transfer penuh. Sistem hanya membuat draft dan meminta pengguna memilih apakah transaksi tersebut pengeluaran biasa atau transfer antar-rekening sendiri beserta akun tujuannya.
5. **Deduplikasi Sebelum Review:** Candidate draft di-deduplikasi menggunakan kombinasi signature `(source_app_id, timestamp_window, amount_minor, dedupe_key)` untuk mencegah tumpukan draft ganda dari notifikasi berulang.
6. **Aliran ke Ledger:** Hanya setelah pengguna mengonfirmasi draft (`CONFIRMED`), use case domain akan membuat entri transaksi/transfer di Room dan mendaftarkannya ke antrean outbox sync. Draft yang ditolak (`REJECTED`) diarsipkan tanpa mempengaruhi saldo.

---

## 4. Konsekuensi

### Dampak Positif
- Integritas data saldo rekening terjamin karena setiap posting diverifikasi oleh manusia.
- Privasi pengguna terjaga sesuai regulasi dan prinsip arsitektur secure-by-design.
- Kerusakan akibat bug regex/parser di lapangan tidak merusak histori buku besar pengguna.

### Dampak Negatif & Trade-off
- Pengguna harus membuka aplikasi dan mereview draft sebelum saldo ter-update.
- Dibutuhkan komponen UI Inbox Review dengan badge jumlah draft yang menunggu konfirmasi.

### Mitigasi & Tindak Lanjut
- Mendesain UI Review yang sangat cepat dan intuitif (batch confirm atau swipe action dengan neobrutalist action buttons).
- Memberikan notifikasi pengingat lokal yang tidak mengganggu jika terdapat draft yang menumpuk.

---

## 5. Referensi & Dokumen Terkait

- [PRD.md FR-020 s.d. FR-028](../../PRD.md)
- [Architecture.md Bagian 7 (`TransactionDraft`) dan Bagian 9 (`Pipeline Notifikasi`)](../../Architecture.md)
- [ADR-003: Transfer dengan Dua Efek Ledger](ADR-003-transfer-dua-efek-ledger.md)
- [ADR-008: Allowlist dan Sumber Notifikasi Pertama](ADR-008-allowlist-sumber-notifikasi.md)
