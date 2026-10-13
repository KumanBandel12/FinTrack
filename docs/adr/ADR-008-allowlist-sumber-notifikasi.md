# ADR-008 — Allowlist dan Sumber Notifikasi Pertama

**Status:** ACCEPTED  
**Tanggal:** 2026-10-13  
**Pengambil Keputusan:** Tim Pengembang FinTrack  
**Konsultasi / Stakeholder:** Tim Rekayasa Android & Desain Produk  

---

## 1. Konteks

Fitur penangkapan transaksi otomatis FinTrack beroperasi melalui Android `NotificationListenerService`. Layanan ini mendengarkan aliran notifikasi sistem Android secara global.

Terdapat implikasi teknis dan etis krusial:
1. **Privasi Pengguna:** Mendengarkan semua notifikasi perangkat secara acak (chat pribadi, email, SMS umum, media sosial) melanggar prinsip minimisasi data dan menimbulkan kecurigaan privasi yang serius.
2. **Kinerja & Baterai:** Memproses teks dari ratusan notifikasi aplikasi non-finansial memboroskan baterai perangkat dan memicu wake lock yang tidak perlu.
3. **Variasi Format Notifikasi:** Setiap aplikasi perbankan atau dompet digital (misal BCA Mobile/myBCA, Mandiri Livin', BRImo, GoPay, OVO, Dana) memiliki format layout, judul, dan bahasa notifikasi yang sangat beragam dan dapat berubah sewaktu-waktu.
4. **Tahapan Rilis:** Mencoba mendukung seluruh bank di Indonesia sekaligus sejak hari pertama akan menghasilkan parser yang rapuh dan sulit dipelihara.

Oleh karena itu, diperlukan strategi seleksi sumber notifikasi yang terfilter ketat (allowlist) dan pendekatan parsing yang modular, terisolasi, serta ber-versi.

---

## 2. Opsi yang Dipertimbangkan

### Opsi 1: Menangkap Semua Notifikasi dengan Regex Universal
- **Deskripsi:** Menerima notifikasi dari semua package aplikasi di ponsel, lalu mencari pola kata kunci seperti "Rp", "Transfer", "Berhasil" menggunakan satu ekspresi reguler raksasa.
- **Kelebihan:**
  - Terlihat fleksibel karena dapat menangkap notifikasi dari bank apa pun tanpa registrasi package.
- **Kekurangan:**
  - Pelanggaran privasi berat: membaca notifikasi percakapan pribadi (WhatsApp, Telegram) yang menyebutkan kata "Rp".
  - Sangat rentan *false positive* (misal spam promo diskon atau chat tagihan teman).
  - Parser regex universal sangat rapuh dan mustahil diuji secara deterministik.

### Opsi 2: Allowlist Statis Keras (Hardcoded) Tanpa Versi Parser
- **Deskripsi:** Mengunci daftar package ID di dalam kode aplikasi dan menggunakan parser if-else panjang.
- **Kelebihan:**
  - Sederhana pada iterasi pertama.
- **Kekurangan:**
  - Sulit diperluas dan jika format notifikasi sebuah bank berubah, seluruh modul notifikasi berisiko mengalami regresi.
  - Tidak ada riwayat versi parser (`parser_version`) pada draft untuk kebutuhan audit atau backward compatibility.

### Opsi 3: Allowlist Berbasis Package ID + Parser Modular Ber-versi dengan Fixture Sintetis (Dipilih)
- **Deskripsi:** 
  1. **Allowlist Terdaftar:** Sistem hanya memproses notifikasi dari package aplikasi yang secara eksplisit ada di daftar allowlist FinTrack. Notifikasi di luar allowlist langsung diabaikan seketika di gerbang masuk listener tanpa membaca teks lengkap.
  2. **Sumber Pertama (Pilot Sources):** Memilih 1–2 sumber paling populer di Indonesia dengan format stabil (misalnya GoPay / BCA) sebagai sumber pilot pada Milestone 0 & 2.
  3. **Parser Modular Ber-versi:** Setiap sumber memiliki kelas parser mandiri (`NotificationParser`) yang mengimplementasikan interface domain dengan atribut `sourceAppId`, `parserId`, dan `parserVersion`.
  4. **Pengujian dengan Fixture Teranonimkan:** Setiap parser wajib diuji menggunakan suite unit test berbasis fixture teks sintetis/teranonimkan (tidak boleh menggunakan data transaksi nyata pengguna yang memuat nama atau nomor rekening asli).
  5. **Status Listener Transparan:** Jika izin notifikasi dicabut pengguna di pengaturan Android, aplikasi menampilkan indikator status non-intrusif di UI.
- **Kelebihan:**
  - Perlindungan privasi mutlak: notifikasi di luar allowlist tidak pernah diproses atau disimpan.
  - Arsitektur modular: penambahan bank baru cukup membuat satu parser baru dan test suite baru tanpa menyentuh parser yang sudah stabil.
  - Performa baterai optimal: zero background wake lock atau polling.
- **Kekurangan:**
  - Aplikasi baru dapat menangkap transaksi dari bank yang sudah dibuatkan parser khususnya.

---

## 3. Keputusan

FinTrack mengadopsi **Opsi 3: Allowlist Berbasis Package ID dan Arsitektur Parser Modular Ber-versi**.

Aturan pelaksanaan keputusan ini:
1. **Penyaringan di Pintu Gerbang:**
   - Metode `onNotificationPosted` pada `NotificationListenerService` wajib memeriksa `sbn.packageName` terhadap daftar `ALLOWLIST_APP_PACKAGES`.
   - Notifikasi dari package yang tidak terdaftar **wajib langsung di-drop (early return)** tanpa parsing, tanpa alokasi memori string yang tidak perlu, dan tanpa penyimpanan apa pun.
2. **Pemilihan Sumber Pertama:**
   - Sumber notifikasi pertama yang diimplementasikan dan divalidasi pada M0.4 dan Milestone 2 adalah:
     - **GoPay** (PT GoTo Gojek Tokopedia) sebagai perwakilan e-wallet.
     - **BCA (myBCA / BCA mobile)** sebagai perwakilan bank konvensional.
   - Sumber tambahan (Livin', BRImo, OVO, Dana) ditambahkan secara bertahap pada milestone lanjutan (P1).
3. **Format Output Parser:**
   - Setiap parser menghasilkan objek `ParsedNotificationResult` yang berisi:
     - `nominal`: dalam satuan integer minor (`amount_minor`)
     - `currency`: kode ISO (misal `IDR`)
     - `timestamp`: waktu transaksi
     - `transactionType`: `EXPENSE`, `INCOME`, atau `UNKNOWN`
     - `accountHint`: petunjuk nomor rekening/e-wallet tersensor
     - `dedupeKey`: signature deterministik untuk mencegah duplikasi
     - `parserVersion`: integer versi parser yang digunakan
4. **Data Mentah Tidak Disimpan:**
   - Sesuai ADR-002, field `raw_text` atau isi penuh notifikasi **tidak disimpan ke database**.
5. **Kebijakan Pengujian Fixture:**
   - Parser wajib diuji dengan test case unit yang memuat file fixture teks teranonimkan (`test/resources/fixtures/notifications/...`). Data riil pribadi dilarang masuk ke codebase git.

---

## 4. Konsekuensi

### Dampak Positif
- Keamanan dan privasi pengguna terlindungi secara transparan dan bertanggung jawab.
- Arsitektur parser bersih, modular, *extensible*, dan mudah diuji secara otomatis dalam pipeline CI.
- Mengeliminasi beban konsumsi baterai berlebih pada perangkat Android pengguna.

### Dampak Negatif & Trade-off
- Notifikasi dari bank yang belum didukung belum dapat dicatat otomatis; pengguna harus menginputnya secara manual via form transaksi.
- Pengembang perlu memantau pembaruan berkala bila bank mengubah format string notifikasi mereka.

### Mitigasi & Tindak Lanjut
- Sediakan fallback `UNPARSED`: jika package ada di allowlist namun format string tidak cocok dengan regex aktif, buat draft berstatus `UNPARSED` agar pengguna tetap dapat mengedit nominal secara cepat.
- Memberikan panduan kontribusi fixture anonim bagi pengembang atau pengguna penguji yang ingin menambahkan dukungan bank baru.

---

## 5. Referensi & Dokumen Terkait

- [PRD.md FR-020, FR-021, FR-024, FR-025, FR-027](../../PRD.md)
- [Architecture.md Bagian 9 (`Pipeline Notifikasi`) dan Bagian 14 (ADR-008)](../../Architecture.md)
- [ADR-002: Draft Notifikasi Wajib Direview](ADR-002-draft-notifikasi-review.md)
