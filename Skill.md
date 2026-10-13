# Skill.md — Pola Prompt untuk Agen Coding FinTrack

Gunakan template berikut dengan mengganti teks dalam kurung siku. Untuk setiap task, agen tetap harus mengikuti `Agents.md`, `PRD.md`, `Architecture.md`, `Workflow.md`, dan `TODO.md`.

## 1. Implementasi task umum

> Baca dokumen proyek dan implementasikan hanya **[ID/judul TODO]**. Periksa kode dan test sebelum mengubah apa pun. Sampaikan rencana, file terdampak, risiko, dan test yang akan dilakukan. Terapkan perubahan terkecil yang memenuhi acceptance criteria. Tambah/perbarui test, jalankan perintah relevan, dan laporkan hasil aktual. Jangan mengklaim test lulus bila tidak dijalankan. Jangan mengubah fitur lain. Minta persetujuan sebelum tindakan destruktif, perubahan keamanan, biaya baru, deployment, atau perubahan lingkup.

## 2. Testing dan debugging

> Investigasi **[perilaku/kegagalan]** dengan reproduksi paling kecil. Identifikasi kasus normal, batas, input invalid, dan failure. Tentukan akar masalah sebelum mengedit. Jangan menghapus test, mematikan lint, atau menyembunyikan error. Buat perbaikan minimum, jalankan kembali test awal dan regresi, lalu laporkan perintah dan hasil persis serta test yang tidak dijalankan.

## 3. Security review

> Audit **[modul/API/alur]** dengan menelusuri data sensitif dari input ke database, log, jaringan, backup, dan penghapusan. Periksa autentikasi, otorisasi ownership, validasi, secret, enkripsi, replay/idempotency, dan error response. Jangan mengubah kode dahulu. Laporkan temuan Critical/High/Medium/Low dengan lokasi file/baris, skenario dampak, langkah reproduksi aman, dan mitigasi minimum. Jangan memasukkan token atau data finansial asli dalam laporan.

## 4. Refactoring

> Refactor **[komponen]** agar lebih mudah dibaca/diuji tanpa mengubah perilaku atau kontrak publik. Catat perilaku dan test yang ada. Hindari dependency baru. Tambahkan characterization test jika dibutuhkan. Setelah refactor, jalankan test terkait dan periksa diff untuk perubahan perilaku tak sengaja. Jika semantik tidak jelas, berhenti dan minta keputusan.

## 5. Code review

> Tinjau diff sebagai reviewer senior. Prioritaskan kerusakan data, nominal/saldo/transfer, akses lintas pengguna, auth/enkripsi, race condition, sync offline, lifecycle Android, serta test yang hilang. Jangan menulis ulang kode. Laporkan temuan actionable dengan severity, lokasi, skenario, dan usulan perbaikan. Jika tidak ada temuan, jelaskan ruang lingkup pemeriksaan dan hal yang belum diverifikasi.

## 6. Database migration dan integrity review

> Review **[entity/migration]** terkait constraint, indeks, nullability, presisi uang, ownership, keamanan migrasi, dan pemulihan. Jangan reset database atau menjalankan migrasi destruktif. Usulkan migrasi forward-only dan test. Minta persetujuan sebelum mengubah migrasi yang sudah diterapkan atau menghapus data.

## 7. Android notification dan background behavior

> Implementasikan/review **[parser/provider/listener]** memakai fixture sintetis/teranonimkan. Uji format valid, malformed, separator nominal lokal, tanggal/nilai hilang, mata uang ambigu, event duplikat, format berubah, dan sumber tak didukung. Parser tidak boleh menebak nilai. Hasil selalu draft untuk review, bukan transaksi otomatis. Periksa permission, lifecycle, callback, API 26+, dan pembatasan background. Jangan menganggap emulator mencerminkan semua OEM. Jangan log atau commit isi notifikasi asli.

## 8. Offline synchronization review

> Telusuri alur Room → outbox → request → persistensi server → response → status sync lokal. Uji offline, timeout setelah server commit, request duplikat, retry, token kedaluwarsa, validation error, partial failure, dan konflik versi. Pastikan sync sukses tidak menghapus record transaksi lokal dan versi baru tidak ditimpa diam-diam.

## 9. UI consistency review

> Tinjau **[layar/fitur]** terhadap `StyleGuide.md`: Neobrutalism, tema light/dark, Sora, spacing, border/shadow, state, animasi, dan aksesibilitas. Periksa loading/empty/error/offline, font scaling, kontras, target sentuh, dan keterbacaan angka. Laporkan ketidakkonsistenan sebelum mengubah banyak komponen. Gunakan data sintetis.

## 10. Implementasi UI

> Implementasikan **[layar/komponen]** sesuai PRD dan StyleGuide. Pisahkan logika domain/network dari composable. Tangani state loading, kosong, error, dan offline bila relevan. Pastikan nominal presisi, teks UI memakai resource, dan animasi tidak menghalangi tugas. Jalankan test sesuai dan laporkan kebutuhan QA manual.

## 11. Perancangan API

> Usulkan kontrak **[use case]** sebelum implementasi. Cantumkan method/path, autentikasi, DTO, validasi, status/error code, pagination, idempotency, dan ownership. Jangan gunakan `userId` dari payload sebagai otorisasi. Pastikan cocok dengan API/OpenAPI yang ada. Minta keputusan bila kontrak mengubah perilaku terdahulu.

## 12. Evaluasi dependency

> Evaluasi **[library]** untuk **[kebutuhan]**. Periksa pemeliharaan, kompatibilitas Android API 26/JDK 21/Spring Boot 3.5.x, lisensi, dependency transitif, riwayat keamanan, batas platform, dan biaya migrasi. Jangan tambahkan dependency dulu. Beri rekomendasi add/avoid/defer beserta alternatif minimum. Layanan berbayar memerlukan persetujuan.

## 13. Debugging build

> Investigasi **[output error persis]**. Reproduksi dengan perintah paling sempit, cari akar masalah dari kode/config/perubahan terkait, dan jangan menaikkan dependency yang tidak terkait sebagai jalan pintas. Terapkan fix minimal, jalankan ulang perintah yang gagal dan test regresi, kemudian laporkan hasil aktual.

## 14. Release readiness

> Nilai kesiapan **[milestone/build]** terhadap PRD dan TODO. Periksa test, otorisasi, HTTPS, secret, migrasi/backup, idempotensi, izin notifikasi, retensi data lokal, dan rollback. Berikan keputusan go/no-go berbasis bukti. Jangan deploy atau menimbulkan biaya tanpa persetujuan eksplisit.

## 15. Handoff antarsesi

> Sebelum mengakhiri sesi, update TODO hanya untuk kemajuan yang terverifikasi. Berikan milestone aktif, TODO selesai beserta bukti, test dan hasil, pekerjaan tersisa, blocker/keputusan, file berubah, branch/commit, serta satu rekomendasi task selanjutnya. Jangan menyatakan pekerjaan yang belum diverifikasi sebagai selesai.
