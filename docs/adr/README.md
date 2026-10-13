# Architecture Decision Records (ADR) — FinTrack

Dokumen ini mencatat seluruh keputusan arsitektural penting (*Architectural Decision Records*) yang diambil dalam pengembangan proyek FinTrack. Setiap keputusan mendokumentasikan konteks, opsi yang dievaluasi, keputusan yang disepakati, konsekuensi (trade-off), serta status keberlakuannya.

Untuk menambahkan keputusan arsitektural baru, gunakan format baku pada [template.md](template.md).

---

## Daftar Rekaman Keputusan (ADR Index)

| Nomor | Judul Keputusan | Status | Tanggal | Topik Utama |
|---|---|---|---|---|
| [ADR-001](ADR-001-offline-first-sync-outbox.md) | Offline-first dan Sync Outbox | ACCEPTED | 2026-10-13 | Room single source of truth, Outbox pattern, WorkManager |
| [ADR-002](ADR-002-draft-notifikasi-review.md) | Draft Notifikasi Wajib Direview | ACCEPTED | 2026-10-13 | Kotak masuk review, larangan auto-posting, privasi payload mentah |
| [ADR-003](ADR-003-transfer-dua-efek-ledger.md) | Transfer dengan Dua Efek Ledger | ACCEPTED | 2026-10-13 | Operasi logis tunggal, simetri ledger, isolasi fee, eksklusi cash flow |
| [ADR-004](ADR-004-google-verification-jwt.md) | Google Verification dan Siklus JWT | ACCEPTED | 2026-10-13 | Verifikasi Google ID token, Access & Refresh JWT internal, Keystore |
| [ADR-005](ADR-005-enkripsi-lokal-key-lifecycle.md) | Enkripsi Lokal dan Key Lifecycle | ACCEPTED | 2026-10-13 | SQLCipher / Room encryption, Android Keystore, App Lock vs DB |
| [ADR-006](ADR-006-idempotensi-sync-conflict.md) | Idempotensi dan Konflik Sinkronisasi | ACCEPTED | 2026-10-13 | UUID client_id stabil, idempotency key, optimistic versioning, bounded retry |
| [ADR-007](ADR-007-backend-lokal-laptop.md) | Backend Lokal Langsung di Laptop | ACCEPTED | 2026-10-13 | Spring Boot 3.5.x + MySQL lokal, profil local di-ignore, instruksi start/stop |
| [ADR-008](ADR-008-allowlist-sumber-notifikasi.md) | Allowlist dan Sumber Notifikasi Pertama | ACCEPTED | 2026-10-13 | Allowlist package ID, parser ber-versi, GoPay & BCA pilot, fixture anonim |
| [ADR-009](ADR-009-neobrutalism-dan-sora.md) | Neobrutalism dan Tipografi Sora | ACCEPTED | 2026-10-13 | Desain sistem berbasis token, border & hard shadow, font Sora bundel APK |

---

## Aturan dan Panduan ADR

1. **Format Baku:** Setiap dokumen wajib menyertakan lima bagian utama:
   - **Konteks:** Mengapa keputusan ini perlu diambil dan batasan yang ada.
   - **Opsi yang Dipertimbangkan:** Alternatif solusi dengan analisis kelebihan dan kekurangan.
   - **Keputusan:** Solusi yang dipilih secara tegas beserta aturan implementasinya.
   - **Konsekuensi:** Dampak positif, trade-off / konsekuensi negatif, dan mitigasi.
   - **Status:** Status aktif dokumen (`PROPOSED`, `ACCEPTED`, `SUPERSEDED`, `DEPRECATED`).
2. **Kesesuaian Spesifikasi:** ADR adalah dokumen hidup yang melengkapi [Architecture.md](../../Architecture.md) dan [PRD.md](../../PRD.md). Jika sebuah keputusan merevisi keputusan terdahulu, status keputusan lama diubah menjadi `SUPERSEDED` dengan mencantumkan tautan ke ADR baru penggantinya.
