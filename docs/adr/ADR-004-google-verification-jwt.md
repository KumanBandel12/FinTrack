# ADR-004 — Google Verification dan Siklus JWT

**Status:** ACCEPTED  
**Tanggal:** 2026-10-13  
**Pengambil Keputusan:** Tim Pengembang FinTrack  
**Konsultasi / Stakeholder:** Tim Backend & Keamanan Aplikasi  

---

## 1. Konteks

FinTrack memerlukan mekanisme autentikasi pengguna yang aman, minim gesekan, dan terstandarisasi. Pengguna aplikasi Android FinTrack melakukan login menggunakan Google Sign-In / Credential Manager.

Tantangan dan kebutuhan keamanan:
1. **Verifikasi Identitas:** Klien Android tidak boleh sekadar mengirimkan alamat email atau user ID ke backend, karena data tersebut sangat mudah dipalsukan. Backend wajib memverifikasi secara kriptografis keaslian token identitas dari Google.
2. **Sesi Aplikasi Terpisah:** Backend tidak boleh bergantung langsung pada Google untuk setiap request API berikutnya. Menggunakan ID token Google untuk setiap panggilan REST API adalah antipattern karena rentan latensi validasi eksternal dan masa berlaku ID token Google yang kaku.
3. **Isolasi Kepemilikan Data Pengguna:** Semua endpoint privat di backend harus mengotorisasi request berdasarkan *authenticated principal* di dalam konteks keamanan Spring Security, bukan dari field `userId` yang dikirimkan klien dalam body payload.
4. **Penyimpanan Token di Android:** Token sesi yang disimpan di perangkat pengguna harus terlindung dari ekstraksi pihak ketiga yang tidak berwenang.

---

## 2. Opsi yang Dipertimbangkan

### Opsi 1: Otentikasi Berbasis Email dan Password Sendiri
- **Deskripsi:** Backend FinTrack mengelola tabel user dengan password hash (BCrypt) sendiri dan form registrasi/login internal.
- **Kelebihan:**
  - Independen penuh dari penyedia identitas pihak ketiga (Google).
- **Kekurangan:**
  - Menambah risiko keamanan penyimpanan kredensial password, alur reset password, dan verifikasi email.
  - Gesekan login bagi pengguna Android lebih tinggi dibandingkan One Tap Google Sign-In.

### Opsi 2: Mengirimkan Google ID Token pada Setiap Request API
- **Deskripsi:** Aplikasi Android menyertakan Google ID Token langsung sebagai Bearer token di setiap header panggilan REST API. Backend memverifikasi token Google ke public key Google pada setiap request.
- **Kelebihan:**
  - Tidak perlu membuat token sesi internal di backend.
- **Kekurangan:**
  - Sangat membebani backend (overhead verifikasi eksternal atau cache JWKS Google).
  - Umur Google ID Token singkat (~1 jam), sehingga klien harus terus-menerus berinteraksi dengan Google API untuk refresh token.
  - Tidak memungkinkan backend mencabut akses (revocation) secara independen.

### Opsi 3: Pertukaran Google ID Token dengan JWT Internal (Access + Refresh Token) (Dipilih)
- **Deskripsi:** 
  1. Klien Android memperoleh Google ID Token via Google Credential Manager.
  2. Klien mengirimkan Google ID Token ke endpoint backend `POST /api/v1/auth/google`.
  3. Backend memverifikasi token tersebut menggunakan library resmi Google API Client (memeriksa issuer, audience/Client ID, expiry, dan signature).
  4. Backend mengekstrak `google_subject` (`sub`) sebagai identitas stabil utama pengguna, membuat/memperbarui record `User`.
  5. Backend menerbitkan pasangan token internal FinTrack:
     - **Access Token:** JWT masa berlaku singkat (misal 15–60 menit), stateless, ditandatangani dengan secret key backend yang aman.
     - **Refresh Token:** Token terenkripsi/acak masa berlaku lebih panjang (misal 14–30 hari), tersimpan di database server dengan kemampuan pencabutan (revocation/rotation).
