# FinTrack — Dokumen Persyaratan Produk (PRD)

**Versi:** 1.0 — baseline rancangan  
**Jenis proyek:** Serious side project untuk penggunaan pribadi  
**Platform:** Android native; distribusi awal APK manual  
**Bahasa dokumentasi dan UI:** Bahasa Indonesia

## 1. Ringkasan produk

FinTrack adalah aplikasi pencatatan keuangan pribadi Android untuk mencatat dan memantau saldo beberapa rekening, pemasukan, pengeluaran, transfer, serta penyesuaian saldo. Transaksi dapat dimasukkan secara manual atau dibuat sebagai draft dari notifikasi aplikasi keuangan Android yang didukung. Draft notifikasi wajib ditinjau pengguna sebelum menjadi transaksi tercatat.

FinTrack menggunakan pendekatan **offline-first**. Data ditulis ke penyimpanan lokal terlebih dahulu, lalu disinkronkan ke backend saat jaringan tersedia. Data transaksi lokal tetap disimpan setelah sinkronisasi sukses. FinTrack bukan layanan bank atau open banking dan tidak menjamin seluruh transaksi dapat ditangkap dari notifikasi.

## 2. Masalah, tujuan, dan bukan tujuan

### Masalah
Pencatatan manual memerlukan usaha berulang; parser notifikasi dapat salah membaca nominal, tanggal, atau jenis transaksi. Pengguna membutuhkan pencatatan yang praktis tetapi tetap dapat dikoreksi, ditelusuri, digunakan saat offline, dan direkonsiliasi dengan saldo aktual.

### Tujuan
1. Memusatkan histori rekening tunai, bank, e-wallet, dan sumber lain yang ditambahkan pengguna.
2. Mendukung transaksi manual dan draft notifikasi dengan konfirmasi pengguna.
3. Menghitung saldo dari saldo awal, transaksi, transfer, dan penyesuaian yang tercatat.
4. Menyinkronkan data secara idempoten tanpa menghapus data lokal.
5. Menyediakan dashboard dan analitik bertahap: budget, target menabung, kalender, perbandingan bulanan, ekspor, backup/pemulihan.
6. Menjaga data melalui autentikasi, otorisasi per pengguna, transport aman, penyimpanan token aman, dan minimisasi data sensitif.

### Bukan tujuan MVP
- Melakukan pembayaran atau transfer uang nyata.
- Menjamin seluruh notifikasi terdeteksi.
- Memposting draft notifikasi tanpa konfirmasi.
- Menggantikan catatan resmi bank.
- Mendukung iOS atau kolaborasi lintas perangkat real-time.
- Memberikan rekomendasi investasi atau nasihat keuangan otomatis.
- Meluncurkan aplikasi secara publik sebelum review keamanan dan kebijakan distribusi.

## 3. Pengguna sasaran dan asumsi

- Pengguna utama adalah individu yang ingin menggabungkan catatan beberapa rekening.
- Login wajib; identitas Google diverifikasi backend lalu API menggunakan token aplikasi/JWT.
- Setup awal mewajibkan minimal satu rekening, saldo awal, dan tanggal efektifnya.
- Stack Android: Kotlin + Jetpack Compose, minimum Android API 26.
- Backend: Java 21 + Spring Boot 3.5.x + MySQL.
- Backend dan database dijalankan langsung di laptop pada tahap awal; VPS adalah tahap lanjutan.
- Bahasa UI dan dokumentasi Indonesia; mata uang utama IDR.
- Tampilan menggunakan arah Neobrutalism dan font Sora, light/dark dengan opsi override.

## 4. Alur utama

1. **Setup:** login Google → buat rekening → isi saldo awal/tanggal → atur PIN/biometrik → opsional aktifkan akses notifikasi.
2. **Input manual:** pilih rekening dan jenis transaksi → isi nominal, tanggal, kategori, deskripsi → validasi → simpan ke Room dan antrean sinkronisasi.
3. **Notifikasi:** listener menerima event → allowlist sumber → parser → validasi dan deduplikasi → draft lokal → pengguna mengedit, mengonfirmasi, atau menolak → hanya transaksi yang dikonfirmasi masuk ledger.
4. **Transfer:** pilih rekening sumber/tujuan → nominal, tanggal, biaya → satu operasi transfer logis dengan dua efek ledger.
5. **Offline:** UI membaca Room; sinkronisasi dicoba WorkManager saat jaringan tersedia; data transaksi lokal tetap ada.
6. **Rekonsiliasi:** masukkan saldo aktual yang diamati → aplikasi mencatat penyesuaian bertanggal dan alasan; tidak menimpa histori diam-diam.

## 5. User stories

- Pengguna dapat membuat banyak rekening tunai, bank, dan e-wallet.
- Pengguna wajib mengisi saldo awal dan tanggalnya saat membuat rekening pertama.
- Pengguna dapat mencatat/mengubah pemasukan, pengeluaran, transfer, dan penyesuaian.
- Pengguna dapat meninjau, memperbaiki, mengonfirmasi, atau menolak draft notifikasi.
- Pengguna dapat mengoperasikan pencatatan tanpa koneksi internet.
- Pengguna dapat memahami komponen saldo dan merekonsiliasi perbedaan.
- Pengguna dapat membuat, mengubah, dan mengarsipkan kategori.
- Pengguna dapat melihat budget, target menabung, kalender transaksi, dan perbandingan bulanan.
- Pengguna dapat mengekspor dan memulihkan catatan.
- Pengguna dapat melindungi akses lokal dengan PIN/biometrik.

