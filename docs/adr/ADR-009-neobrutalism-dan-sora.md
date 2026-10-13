# ADR-009 — Neobrutalism dan Tipografi Sora

**Status:** ACCEPTED  
**Tanggal:** 2026-10-13  
**Pengambil Keputusan:** Tim Pengembang FinTrack  
**Konsultasi / Stakeholder:** Desainer UI/UX & Tim Android Jetpack Compose  

---

## 1. Konteks

Aplikasi pencatatan keuangan sering kali memiliki tampilan yang homogen: didominasi layout korporat bernuansa putih-biru standar atau tampilan minimalis yang datar (flat) sehingga angka saldo dan tombol aksi kurang memiliki hirarki visual yang menonjol.

FinTrack membutuhkan identitas visual yang:
1. **Berkarakter Kuat dan Segar:** Memberikan daya tarik visual yang modern dan unik bagi pengguna generasi muda, tanpa terasa murahan.
2. **Keterbacaan Finansial Maksimal:** Angka nominal saldo, status transaksi (+/-), dan nama rekening harus terbaca sangat jelas dalam berbagai kondisi pencahayaan dan ukuran layar ponsel.
3. **Ketegasan Aksi (Tactile Feedback):** Aksi finansial (seperti konfirmasi draft transaksi, transfer dana, simpan) memerlukan batasan tombol yang tegas dengan feedback visual fisik saat ditekan (tactile press interaction).
4. **Performa Rendering:** Sistem desain harus dapat diimplementasikan secara efisien di Jetpack Compose tanpa menyebabkan drop frame, kompleksitas layout yang berlebihan, atau dependensi unduhan font eksternal yang tidak andal.

---

## 2. Opsi yang Dipertimbangkan

### Opsi 1: Material Design 3 Standar Tanpa Kustomisasi (Vanilla Material You)
- **Deskripsi:** Menggunakan tema bawaan Material 3 dengan warna dinamis (Dynamic Color/Material You) dan elevasi berbasis shadow lembut (soft elevation).
- **Kelebihan:**
  - Kompatibilitas langsung dengan seluruh komponen Jetpack Compose bawaan.
- **Kekurangan:**
  - Tampilan menjadi sangat generik dan kehilangan diferensiasi produk.
  - Elevasi bayangan halus (soft blur) di Compose sering kali kurang memberikan kontras batas yang tajam untuk data tabel finansial.

### Opsi 2: Skewmorphism atau Glassmorphism
- **Deskripsi:** Menggunakan efek kaca buram (blur, backdrop-filter) atau gradien 3D realistis.
- **Kelebihan:**
  - Tampilan terlihat mewah dan futuristik.
- **Kekurangan:**
  - Sangat berat di-render oleh GPU pada perangkat Android berspesifikasi menengah ke bawah (menyebabkan stuttering saat scroll daftar transaksi).
  - Mengurangi keterbacaan angka kontras tinggi di bawah sinar matahari.

### Opsi 3: Neobrutalism Terukur dengan Tipografi Sora Terbundel (Dipilih)
- **Deskripsi:**
  - **Arah Visual Neobrutalism:** Menggabungkan border tegas (default 2 dp), kontras tinggi antara teks dan bidang permukaan, shadow offset keras (hard offset shadow) yang terukur pada kartu utama dan CTA penting, serta blok warna aksen fungsional.
  - **Prinsip Keterbacaan:** Neobrutalism tidak boleh mengorbankan aksesibilitas. Pembedaan transaksi tidak hanya lewat warna (hijau/merah), melainkan wajib disertai tanda (+/-) dan ikon yang jelas.
  - **Tipografi Sora:** Menggunakan jenis huruf *Sora* (geometric sans-serif dari Google Fonts) yang memiliki keterbacaan angka tinggi, sudut lengkung presisi, dan ketersediaan bobot yang bervariasi. Font dibundel sebagai aset lokal aplikasi (res/font) guna menjamin kemandirian offline dan stabilitas rendering tanpa download runtime.
  - **Dark Mode Independen:** Dark mode dirancang sebagai kumpulan token semantik tersendiri dengan kontras yang terkalibrasi, bukan sekadar algoritma inversi warna otomatis.
- **Kelebihan:**
  - Memberikan identitas unik, energik, dan sangat modern bagi FinTrack.
  - Border dan shadow tegas meningkatkan pemahaman hierarki konten dan affordance tombol.
  - Font lokal menjamin rendering seketika saat aplikasi dibuka dalam mode offline.
  - Mendukung penskalaan teks aksesibilitas sistem Android (font scaling).
