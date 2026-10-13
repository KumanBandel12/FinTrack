# TODO.md — Rencana Implementasi FinTrack

Gunakan file ini sebagai sumber status. Checklist hanya dicentang setelah acceptance criteria diuji. Pola kerja default: supervised — satu task, test dan laporan, lalu berhenti untuk instruksi berikutnya.

Status: `[ ]` belum dikerjakan · `[x]` selesai dan terverifikasi · `[!]` terblokir/perlu keputusan.

## Milestone 0 — Konfirmasi fondasi

- [x] **M0.1 Review dokumentasi**
  - Acceptance: PRD, Architecture, Workflow, StyleGuide, dan TODO konsisten dengan keputusan pengguna.
  - Verifikasi: review stack, bahasa, MVP, model saldo, sync, security, dan gaya UI.
  - Selesai: 2026-10-13. Semua dokumen konsisten. 6 keputusan di-defer (adjustment direction, library enkripsi, patch Spring Boot, sumber notifikasi pertama, kebijakan sign-out, retensi backup) sudah terdaftar di TODO/PRD §11 dan dijadwalkan di milestone terkait. Lihat laporan `M0.1_review_dokumentasi.md`.
- [x] **M0.2 Buat repository Git**
  - Acceptance: repository dibuat, branch default jelas, file sensitif/build di-ignore, tujuh dokumen awal masuk Git.
  - Verifikasi: `git status`, inspeksi `.gitignore`, review diff.
  - Selesai: 2026-10-13. Repository ada di GitHub (KumanBandel12/FinTrack), branch default `main`. Ketujuh dokumen ter-track. `.gitignore` diperluas: perbaiki bug `local.properties/` → `local.properties`, tambah `google-services.json`, Spring Boot config lokal (`application-*.yml/yaml/properties`), format sertifikat (`.p12`, `.pfx`, `.pem`, `.key`), VS Code, dan macOS. Semua 10 pola kritis diverifikasi dengan `git check-ignore`. Ketujuh dokumen tidak ter-ignore. Commit: `docs(gitignore): perbaiki dan perluas aturan ignore M0.2`.
- [x] **M0.3 Buat template dan ADR awal**
  - Acceptance: keputusan offline-first, draft notifikasi, transfer, JWT, enkripsi, sync conflict, development lokal, dan visual tercatat.
  - Verifikasi: setiap ADR memuat konteks, opsi, keputusan, konsekuensi, status.
  - Selesai: 2026-10-13. Template resmi (`docs/adr/template.md`), indeks ADR (`docs/adr/README.md`), dan 9 ADR awal (ADR-001 s.d. ADR-009) selesai dibuat. Mencakup offline-first & outbox (ADR-001), draft notifikasi wajib review (ADR-002), transfer dua efek ledger (ADR-003), Google verification & siklus JWT (ADR-004), enkripsi lokal & key lifecycle (ADR-005), idempotensi & konflik sync (ADR-006), backend lokal langsung di laptop (ADR-007), allowlist notifikasi & parser ber-versi (ADR-008), serta visual neobrutalism & font Sora (ADR-009). Seluruh ADR terverifikasi memuat kelima seksi wajib. Tautan ADR diperbarui di `Architecture.md` §14.
- [ ] **M0.4 Validasi versi dan kompatibilitas**
  - Acceptance: patch Spring Boot 3.5.x dan dependency dipilih; kompatibilitas Android API 26 diuji; library enkripsi serta sumber notifikasi pertama ditetapkan.
  - Verifikasi: catat versi, dukungan, batasan, dan hasil proof of concept. Jangan menandai selesai hanya karena dependency ditambahkan.
- [ ] **M0.5 Dokumentasikan cara kerja lokal**
  - Acceptance: prasyarat dan instruksi start/stop backend, MySQL, serta Android terdokumentasi; secret memakai konfigurasi lokal yang tidak di-commit.
  - Verifikasi: ikuti instruksi pada checkout bersih bila memungkinkan.

## Milestone 1 — Scaffold Android dan backend

- [ ] **M1.1 Scaffold Android**
  - Acceptance: Kotlin + Compose dapat dibangun dan menampilkan layar awal; tema terpusat tersedia.
  - Verifikasi: `./gradlew assembleDebug`, `./gradlew test`, `./gradlew lint`.
