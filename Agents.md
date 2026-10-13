# Agents.md — Aturan AI Coding Agent

Dokumen ini berlaku untuk agen coding yang digunakan melalui IDE, CLI, atau pola multi-agent. Kode yang dihasilkan agen bukan bukti bahwa fitur sudah benar; hasil wajib diuji.

## 1. Dokumen yang wajib dibaca

Sebelum mengubah kode, baca `PRD.md`, `Architecture.md`, `Workflow.md`, `StyleGuide.md`, `TODO.md`, dan prompt yang relevan di `Skill.md`. Jika ada konflik kebutuhan atau keputusan arsitektur, laporkan dan berhenti sebelum menebak kebijakan produk/keamanan.

## 2. Perintah build dan verifikasi

Pastikan wrapper benar-benar tersedia sebelum menjalankan perintah. Ini adalah target perintah setelah scaffold, bukan klaim bahwa proyek telah dibangun.

### Android
- Build debug: `./gradlew assembleDebug`
- Unit test: `./gradlew test`
- Lint: `./gradlew lint`
- Instrumented test: `./gradlew connectedAndroidTest`
- Pada Windows gunakan `gradlew.bat`.

### Backend
- Test: `./mvnw test`
- Build paket: `./mvnw package`
- Periksa endpoint dan dokumentasi OpenAPI saat server lokal berjalan.

### Database
- Gunakan Flyway untuk migrasi.
- Jangan ubah migrasi yang sudah diterapkan pada lingkungan bersama; buat migrasi baru.
- Jangan reset database tanpa memahami data yang terdampak dan memperoleh persetujuan.

## 3. Konvensi umum

- Kerjakan satu task TODO yang terukur.
- Buat perubahan terkecil yang memenuhi acceptance criteria.
- Jangan menambah abstraksi/dependency tanpa kebutuhan nyata.
- Jangan gunakan floating point untuk nominal uang.
- Validasi data dari API, file impor, dan notifikasi.
- Tetapkan kebijakan waktu/tanggal secara konsisten; bedakan waktu sinkronisasi dan tanggal transaksi lokal.
- Jangan mencetak token, isi notifikasi mentah, rekening, atau rincian transaksi ke log.
- Komentar menerangkan alasan, invariant, atau workaround yang tidak jelas.

## 4. Android/Kotlin

- Gunakan Kotlin, Jetpack Compose, Material 3, ViewModel, coroutines/Flow, Room, DataStore, WorkManager, Retrofit/OkHttp, Google Sign-In yang didukung, Android Keystore, dan BiometricPrompt bila sesuai.
- Minimum API adalah 26 kecuali ditemukan incompatibility nyata yang dilaporkan untuk keputusan pengguna.
- UI mengamati Room melalui repository; composable tidak memiliki SQL atau aturan bisnis.
- Parser notifikasi dibuat teruji melalui fixture bila memungkinkan.
- `NotificationListenerService` harus ringan; bukan mekanisme polling.
- Hindari foreground service/wake lock tanpa justifikasi kuat.
- Gunakan string resource untuk teks UI dan dukung font scaling.
- Uji izin dan perilaku background di perangkat nyata bila relevan.

## 5. Java/Spring Boot

- Baseline: Java 21 + Spring Boot 3.5.x.
- Gunakan Spring Security, Spring Data JPA, Bean Validation, Flyway, REST/JSON, dan Maven Wrapper.
- Controller tipis; aturan saldo, transfer, dan sync berada di service/domain.
- Gunakan DTO pada API; jangan expose entity database langsung.
- Setiap query privat dibatasi principal pengguna terautentikasi.
- Operasi transfer/ledger wajib atomik.
- Jangan mengembalikan stack trace, detail SQL, atau secret kepada klien.
- Dokumentasikan satuan uang, timestamp, pagination, idempotency, dan error.

## 6. Penamaan

- Class/interface: `UpperCamelCase`.
- Fungsi/properti: `lowerCamelCase`.
- Konstanta: `UPPER_SNAKE_CASE` jika sesuai.
- Database: `snake_case` dengan konvensi singular/plural konsisten.
- Branch: `feat/...`, `fix/...`, `test/...`, atau `docs/...`.
- Commit: `type(scope): ringkasan`, contoh `feat(ledger): tambah transfer antar rekening`.

## 7. Alur kerja per task

1. Cari item TODO dan acceptance criteria.
2. Periksa kode dan test yang tersedia.
3. Sampaikan rencana singkat, file terdampak, risiko, dan test.
4. Tambahkan/perbarui test.
5. Implementasikan perubahan.
6. Jalankan test relevan dan pemeriksaan lebih luas jika memungkinkan.
7. Periksa diff untuk regresi, secret, penghapusan data, dan scope creep.
8. Update TODO hanya jika kriterianya terverifikasi.
9. Laporkan file berubah, perintah dan hasil aktual, test yang belum dijalankan, risiko, dan blocker.

## 8. Wajib minta persetujuan

Minta persetujuan sebelum: menghapus/menimpa data atau reset DB; migrasi destruktif; perubahan model saldo/transfer/auth/enkripsi/sync conflict; menambah biaya atau layanan cloud; deploy VPS, membuka port publik, mengubah DNS/firewall; mengirim data finansial ke layanan eksternal; menambah telemetry yang mengirim data pengguna; mengubah strategi kredensial atau menurunkan keamanan; mengganti framework/dependency inti; force push pada branch bersama.

Perubahan rutin yang kecil dan reversibel boleh dilakukan dalam task yang disetujui selama tidak mengubah keputusan tersebut.

## 9. Larangan

- Jangan hardcode secret, password, token, signing key, atau credential produksi.
- Jangan log isi notifikasi mentah atau rincian transaksi di produksi.
- Jangan otomatis memposting draft notifikasi.
- Jangan menghapus data transaksi lokal setelah sync berhasil.
- Jangan menghitung transfer sebagai income/expense.
- Jangan percaya `userId` dari payload sebagai dasar otorisasi.
- Jangan menonaktifkan validasi sertifikat TLS di produksi.
- Jangan menjalankan operasi destruktif tanpa persetujuan dan backup terverifikasi.
- Jangan menandai test lulus jika tidak dijalankan.
- Jangan mengirim data pengguna ke layanan AI/eksternal tanpa alur yang telah disetujui.

## 10. Definition of done

Task selesai jika acceptance criteria terpenuhi, test relevan lulus atau kendalanya dilaporkan eksplisit, keamanan dan integritas data tidak mengalami regresi yang diketahui, dokumentasi diperbarui bila perlu, diff diperiksa, dan TODO mencerminkan kondisi sebenarnya. Test wajib yang gagal berarti task belum selesai.