- **Kekurangan:**
  - Perlu membuat modul token desain khusus di Jetpack Compose (border, shape, color palette, hard shadow modifier).
  - Membutuhkan penambahan ukuran APK sekitar 200–400 KB untuk file font `.ttf` Sora yang dibundel.

---

## 3. Keputusan

FinTrack mengadopsi **Opsi 3: Neobrutalism Terukur dengan Tipografi Sora Terbundel**.

Aturan pelaksanaan sistem visual ini:
1. **Sistem Desain Berbasis Token:**
   - Semua elemen UI Jetpack Compose dilarang memakai nilai warna atau dimensi hardcoded. Semua wajib merujuk ke token semantik (`FinTrackTheme.colors`, `FinTrackTheme.typography`, `FinTrackTheme.shapes`, `FinTrackTheme.borders`, `FinTrackTheme.shadows`).
   - Token warna awal light mode mengikuti spesifikasi [StyleGuide.md](../../StyleGuide.md):
     - `background`: `#F7F3E8`
     - `surface`: `#FFFFFF`
     - `textPrimary`: `#171717`
     - `outline`: `#171717` (border tegas)
     - `primary`: `#FFD84D`
     - `accentBlue`: `#A8D5FF`, `accentPink`: `#FFA8C5`, `accentGreen`: `#B8F28B`
2. **Karakter Border dan Shadow:**
   - Border utama menggunakan garis solid 2 dp dengan warna `outline`.
   - Hard offset shadow (misal `offset: 4.dp, blur: 0.dp`) diterapkan secara terukur pada CTA utama (tombol simpan/konfirmasi) dan kartu ringkasan saldo.
   - Jangan menerapkan hard shadow ke setiap item daftar transaksi secara berlebihan agar tidak menimbulkan visual clutter.
   - Efek penekanan tombol (pressed state) menurunkan offset shadow untuk memberikan sensasi taktil fisik yang memuaskan.
3. **Strategi Distribusi Tipografi Sora:**
   - File font Sora (bobot Regular, Medium, SemiBold, Bold) **wajib dibundel langsung di dalam aset APK (`res/font/`)**.
   - **Dilarang keras mengunduh file font dari URL internet arbitrer saat runtime** untuk menjaga reliabilitas offline-first dan keamanan APK.
   - Mengaktifkan fitur angka tabular (*tabular figures / tnum*) jika didukung untuk menyelaraskan nominal uang dalam daftar transaksi.
   - Wajib mendukung penskalaan font sistem Android (`sp`), dilarang mengunci ukuran teks ke unit `dp` tetap yang membatalkan aksesibilitas.
4. **Kejelasan Aksesibilitas Finansial:**
   - Transaksi masuk dan keluar dilarang hanya dibedakan berdasarkan warna hijau/merah. Wajib menyertakan tanda eksplisit (`+Rp` atau `-Rp`) dan ikon indikator arah panah.
5. **Dukungan Dark Mode:**
   - Dark mode dibangun dengan token kontras gelap yang nyaman di mata, menghindari background hitam pekat 100% yang menyilaukan saat dipadu border kontras tinggi.

---

## 4. Konsekuensi

### Dampak Positif
- Tampilan aplikasi terlihat sangat khas, premium, dan berbeda secara mencolok dari aplikasi pencatatan keuangan biasa.
- Tingkat keterbacaan angka dan kejelasan aksi pengguna meningkat secara signifikan.
- UI berfungsi konsisten tanpa internet sejak detik pertama aplikasi diinstal.

### Dampak Negatif & Trade-off
- Perlu membuat custom Compose modifiers atau kustomisasi Material 3 Shape/Border/Shadow yang tidak disediakan secara default oleh library standar.
- Sedikit penambahan ukuran binary APK untuk aset font Sora.

### Mitigasi & Tindak Lanjut
- Membuat modul dasar `designsystem` pada scaffold Android (M1.1) yang merangkum komponen umum (FinTrackButton, FinTrackCard, FinTrackTextField) dengan styling neobrutalism yang konsisten.
- Menguji rendering tema pada perangkat dengan berbagai DPI dan memastikan rasio kontras warna memenuhi standar WCAG AA.

---

## 5. Referensi & Dokumen Terkait

- [StyleGuide.md Bagian 1, 2, dan 3](../../StyleGuide.md)
- [PRD.md FR-040, NFR-001 s.d. NFR-003](../../PRD.md)
- [Architecture.md Bagian 14 (ADR-009)](../../Architecture.md)
- [TODO.md Milestone 1.1: Scaffold Android](../../TODO.md)
