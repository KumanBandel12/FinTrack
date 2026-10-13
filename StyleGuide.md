# StyleGuide.md — Pedoman UI dan Kode FinTrack

## 1. Arah visual

- **Gaya:** Neobrutalism.
- **Font utama:** Sora dari Google Fonts.
- **Tema:** light dan dark mengikuti sistem, dengan override manual.
- **Kepadatan:** balanced—ringkas tetapi nyaman dibaca.
- **Animasi:** transisi halus dan fungsional.
- **Bahasa UI:** Bahasa Indonesia.

FinTrack boleh mempunyai karakter visual yang tegas, tetapi saldo dan status transaksi harus selalu jelas. Neobrutalism tidak boleh mengorbankan keterbacaan, aksesibilitas, atau pemahaman konsekuensi aksi.

## 2. Prinsip Neobrutalism

- Kontras kuat antara teks dan permukaan.
- Border tegas dan konsisten.
- Shadow keras/offset secara terukur.
- Bentuk sederhana, blok warna jelas, dan hirarki informasi tegas.
- Feedback interaksi nyata pada tombol, kartu, dan kontrol.
- Dekorasi tidak boleh mengganggu pembacaan daftar transaksi.
- Warna aksen digunakan untuk penekanan; tidak semua elemen harus berwarna.

### Token warna awal

Palet berikut adalah **usulan awal** untuk implementasi dan review, bukan palet final yang sudah disahkan. Semua warna harus dikelola lewat token semantic.

| Token | Light mode awal | Kegunaan |
|---|---|---|
| `background` | `#F7F3E8` | Latar netral hangat |
| `surface` | `#FFFFFF` | Permukaan konten |
| `textPrimary` | `#171717` | Teks utama |
| `primary` | `#FFD84D` | CTA/aksen utama |
| `accentBlue` | `#A8D5FF` | Aksen sekunder |
| `accentPink` | `#FFA8C5` | Aksen sekunder |
| `accentGreen` | `#B8F28B` | Aksen/konfirmasi |
| `income` | Hijau dengan kontras memadai | Pemasukan |
| `expense` | Merah dengan kontras memadai | Pengeluaran |
| `outline` | `#171717` | Border utama |
| `shadow` | `#171717` | Shadow offset |

Dark mode harus dirancang sebagai set token tersendiri, bukan sekadar membalik warna. Periksa kontras teks, border, ikon, dan aksen; hindari warna yang menyilaukan. Income/expense tidak boleh dibedakan hanya dengan warna—gunakan juga tanda, teks, atau ikon.

### Border, radius, dan shadow

- Border utama dapat dimulai dari 2 dp lalu disesuaikan menurut densitas layar.
- Gunakan radius kecil sampai sedang secara konsisten.
- Shadow offset dipakai pada CTA atau kartu penting; jangan diberikan ke semua elemen.
- Interaksi pressed dapat mengurangi offset shadow sebagai feedback fisik, tetapi tidak boleh menggeser konten secara tidak sengaja.

## 3. Tipografi Sora

- Gunakan Sora sebagai font utama setelah kompatibilitas, ukuran APK, dan lisensi/ketentuan distribusi diperiksa.
- Untuk APK manual, pilih strategi font yang stabil—misalnya aset font yang dibundel—dengan fallback sistem yang wajar.
- Jangan mengambil font dari URL arbitrer saat runtime.
- Buat hirarki konsisten: judul layar, heading bagian, saldo utama, body, label, dan caption.
- Gunakan angka tabular jika tersedia dan membantu perbandingan nominal.
- Dukung font scaling Android; jangan mengunci semua teks ke ukuran tetap.
- Jangan membuat semua teks bold; manfaatkan ukuran, bobot, kontras, dan ruang.

## 4. Spacing dan layout

- Skala dasar 4 dp: 4, 8, 12, 16, 24, 32 dp.
- Padding layar umumnya 16–24 dp, disesuaikan lebar perangkat.
- Balanced density: hindari kartu berlapis berlebihan, tetapi jangan membuat informasi terlalu padat.
- Hormati system insets, keyboard, gesture navigation, serta ukuran layar berbeda.
- Jangan mengandalkan satu ukuran layar untuk layout final.

## 5. Panduan layar

### Dashboard
- Total saldo terlihat jelas dengan keterangan bahwa nilainya berasal dari catatan FinTrack.
- Ringkasan pemasukan/pengeluaran dipisahkan dari transfer.
- Tampilkan transaksi terbaru dan aksi input yang mudah ditemukan.
- Status sync tidak lebih dominan daripada informasi utama.
- Jika saldo perlu rekonsiliasi, sediakan jalur yang jelas.