- [ ] **M1.2 Validasi minimum API 26**
  - Acceptance: `minSdk` adalah 26 dan dependency inti kompatibel atau risiko yang ditemukan disetujui.
  - Verifikasi: build dan uji startup emulator API 26 jika tersedia.
- [ ] **M1.3 Scaffold backend Java 21/Spring Boot 3.5.x**
  - Acceptance: Maven Wrapper, endpoint health non-sensitif, konfigurasi terpisah, dan test dasar tersedia.
  - Verifikasi: `./mvnw test`, `./mvnw package`.
- [ ] **M1.4 Hubungkan MySQL lokal langsung di laptop**
  - Acceptance: aplikasi backend terhubung ke DB lokal; password tidak hardcoded; cara start/stop dicatat.
  - Verifikasi: startup dan koneksi DB berhasil.
- [ ] **M1.5 Pasang Flyway dan format error API**
  - Acceptance: migrasi berjalan pada DB kosong; error validasi memiliki format konsisten tanpa stack trace.
  - Verifikasi: migration test dan API test.

## Milestone 2 — Identitas dan keamanan dasar

- [ ] **M2.1 Verifikasi Google Sign-In**
  - Acceptance: backend memeriksa signature, issuer, audience, expiry, dan klaim wajib.
  - Verifikasi: token valid, invalid, expired, dan audience tidak sesuai diuji.
- [ ] **M2.2 Tetapkan lifecycle JWT**
  - Acceptance: expiry, refresh/revocation, rotasi kunci, issuer, dan audience didokumentasikan.
  - Verifikasi: test token kedaluwarsa/tidak valid.
- [ ] **M2.3 Penyimpanan token Android**
  - Acceptance: token dilindungi penyimpanan yang memanfaatkan Android Keystore dan tidak dicatat ke log.
  - Verifikasi: code review dan uji logout.
- [ ] **M2.4 PIN/biometric lock**
  - Acceptance: layar privat terkunci sesuai pilihan; tersedia fallback jika biometrik tidak tersedia.
  - Verifikasi: uji perangkat sasaran dan test state.
- [ ] **M2.5 Proof of concept enkripsi Room**
  - Acceptance: solusi kompatibel dengan Room/API 26; kebijakan kunci, kehilangan kunci, migrasi, backup, dan pemulihan dijelaskan.
  - Verifikasi: buka DB dengan/tanpa kunci, restart, dan migrasi diuji.
  - Catatan: belum selesai sampai benar-benar diuji.

## Milestone 3 — Ledger lokal

- [ ] **M3.1 Skema Room**
  - Acceptance: account, category, transaction, transfer, draft, dan outbox mempunyai ID stabil serta constraint yang sesuai.
  - Verifikasi: DAO dan migration test.
- [ ] **M3.2 Setup rekening dan saldo awal**
  - Acceptance: pengguna wajib membuat rekening serta memasukkan saldo awal dan tanggal sebelum dashboard penuh.
  - Verifikasi: alur UI dan validasi.
- [ ] **M3.3 Pemasukan/pengeluaran**
  - Acceptance: tambah, edit, dan void menghasilkan saldo benar tanpa floating point.
  - Verifikasi: test nominal batas, kategori, tanggal, serta void.
- [ ] **M3.4 Transfer antar rekening**
  - Acceptance: satu transfer logis menghasilkan dua efek ledger; transfer tidak dihitung sebagai income/expense; fee terpisah.
  - Verifikasi: saldo kedua rekening, rekening sama, edit, void, dan fee diuji.
- [ ] **M3.5 Rekonsiliasi**
  - Acceptance: selisih saldo membuat adjustment dengan tanggal dan alasan; histori lama tidak ditimpa.
  - Verifikasi: saldo sebelum/sesudah dan histori.
- [ ] **M3.6 Kategori**
  - Acceptance: kategori bawaan tersedia; pengguna bisa membuat, mengubah, dan mengarsipkan; histori tetap valid.
  - Verifikasi: test arsip dan ringkasan historis.

## Milestone 4 — UI inti dan sistem desain

- [ ] **M4.1 Token visual Neobrutalism**
  - Acceptance: warna semantic, border, shadow offset, radius, spacing, state dan theme light/dark terpusat.
  - Verifikasi: inspeksi dashboard/form/list di kedua tema dan cek kontras.
- [ ] **M4.2 Font Sora**
  - Acceptance: Sora tampil konsisten, fallback tersedia, ukuran file/lisensi dipertimbangkan, font scaling tetap berfungsi.
  - Verifikasi: judul, angka saldo, form, teks panjang, fallback.