## 6. Kebutuhan fungsional

Prioritas: **P0** fondasi transaksi dan keamanan; **P1** fitur produk versi awal yang dibangun setelah fondasi stabil tetapi tetap termasuk target rilis awal; **P2** fitur lanjutan di luar target rilis awal.

### Identitas dan setup
- **FR-001 (P0):** Wajib autentikasi sebelum membuka data privat.
- **FR-002 (P0):** Backend memverifikasi identitas Google; email atau user ID dari klien bukan bukti autentikasi.
- **FR-003 (P0):** Semua akses data server dibatasi untuk pengguna terautentikasi yang tepat.
- **FR-004 (P0):** Setup meminta rekening, saldo awal, dan tanggal efektif.
- **FR-005 (P0):** Izin notifikasi dijelaskan dan bersifat opsional; input manual tetap berfungsi jika ditolak.

### Rekening dan ledger
- **FR-010 (P0):** Rekening memiliki ID stabil, nama, tipe, mata uang, saldo awal, tanggal saldo awal, status arsip, dan timestamp.
- **FR-011 (P0):** Mendukung pemasukan, pengeluaran, transfer, dan penyesuaian saldo.
- **FR-012 (P0):** Saldo dihitung dari saldo awal dan efek semua entri ledger yang berlaku.
- **FR-013 (P0):** Ada kategori bawaan; pengguna dapat membuat, mengubah, dan mengarsipkan kategori sendiri.
- **FR-014 (P0):** Perubahan/pembatalan transaksi mempertahankan konsistensi saldo dan jejak koreksi yang relevan.
- **FR-015 (P0):** Transfer adalah satu operasi logis dengan dua efek ledger tertaut.
- **FR-016 (P0):** Biaya transfer dicatat terpisah sebagai pengeluaran.
- **FR-017 (P0):** Rekonsiliasi membuat penyesuaian eksplisit dan bertanggal; histori tidak ditimpa untuk memaksa saldo cocok.
- **FR-018 (P0):** Uang menggunakan integer satuan terkecil atau representasi desimal presisi; floating point dilarang.
- **FR-019 (P0):** Transfer dikecualikan dari agregasi pemasukan/pengeluaran.

### Notifikasi
- **FR-020 (P0):** Gunakan `NotificationListenerService` untuk menerima event notifikasi.
- **FR-021 (P0):** Proses hanya aplikasi pada allowlist dan pola yang dikenali sebagai kandidat transaksi.
- **FR-022 (P0):** Setiap kandidat disimpan sebagai draft lokal dan wajib dikonfirmasi atau ditolak.
- **FR-023 (P0):** Draft bisa diedit sebelum dikonfirmasi.
- **FR-024 (P0):** Deduplikasi menggunakan identitas stabil/idempotency key yang sesuai.
- **FR-025 (P0):** Isi notifikasi mentah tidak disimpan secara default; simpan data minimum yang diperlukan.
- **FR-026 (P0):** Tampilkan status saat akses notifikasi dicabut/tidak tersedia.
- **FR-027 (P1):** Tambah parser per sumber secara bertahap dengan fixture sintetis/teranonimkan dan versi parser.
- **FR-028 (P0):** Jika hanya satu sisi transfer terdeteksi, buat draft; jangan mengubah saldo seolah transfer lengkap.

### Penyimpanan dan sinkronisasi
- **FR-030 (P0):** Data transaksi dan rekening disimpan lokal menggunakan Room.
- **FR-031 (P0):** Antrean sinkronisasi terpisah dari data domain.
- **FR-032 (P0):** WorkManager menjalankan sync berdasarkan kondisi jaringan dan retry yang terukur.
- **FR-033 (P0):** Setelah sync sukses, data transaksi lokal tetap disimpan; hanya operasi antrean yang ditandai selesai.
- **FR-034 (P0):** ID stabil dan idempotensi mencegah duplikasi saat retry.
- **FR-035 (P0):** Jaringan gagal, token kedaluwarsa, input invalid, dan konflik versi tidak boleh menghilangkan data lokal.

### Analitik dan pemulihan
- **FR-040 (P0):** Dashboard menampilkan saldo, transaksi terbaru, total pemasukan/pengeluaran, serta ringkasan kategori.
- **FR-041 (P1):** Budget per kategori/periode.
- **FR-042 (P1):** Target menabung.
- **FR-043 (P1):** Kalender transaksi dan perbandingan bulanan.
- **FR-044 (P1):** Ekspor data ke CSV dan format yang dapat digunakan di Excel (misalnya CSV yang kompatibel; dukungan XLSX native dapat dipilih berdasarkan kebutuhan dan dependency).
- **FR-045 (P1):** Backup/pemulihan memiliki kebijakan enkripsi, retensi, dan penghapusan.
- **FR-046 (P2):** Deteksi transaksi berulang/langganan.
- **FR-047 (P2):** Peringatan pengeluaran tidak biasa dengan aturan transparan.
- **FR-048 (P2):** Peramalan pengeluaran dengan asumsi yang dijelaskan.
- **FR-049 (P2):** Impor file dengan preview, pemetaan, validasi, dan deduplikasi.
- **FR-050 (P2):** Kueri bahasa alami hanya setelah evaluasi privasi dan pembatasan akses.