- **Kelebihan:**
  - Otentikasi kuat dan zero-trust: identitas diverifikasi langsung dari Google.
  - Performa tinggi: request API berikutnya cukup memvalidasi JWT internal tanpa panggilan ke Google.
  - Kontrol sesi penuh di backend: dapat mencabut refresh token saat logout atau terdeteksi anomali.
  - Mendukung siklus hidup aplikasi mobile offline-first dengan graceful token refresh.
- **Kekurangan:**
  - Perlu implementasi rotasi refresh token, penyimpanan aman di Android (EncryptedSharedPreferences / Keystore), dan interceptor OkHttp untuk auto-refresh.

---

## 3. Keputusan

FinTrack mengadopsi **Opsi 3: Pertukaran Google ID Token dengan JWT Internal FinTrack (Access Token & Refresh Token)**.

Aturan pelaksanaan keputusan ini:
1. **Verifikasi Resmi Google:**
   - Endpoint `POST /api/v1/auth/google` wajib memvalidasi token menggunakan Google ID Token Verifier resmi.
   - Wajib memeriksa: `issuer` (`https://accounts.google.com` atau `accounts.google.com`), `audience` cocok dengan Google OAuth Client ID aplikasi, waktu kedaluwarsa (`exp`), dan keberadaan klaim `sub` serta `email`.
2. **Kunci Identitas Pengguna:**
   - Identitas permanen pengguna di backend adalah `google_subject` (`sub`), bukan alamat email (karena alamat email dapat berubah atau dimutasi di Google).
3. **Spesifikasi JWT Internal:**
   - **Access Token:** Menggunakan format JWT (RFC 7519), algoritma HMAC-SHA256 (HS256) atau RSA (RS256), masa aktif singkat (misal 30 menit). Memuat klaim standar: `sub` (internal FinTrack User UUID), `iss` (`fintrack-api`), `iat`, `exp`.
   - **Refresh Token:** Token opaque dengan rotasi saat digunakan (refresh token rotation) dan masa berlaku maksimum 30 hari. Disimpan di backend dengan status aktif/revoked.
4. **Otorisasi Ketat pada API:**
   - Semua operasi database dan bisnis di backend **wajib mengambil `userId` dari Security Context (Principal)** yang diekstrak dari validasi JWT.
   - Endpoint privat **dilarang mempercayai `userId` dari query parameter atau request body** untuk mencegah serangan Insecure Direct Object Reference (IDOR) atau akses silang pengguna.
5. **Keamanan Klien Android:**
   - Token disimpan menggunakan `EncryptedSharedPreferences` yang kuncinya dikelola oleh `Android Keystore`.
   - Menggunakan OkHttp `Authenticator` untuk menangani status HTTP 401 dan melakukan refresh token otomatis ke `POST /api/v1/auth/refresh`.

---

## 4. Konsekuensi

### Dampak Positif
- Arsitektur autentikasi standar industri yang aman dan teruji.
- Pengguna mendapatkan pengalaman login 1-klik yang mulus di perangkat Android.
- Perlindungan penuh terhadap spoofing data pengguna dan kebocoran identitas.

### Dampak Negatif & Trade-off
- Membutuhkan penyiapan Google Cloud Console OAuth 2.0 Credentials (Client ID untuk Android dan Client ID untuk Backend/Web).
- Penanganan token expiry di lapisan jaringan Android (interceptor & authenticator) harus bebas dari infinite loop saat refresh gagal.

### Mitigasi & Tindak Lanjut
- Konfigurasi Client ID disimpan dalam environment variable backend (`FINTRACK_GOOGLE_CLIENT_ID`) dan tidak boleh di-hardcode.
- Jika refresh token kedaluwarsa atau tidak valid, OkHttp Authenticator harus memicu event sign-out bersih dan mengarahkan pengguna kembali ke layar Login.

---

## 5. Referensi & Dokumen Terkait

- [PRD.md FR-001 s.d. FR-005](../../PRD.md)
- [Architecture.md Bagian 7 (`User`), Bagian 11 (Tabel API), dan Bagian 12 (`Autentikasi dan Keamanan`)](../../Architecture.md)
- [ADR-005: Enkripsi Lokal dan Key Lifecycle](ADR-005-enkripsi-lokal-key-lifecycle.md)
