# Workflow.md — Alur Kerja Pengembangan FinTrack

## 1. Tujuan

Dokumen ini mengatur cara developer dan AI coding agent bekerja bertahap, dapat diuji, dan aman untuk data finansial. Pola default adalah **supervised**: satu task → test dan laporan → menunggu instruksi berikutnya.

## 2. Protokol awal sesi

1. Periksa `git status`, branch aktif, dan commit terbaru.
2. Baca `Agents.md` serta bagian PRD/Architecture yang relevan.
3. Baca `TODO.md`, pilih satu task belum selesai dengan acceptance criteria jelas.
4. Periksa implementasi/test yang sudah ada.
5. Sampaikan rencana, file yang mungkin berubah, risiko, dan test yang akan dijalankan.
6. Minta keputusan jika task membutuhkan kebijakan produk/keamanan yang belum disetujui atau dokumen saling bertentangan.

Jangan mengulang task yang telah selesai hanya karena sesi baru dimulai. Verifikasi status repository terlebih dahulu.

## 3. Otonomi dan batas task

Selesaikan satu task dengan lingkup jelas lalu berhenti untuk ditinjau. Subtask kecil boleh digabung jika tidak masuk akal dipisahkan, tetapi jangan berpindah milestone tanpa penjelasan.

Agen boleh mengambil keputusan teknis kecil yang reversibel di dalam task yang disetujui. Agen wajib berhenti dan meminta persetujuan sebelum:
- mengubah semantik saldo, transfer, rekonsiliasi, atau sinkronisasi;
- mengubah autentikasi, enkripsi, key lifecycle, atau privasi;
- menghapus data, reset database, atau migrasi destruktif;
- menambah layanan berbayar, biaya berulang, atau resource cloud;
- deploy VPS, membuka port publik, atau mengubah DNS/firewall;
- mengirim data finansial ke layanan eksternal;
- mengganti framework/dependency inti.

## 4. Siklus implementasi

```text
Baca requirement
    ↓
Periksa implementasi dan test
    ↓
Pilih satu item TODO
    ↓
Rencana + risiko
    ↓
Tambah/perbarui test
    ↓
Implementasi minimum
    ↓
Jalankan test dan pemeriksaan
    ↓
Tinjau diff, secret, dan regresi
    ↓
Update dokumentasi/TODO berdasarkan bukti
    ↓
Laporkan hasil, lalu berhenti untuk review
```

## 5. Git workflow

1. Buat branch fitur/perbaikan dari branch dasar yang disepakati.
2. Batasi branch pada satu task atau perubahan yang saling terkait.
3. Jalankan test relevan sebelum commit.
4. Periksa `git diff` dan `git status`.
5. Gunakan commit terstruktur, misalnya:
   - `feat(android): tambah setup rekening`
   - `feat(api): sinkronisasi transaksi idempoten`
   - `fix(ledger): perbaiki biaya transfer`
   - `test(sync): uji retry setelah timeout`
   - `docs: jelaskan review draft notifikasi`
6. Merge ke `main` setelah test lulus dan diff ditinjau.
7. Jangan force-push branch bersama atau commit secret.

Untuk proyek pribadi tanpa Pull Request, review diff di branch tetap wajib. Commit langsung ke `main` bukan pola default.

## 6. Kebijakan pengujian

- Perubahan logika, persistence, API, parser, keamanan, uang, atau sync wajib memiliki test.
- Perubahan UI harus diuji pada state/interaksi relevan dan ditinjau manual untuk aspek visual/aksesibilitas.
- Jalankan test paling sempit dahulu; lanjutkan ke test modul, build dan lint sebelum milestone ditutup.
- Jika test tidak dapat dijalankan karena device/service/environment tidak tersedia, laporkan sebagai belum dijalankan/terblokir, bukan lulus.
- Acceptance criteria tidak boleh dicentang jika test wajib gagal.
- Gunakan data sintetis; jangan masukkan notifikasi asli, token, atau data rekening ke fixture.

## 7. Aturan keamanan data dan migrasi

- Record domain lokal tetap ada setelah sync sukses.
- Status outbox dipisahkan dari ledger transaksi.
- Jangan reset database tanpa persetujuan dan backup yang telah diverifikasi.
- Migrasi yang diterapkan di environment bersama/deployment tidak diedit ulang; tambahkan migrasi baru.
- Gunakan database test dan akun sintetis untuk integrasi.
- Sebelum restore, tentukan data apa yang akan ditimpa.

## 8. Pengembangan lokal

- Jalankan backend Spring Boot dan MySQL langsung di laptop pada tahap pertama.
- Simpan credential lokal dalam konfigurasi yang di-ignore Git.
- Gunakan data uji sintetis.
- Bila emulator/perangkat membutuhkan alamat host tertentu, dokumentasikan konfigurasi tersebut.
- Konfigurasi HTTP yang dibutuhkan untuk pengembangan lokal tidak boleh masuk build release.
- Uji satu alur end-to-end sebelum menambah banyak fitur.

## 9. Gate sebelum VPS

Deployment harus mendapat persetujuan eksplisit. Sebelum deploy:
- biaya VPS dan biaya berulang disepakati;
- secret tidak ada di Git;
- HTTPS valid;
- database tidak terbuka ke internet;
- autentikasi dan akses silang pengguna telah diuji;
- backup otomatis dan restore drill tersedia;
- log tidak berisi data sensitif;
- smoke test memakai akun/data sintetis;
- langkah rollback terdokumentasi.

## 10. Handoff akhir sesi

Laporan akhir wajib berisi: ID TODO dan status, file berubah, perintah test dan hasil aktual, test belum dijalankan beserta alasannya, risiko/blocker/keputusan, status branch/commit, dan satu task terbaik selanjutnya.

Update `TODO.md` hanya untuk progres terverifikasi. Pastikan sesi berikutnya dapat melanjutkan tanpa menebak.