### Keamanan
- **FR-060 (P0):** PIN dan/atau biometrik untuk app lock.
- **FR-061 (P0):** Lindungi token dengan mekanisme Android Keystore; tidak disimpan sebagai teks biasa di preferensi.
- **FR-062 (P0):** HTTPS untuk deployment.
- **FR-063 (P0):** Backend memeriksa autentikasi dan ownership pada semua endpoint privat.
- **FR-064 (P0):** Jangan menaruh token, isi notifikasi, dan detail keuangan pada log.
- **FR-065 (P0):** Perilaku data lokal setelah sign-out ditentukan dan diuji.
- **FR-066 (P0):** Pilih dan uji enkripsi database lokal; jangan mengklaim terenkripsi sebelum terbukti.
- **FR-067 (P1):** Alur ekspor/penghapusan data server dan retensi backup terdokumentasi.

## 7. Kebutuhan nonfungsional

- **Keandalan:** transaksi lokal bertahan setelah restart dan ketika jaringan putus.
- **Integritas:** request sync berulang tidak membuat duplikasi.
- **Offline-first:** layar ledger/rekening membaca data lokal.
- **Keamanan:** least privilege, otorisasi server, HTTPS, token aman, log minim.
- **Kinerja:** interaksi umum responsif pada perangkat sasaran, diverifikasi dengan pengujian.
- **Aksesibilitas:** kontras, font scaling, label screen reader, dan target sentuh memadai.
- **Pemeliharaan:** kode, migrasi, test, dan keputusan teknis terdokumentasi.
- **Portabilitas:** data dapat diekspor ke format terdokumentasi.
- **Privasi:** jelaskan pemrosesan notifikasi, pengiriman data, retensi dan penghapusan.

## 8. Definisi MVP

MVP wajib mencakup login terverifikasi, setup rekening/saldo awal, ledger lokal, pemasukan/pengeluaran/transfer/adjustment, kategori, dashboard dan daftar transaksi, draft notifikasi dengan review manual, backend MySQL dengan kontrol akses pengguna, sinkronisasi idempoten, serta keamanan dasar termasuk app lock dan keputusan enkripsi database yang diuji.

Target rilis awal mencakup seluruh fitur pilihan pengguna: ledger dan capture notifikasi yang wajib ditinjau, banyak rekening dan saldo awal, transfer, rekonsiliasi, offline-first/sinkronisasi, budget kategori, target menabung, kalender transaksi, perbandingan bulanan, ekspor CSV/Excel, backup/pemulihan, serta PIN/biometrik. Implementasi dilakukan bertahap: fitur P0 (ledger, capture, auth, sync, keamanan dasar) dikerjakan lebih dulu; budget, target, kalender, perbandingan, ekspor, dan pemulihan (P1) menyusul setelah fondasi terbukti stabil, tetapi tetap termasuk target rilis awal. Fitur P2 tidak boleh menghambat target rilis awal dan diputuskan secara terpisah.

## 9. Kriteria keberhasilan

- Transaksi terkonfirmasi bertahan di perangkat setelah restart dan saat offline.
- Retry request tidak menghasilkan transaksi/transfer duplikat.
- Tidak ada draft notifikasi yang diposting otomatis.
- Saldo hitungan cocok dengan hasil ledger pada test.
- Akses lintas pengguna ditolak oleh test.
- Transfer tidak dihitung sebagai income/expense.
- Ekspor bisa dibuka dan tidak merusak nilai.

## 10. Risiko

| Risiko | Mitigasi |
|---|---|
| Format notifikasi berubah | Parser per sumber, fixture test, draft untuk ketidakpastian |
| Pembatasan background Android/OEM | Status izin, uji perangkat nyata, tidak menjanjikan capture sempurna |
| Sync duplikat | ID stabil, idempotency key, test replay |
| Saldo menyimpang | Ledger sebagai sumber perhitungan, rekonsiliasi eksplisit |
| Data finansial bocor | Enkripsi lokal yang diuji, HTTPS, otorisasi, log redacted |
| Scope terlalu luas | Prioritaskan P0 sebelum P1/P2 |

## 11. Keputusan yang perlu divalidasi saat implementasi

- Patch Spring Boot 3.5.x yang digunakan dan dukungan saat scaffolding.
- Kompatibilitas API 26 dengan seluruh library inti.
- Library enkripsi Room/SQLite, pengelolaan kunci, migrasi dan pemulihan bila kunci hilang.
- Aplikasi sumber notifikasi pertama.
- Kebijakan data lokal sesudah sign-out dan detail backup/pemulihan.