- [ ] **M4.3 Navigasi dan komponen dasar**
  - Acceptance: komponen konsisten dengan state loading/empty/error/offline; opsi tema mengikuti sistem/terang/gelap.
  - Verifikasi: layar kecil dan ukuran teks besar.
- [ ] **M4.4 Dashboard**
  - Acceptance: UI membaca Room, menampilkan saldo dan ringkasan benar, update lokal tanpa menunggu server.
  - Verifikasi: uji offline.
- [ ] **M4.5 Daftar/detail/input transaksi**
  - Acceptance: form tervalidasi; filter, edit/void, nominal IDR, dan tanggal konsisten.
  - Verifikasi: ViewModel/UI test dan QA manual.
- [ ] **M4.6 Layar rekening**
  - Acceptance: rekening, saldo turunan, tanggal saldo awal dan rekonsiliasi mudah ditemukan.
  - Verifikasi: cocokkan layar dengan hasil ledger.

## Milestone 5 — Capture notifikasi dan review

- [ ] **M5.1 Edukasi permission**
  - Acceptance: tujuan akses notifikasi jelas; pengguna dapat menolak/mencabutnya dan tetap memakai input manual.
  - Verifikasi: onboarding/permission test.
- [ ] **M5.2 Listener dan allowlist**
  - Acceptance: hanya sumber allowlist diproses; notifikasi lain diabaikan; callback ringan.
  - Verifikasi: event sintetis dan uji perangkat nyata.
- [ ] **M5.3 Parser sumber pertama**
  - Acceptance: fixture mencakup format valid, malformed, separator nominal lokal, nilai hilang, dan format berubah.
  - Verifikasi: unit test parser; tidak ada notifikasi pribadi di repository.
- [ ] **M5.4 Draft dan deduplikasi**
  - Acceptance: draft disimpan lokal dan dapat diedit/confirm/reject; event duplikat tidak menghasilkan draft berulang.
  - Verifikasi: uji duplikat, restart, dan transisi status.
- [ ] **M5.5 Status capture**
  - Acceptance: permission/listener bermasalah terlihat; aplikasi tidak mengklaim capture selalu aktif.
  - Verifikasi: cabut izin lalu pulihkan.
- [ ] **M5.6 Transfer satu sisi**
  - Acceptance: notifikasi satu sisi hanya membentuk draft dan tidak mem-posting transfer lengkap.
  - Verifikasi: test matching dan false positive.

## Milestone 6 — API dan persistensi server

- [ ] **M6.1 Entity dan migrasi dengan ownership**
  - Acceptance: semua data privat terikat ke user atau relasi kepemilikan yang jelas.
  - Verifikasi: schema review dan migration test.
- [ ] **M6.2 Endpoint rekening dan kategori**
  - Acceptance: list/create/update tervalidasi dan ownership ditegakkan.
  - Verifikasi: API test normal, tanpa auth, dan cross-user.
- [ ] **M6.3 Endpoint transaksi dan transfer**
  - Acceptance: aturan server konsisten dengan Android; transfer atomik.
  - Verifikasi: integration test dan OpenAPI.
- [ ] **M6.4 Sync push/pull**
  - Acceptance: stable ID, idempotency key, cursor/version, dan konflik didefinisikan.
  - Verifikasi: replay, pagination, retry, konflik.
- [ ] **M6.5 Dokumentasi API**
  - Acceptance: auth, DTO, nominal, timestamp, error, filter dan pagination terdokumentasi.
  - Verifikasi: periksa OpenAPI lokal.

## Milestone 7 — Sinkronisasi offline-first

- [ ] **M7.1 Tulis ledger dan outbox secara atomik**
  - Acceptance: perubahan domain dan operasi sync tersimpan sebagai satu transaksi lokal.
  - Verifikasi: database transaction test.
- [ ] **M7.2 WorkManager dan retry**
  - Acceptance: sync dimulai saat kondisi sesuai, retry terbatas/backoff, error dilaporkan.
  - Verifikasi: airplane mode, server unavailable, token expired.
- [ ] **M7.3 Idempotensi dan konflik**
  - Acceptance: replay tidak menggandakan data; versi lama tidak menimpa versi baru secara diam-diam.
  - Verifikasi: replay request dan concurrent-version test.
- [ ] **M7.4 Pertahankan data lokal setelah sync**
  - Acceptance: hanya outbox yang ditandai selesai; transaksi tetap ada di Room.
  - Verifikasi: test otomatis dan inspeksi database uji.