### Daftar transaksi
- Minimal: tanggal, deskripsi, rekening, kategori bila tersedia, dan nominal.
- Label income, expense, transfer, dan adjustment konsisten.
- Pengelompokan tanggal mengikuti kalender lokal.
- Filter harus bisa dipahami dan direset.
- Void/hapus menjelaskan pengaruh terhadap saldo.

### Review notifikasi
- Draft harus jelas berbeda dari transaksi posted.
- Tampilkan sumber dan field hasil parsing.
- Field kosong/tidak pasti ditandai; jangan menyembunyikan ketidakpastian.
- Aksi utama adalah tinjau/edit, konfirmasi, dan tolak.
- Tidak boleh ada alur one-tap yang langsung mem-posting notifikasi tanpa review.

### Rekening dan transfer
- Tampilkan nama, tipe, dan saldo turunan.
- Saldo awal serta histori rekonsiliasi dapat ditemukan.
- Transfer menjelaskan rekening sumber, tujuan, nominal, dan fee.
- Jangan menghapus keras rekening berhistori tanpa alur pemindahan data yang disetujui.

### Form dan feedback
- Label bukan sekadar placeholder.
- Error ditampilkan dekat field dan input dipertahankan jika error dapat diperbaiki.
- Bedakan offline, sync pending, retryable failure, konflik, dan auth kedaluwarsa.
- Aksi yang mengubah ledger perlu konfirmasi yang menerangkan konsekuensi.

## 6. Tema light/dark dan override

Pengaturan tema menyediakan `Ikuti sistem`, `Terang`, dan `Gelap`. Default adalah mengikuti sistem. Pilihan manual disimpan sebagai preferensi. Semua layar mengambil warna dari token tema, bukan warna hardcoded. QA harus memeriksa kedua tema, termasuk border/shadow dan state error.

## 7. Animasi

- Transisi singkat untuk navigasi, dialog, expand/collapse, dan feedback aksi.
- Animasi tidak boleh menunda penyimpanan transaksi atau mengaburkan perubahan nominal.
- Hindari animasi dekoratif yang terus bergerak.
- Hormati preferensi aksesibilitas/reduced motion bila tersedia.
- Informasi sync tetap jelas meski animasi tidak berjalan.

## 8. Aksesibilitas dan format lokal

- Target sentuh idealnya minimal 48 dp.
- Dukung font scaling dan screen reader.
- Periksa kontras dan fokus; status tidak hanya dibedakan dengan warna.
- Gunakan Bahasa Indonesia untuk UI dan resource string untuk semua teks.
- Format mata uang mengikuti lokal Indonesia dan menggunakan IDR.
- Format tanggal familier dalam Bahasa Indonesia; waktu disimpan secara konsisten dan dikonversi ke zona waktu tampilan yang benar.

## 9. Gaya kode Android

- Gunakan Material 3 dan token theme terpusat.
- Composable fokus pada rendering dan event; logika domain/database/network berada di layer terpisah.
- Kumpulkan Flow menggunakan API lifecycle-aware.
- Gunakan UI state yang eksplisit dan mudah diuji.
- Pecah composable besar ketika tanggung jawabnya berbeda.
- Sediakan preview untuk state loading, empty, error, teks panjang, light, dan dark jika memungkinkan.

## 10. Gaya backend/API

- Gunakan DTO dan validasi di batas API.
- Error response stabil, tanpa stack trace/SQL internal.
- Dokumentasikan satuan uang, timestamp, pagination, idempotency, dan status sync.
- Query privat wajib menerapkan ownership.
- Controller tipis dan aturan bisnis berada di service/domain.
- Jangan mengekspos field entity yang tidak diperlukan.

## 11. Checklist review desain

- [ ] Hierarki informasi utama mudah dipahami.
- [ ] State loading, empty, error, offline, dan sync ditangani.
- [ ] Nominal presisi dan format uang konsisten.
- [ ] Transfer tidak dihitung sebagai income/expense.
- [ ] Neobrutalism konsisten dan tidak berlebihan.
- [ ] Sora tampil benar dengan fallback dan font scaling.
- [ ] Light/dark serta override tema berfungsi.
- [ ] Aksi berisiko menerangkan konsekuensinya.
- [ ] Kontras, screen reader, fokus, dan target sentuh diperiksa.
- [ ] Animasi tidak mengganggu fungsi atau keterbacaan saldo.