- [ ] **M7.5 Pemulihan setelah process death**
  - Acceptance: antrean pending dilanjutkan setelah aplikasi dibuka ulang.
  - Verifikasi: hentikan proses saat sync tertunda dan periksa pemulihan.

## Milestone 8 — Analitik dan ekspor

- [ ] **M8.1 Ringkasan income/expense**
  - Acceptance: transfer/void dikecualikan dan batas periode memakai zona waktu yang benar.
  - Verifikasi: fixture batas bulan.
- [ ] **M8.2 Budget kategori**
  - Acceptance: target periode dan progres dihitung konsisten.
  - Verifikasi: nilai melebihi budget dan pergantian periode.
- [ ] **M8.3 Target menabung**
  - Acceptance: target/progres tidak disamakan dengan saldo rekening.
  - Verifikasi: test progres dan perubahan target.
- [ ] **M8.4 Kalender/perbandingan bulanan**
  - Acceptance: tanggal lokal dan periode pembanding konsisten.
  - Verifikasi: transaksi pada batas bulan.
- [ ] **M8.5 Ekspor CSV**
  - Acceptance: kolom/encoding didokumentasikan, berbagi file eksplisit, CSV formula injection dimitigasi.
  - Verifikasi: buka di spreadsheet dan uji field berawalan karakter formula.
- [ ] **M8.6 Excel dan backup/pemulihan**
  - Acceptance: format, enkripsi, retensi dan pemulihan dijelaskan.
  - Verifikasi: export/restore dengan data sintetis.

## Milestone 9 — Security review dan acceptance test

- [ ] **M9.1 Review otorisasi server**
  - Acceptance: semua endpoint privat diuji untuk request tanpa auth dan cross-user.
  - Verifikasi: security test suite.
- [ ] **M9.2 Review secret dan logging**
  - Acceptance: tidak ada secret, raw notification, atau data finansial tak semestinya di source/log.
  - Verifikasi: pencarian terarah dan code review.
- [ ] **M9.3 Uji backup/restore**
  - Acceptance: prosedur pemulihan terbukti dan data yang tertimpa diketahui.
  - Verifikasi: recovery drill menggunakan data sintetis.
- [ ] **M9.4 Review permission dan aliran data**
  - Acceptance: akses notifikasi bersifat opt-in dan terbatasi sumber.
  - Verifikasi: cabut permission dan review privasi.
- [ ] **M9.5 End-to-end MVP**
  - Acceptance: setup → input manual → draft notifikasi → transfer → offline → sync → budget/target menabung → kalender/perbandingan → ekspor → backup/pemulihan berjalan untuk target rilis awal.
  - Verifikasi: checklist QA lengkap beserta bukti.

## Milestone 10 — Fitur P2 lanjutan dan VPS

- [ ] **M10.1 Pilih fitur P2 setelah MVP stabil**
  - Acceptance: langganan, alert pengeluaran tidak biasa, forecasting, import, atau kueri bahasa alami dipilih satu per satu dengan acceptance criteria dan evaluasi privasi.
  - Verifikasi: PRD/keputusan diperbarui.
- [ ] **M10.2 Rencana VPS dan biaya**
  - Acceptance: provider, biaya berulang, akses, lokasi, dan backup disetujui.
  - Verifikasi: keputusan tertulis sebelum membeli.
- [ ] **M10.3 Konfigurasi produksi**
  - Acceptance: secret eksternal, HTTPS valid, database tidak terbuka publik, profil produksi terpisah.
  - Verifikasi: review konfigurasi/TLS/port.
- [ ] **M10.4 Backup otomatis dan restore drill**
  - Acceptance: backup terjadwal dan restore teruji.
  - Verifikasi: pemulihan di lingkungan terisolasi.
- [ ] **M10.5 Deploy backend dan APK rilis**
  - Acceptance: auth, ownership, sync, log, backup dan rollback diverifikasi; signing key aman.
  - Verifikasi: smoke test menggunakan akun/data sintetis.

## Daftar fitur P2 — tidak boleh menghambat target rilis awal

- [ ] Deteksi langganan/pembayaran berulang.
- [ ] Peringatan pengeluaran tidak biasa.
- [ ] Peramalan pengeluaran.
- [ ] Impor transaksi dari file.
- [ ] Kueri bahasa alami dengan kontrol privasi.
