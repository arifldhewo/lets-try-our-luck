# PRD — "let's Try My Luck" · Personal Job Tracker

**Versi:** 1.0 (draft)
**Status:** Untuk direview
**Tipe produk:** Personal web app (single-user)
**Owner:** Irfan Prima Putra
**Terakhir diperbarui:** 2026-11-25

---

## 1. Ringkasan

"let's Try My Luck" adalah aplikasi web personal untuk melacak lamaran kerja — mulai dari menemukan lowongan, mengirim CV, sampai tahu hasil akhirnya. Saat ini data hidup di satu database Notion (`let's Try My Luck`) berisi 56 perusahaan dan 26 lamaran (periode Jan–Feb 2024).

Aplikasi ini mengubah tracker Notion yang pasif menjadi alat kerja yang aktif: pipeline yang jelas, analitik channel mana yang benar-benar menghasilkan, dan pengingat follow-up supaya lamaran tidak menggantung tanpa kabar.

### Masalah yang diselesaikan

1. **Tidak ada gambaran funnel.** Saat ini status hanya berupa teks bebas; susah tahu berapa yang masih menggantung vs ditolak vs lolos.
2. **Channel tidak terukur.** Dari data lama: semua 7 penolakan datang via Techinasia, sedangkan 2 "Approved" justru dari website perusahaan — tapi insight ini tidak kelihatan tanpa analisis manual.
3. **Lamaran menggantung.** 17 dari 26 lamaran masih "Sent CV" tanpa tindak lanjut; tidak ada sistem pengingat.
4. **Skema data berantakan.** Kolom "Industri" salah opsi, "Overal State" kosong semua, ada baris kosong dan typo.
5. **Input lambat.** Menambah lamaran di Notion butuh banyak klik dan mengetik ulang.

---

## 2. Tujuan & Non-Tujuan

### Tujuan (Goals)
- **G1** — Satu tempat untuk mencatat, memfilter, dan melihat semua lamaran dengan cepat (input < 30 detik per lamaran).
- **G2** — Pipeline status yang jelas dan konsisten, menggantikan status teks bebas.
- **G3** — Dashboard analitik yang menjawab: *channel mana yang paling efektif, berapa response rate, lamaran mana yang perlu di-follow up.*
- **G4** — Migrasi mulus dari Notion tanpa kehilangan data.
- **G5** — Bisa diakses cepat di desktop maupun HP (job hunting sering di HP).

### Non-Tujuan (Non-Goals) — untuk MVP
- Bukan multi-user / SaaS. Tidak ada tim, sharing, atau billing.
- Bukan job board. Tidak menarik lowongan otomatis dari internet.
- Tidak membuat/mengirim lamaran otomatis ke perusahaan.
- Tidak ada mobile native app (cukup web responsif + PWA di fase berikutnya).
- Tidak ada AI resume tailoring di MVP (kandidat untuk fase 3).

---

## 3. Persona & Use Case Utama

**Persona:** Job seeker tunggal (UI/UX Designer) yang melamar 10–30 lowongan dalam periode aktif, lintas channel (LinkedIn, JobStreet, Techinasia, website perusahaan, referral).

**Use case inti:**
- **UC1 — Catat lamaran baru:** saat lihat lowongan menarik, simpan ke wishlist; saat kirim CV, update jadi "Applied" dengan tanggal + channel.
- **UC2 — Pantau pipeline:** buka board, lihat mana yang masih proses interview, mana yang perlu ditindak.
- **UC3 — Follow up:** lihat daftar lamaran yang "stale" (>14 hari tanpa respons) dan lakukan follow-up.
- **UC4 — Evaluasi strategi:** lihat conversion per channel/tipe perusahaan, putuskan harus fokus melamar ke mana.
- **UC5 — Refleksi mingguan:** lihat berapa lamaran minggu ini, berapa respons yang masuk.

---

## 4. Asumsi

Karena scope belum didiskusikan, PRD ini memakai asumsi berikut (mohon dikoreksi bila tidak sesuai):

- **A1** — Single-user, data pribadi; autentikasi cukup sederhana (passkey/passphrase), bukan sistem akun kompleks.
- **A2** — Web app responsif (desktop-first untuk tabel, mobile-friendly untuk input & board).
- **A3** — Data lama diimpor sekali dari export Notion (CSV/JSON), bukan integrasi live ke Notion.
- **A4** — Hosting cloud (Vercel + Postgres) agar bisa diakses dari mana saja; alternatif self-host didukung lewat export/backup.
- **A5** — "Approved" pada data lama = lamaran lolos/diterima di tahap akhir; "Declined" = ditolak.
- **A6** — Bahasa UI: Indonesia (dengan opsi Inggris di kemudian hari).

---

## 5. Scope

### 5.1 MVP (Fase 1)
- CRUD lamaran (create, read, update, archive/delete) + quick-add.
- Model data baru: `Company`, `Application`, `Activity` (log).
- Pipeline status + outcome terpisah.
- **Table view** lengkap (filter, sort, group, inline edit, bulk action).
- **Board (Kanban)** per tahap pipeline, drag-and-drop.
- **Detail lamaran** (semua field + timeline aktivitas + catatan).
- **Dashboard** dasar: KPI, funnel, conversion per channel, daftar follow-up.
- **Import dari Notion** (CSV/JSON) dengan preview & mapping.
- **Export** ke CSV/JSON (data ownership).
- Search global + filter tersimpan (saved views).
- Dark mode, responsif.

### 5.2 Fase 2
- Kalender (tanggal apply, jadwal interview, deadline challenge).
- Pengingat/follow-up otomatis (in-app + email/notif).
- Analytics lanjutan: time-to-response, time-to-offer, tren mingguan, per tipe/ukuran perusahaan.
- Manajemen dokumen (versi CV, portofolio) + lampiran per lamaran.
- PWA (installable + offline read).
- Multi-view: timeline & galeri kartu.

### 5.3 Fase 3 (eksperimental)
- Parsing JD otomatis dari URL lowongan (pre-fill form).
- AI: saran tailoring CV/cover letter per JD.
- Browser extension "simpan lowongan" sekali klik.
- Multi-user opsional / share read-only.

### 5.4 Di luar scope
Job scraping massal, auto-apply, integrasi ATS perusahaan, billing.

---

## 6. Model Data

### 6.1 Entitas

#### `Company`
| Field | Tipe | Catatan |
|---|---|---|
| id | uuid | PK |
| name | text | unik (case-insensitive) |
| website | text? | |
| industry | enum? | **taksonomi industri yang benar** (bukan rating) — lihat 6.3 |
| company_type | enum? | Startup, Corporate, IT Service, Design Studio, Unicorn, Agency |
| size | enum? | `1-100`, `100-1000`, `>1000` |
| design_team_size | enum? | `1-30`, `31-50`, `>50` |
| location | text? | |
| notes | text? | |
| created_at / updated_at | timestamptz | |

#### `Application`
| Field | Tipe | Catatan |
|---|---|---|
| id | uuid | PK |
| company_id | fk | → Company |
| position | text | mis. "UI/UX Designer" |
| position_type | enum? | UI, UX, UI/UX, Product, UX Research, Interaction |
| job_url | text? | link lowongan |
| source | enum? | **di mana info ditemukan**: LinkedIn, JobStreet, Techinasia, Glints, Kalibrr, Dealls, Website, Referral, Lainnya |
| applied_via | enum? | **cara melamar**: Website, Email, LinkedIn Easy Apply, ATS, Referral, Lainnya |
| found_date | date? | tanggal info lowongan |
| applied_date | date? | tanggal kirim CV |
| stage | enum | pipeline stage — lihat 6.2 |
| outcome | enum | `active`, `rejected`, `offer`, `accepted`, `closed`, `withdrawn` — lihat 6.2 |
| is_submitted | bool | derived dari `applied_date != null` |
| salary_expectation | enum? | `Dibawah`, `Sesuai`, `Diatas` ekspektasi |
| rating_branding | enum? | `Tidak Menarik`, `Menarik`, `Sangat Menarik` |
| rating_product | enum? | idem |
| rating_design | enum? | `Tidak OK`, `OK`, `Sangat OK` |
| rating_vibe | enum? | idem `Tidak/Menarik/Sangat Menarik` |
| overall_score | enum? | `Tidak Menarik`, `Perusahaan OK`, `Perusahaan Bagus`, `Perusahaan Impian` |
| notes | text? | |
| archived | bool | default false |
| created_at / updated_at | timestamptz | |

> **Catatan desain:** field rating (`branding`, `product`, `design`, `vibe`, `overall`) **opsional** dan tidak dipakai untuk filtering inti. `overall_score` bisa manual atau dihitung (rata-rata rating) — default manual sesuai data lama.

#### `Activity` (log)
| Field | Tipe | Catatan |
|---|---|---|
| id | uuid | PK |
| application_id | fk | |
| type | enum | `created`, `applied`, `stage_changed`, `outcome_changed`, `note`, `interview`, `followup`, `reminder` |
| from_value / to_value | text? | untuk perubahan stage/outcome |
| note | text? | |
| occurred_at | timestamptz | dipakai untuk time-to-response |

### 6.2 Kanonik Pipeline

Memisahkan **stage** (posisi di proses) dari **outcome** (hasil akhir) menyelesaikan masalah "Overal State kosong".

**Stage (urut):**
1. `wishlist` — sudah di-track, belum melamar
2. `applied` — CV terkirim
3. `screening` — screening / Interview HRD
4. `interview` — Interview user / teknis
5. `challenge` — take-home / challenge
6. `offer` — dapat tawaran

**Outcome (independen dari stage):**
- `active` — masih berjalan
- `rejected` — ditolak (kapan pun)
- `offer` — tawaran masuk
- `accepted` — diterima / sepakat
- `closed` — lowongan ditutup / expired sebelum sempat diproses
- `withdrawn` — kandidat mundur

**Mapping dari status Notion lama:**

| Status lama | stage | outcome |
|---|---|---|
| Not started | wishlist | active |
| Sent CV | applied | active |
| lowongan ditutup | wishlist/applied | closed |
| Proses - Interview HRD | screening | active |
| Proses - Interview User | interview | active |
| Proses - Challange | challenge | active |
| Waiting Interview / Waiting Interview User | interview | active |
| Waiting Challange | challenge | active |
| Done - Interview HRD / Done - Interview User | interview | active |
| Done - Challange | challenge | active |
| Declined | — | rejected |
| Approved | offer | accepted |
| (kosong, belum submit) | wishlist | active |

### 6.3 Taksonomi (perbaikan dari skema lama)

- **Industry** — daftar tertutup nyata: `IT Services`, `Software/SaaS`, `E-commerce`, `Fintech`, `Media`, `Education`, `Healthcare`, `Manufacturing`, `Retail`, `Consulting`, `Agency/Design`, `Telecom`, `Other`. (Menggantikan kolom "Industri" lama yang opsi-nya salah jadi rating.)
- **Channel** dipisah jadi `source` (info) dan `applied_via` (cara), karena di data lama "From" dan "Send To" tumpang tindih.
- Semua enum disimpan sebagai kode stabil (`it_services`, `applied`, dst.) dan punya label tampilan; tidak ada free-text untuk field kategorikal.

---

## 7. Fitur & User Story

### 7.1 Quick Add Lamaran
**Story:** Sebagai job seeker, saya ingin menambah lamaran dalam < 30 detik.
- **AC1** — Tombol "+ Lamaran" di semua halaman; form minimal: Company (autocomplete), Posisi, Stage.
- **AC2** — Company baru otomatis dibuat bila belum ada; company lama di-reuse.
- **AC3** — Field lanjutan (channel, rating, notes) opsional / di-collapse.
- **AC4** — Bisa simpan sebagai `wishlist` dulu, update belakangan.

### 7.2 Table View
**Story:** Saya ingin melihat & mengedit semua lamaran dalam satu tabel padat.
- **AC1** — Kolom sesuai skema; bisa show/hide & reorder kolom.
- **AC2** — Filter multi-kriteria (stage, outcome, source, company_type, size, rentang tanggal).
- **AC3** — Sort semua kolom; group by (stage / company_type / source).
- **AC4** — Inline edit sel (select, date, checkbox).
- **AC5** — Bulk action: ubah stage/outcome, arsip, hapus.
- **AC6** — Saved views (mis. "Applied", "Perlu follow-up").
- **AC7** — Virtualized rows; tetap mulus untuk ratusan lamaran.

### 7.3 Board (Kanban)
- **AC1** — Kolom per `stage`; drag-and-drop mengubah stage + mencatat `Activity`.
- **AC2** — Kartu ringkas: company, posisi, tanggal, badge outcome, umur lamaran.
- **AC3** — Filter aktif berlaku di board.

### 7.4 Detail Lamaran
- **AC1** — Semua field + edit inline.
- **AC2** — Timeline `Activity` kronologis (created → applied → tiap perubahan stage → notes).
- **AC3** — Link ke job_url & website company.
- **AC4** — Tombol aksi cepat: "Tandai rejected", "Pindah stage", "Tambah catatan", "Set follow-up".

### 7.5 Dashboard
- **AC1** — KPI: total tracked, applied, active, rejected, offers, accepted.
- **AC2** — Funnel: wishlist → applied → screening → interview → challenge → offer.
- **AC3** — **Conversion per channel** (bar chart): response rate & offer rate per `source` / `applied_via`.
- **AC4** — Distribusi per company_type & size.
- **AC5** — **Follow-up list**: lamaran berstatus `applied`/`active` dengan `applied_date` > 14 hari dan belum ada aktivitas baru.
- **AC6** — Aktivitas mingguan (lamaran per minggu).

### 7.6 Import dari Notion
- **AC1** — Upload CSV export Notion (dan JSON opsional).
- **AC2** — Layar mapping kolom (auto-detect berdasarkan header Indonesia: Company, Posisi, From, Send To, Status, dll.).
- **AC3** — Preview + validasi; tampilkan baris yang gagal & alasannya.
- **AC4** — Terapkan normalisasi: trim spasi, buang baris kosong, perbaiki kapitalisasi enum, map status lama → stage/outcome (lihat 6.2), fix typo umum (Product Desginer→Product Designer, dsb.).
- **AC5** — Idempoten: import ulang tidak menduplikasi (dedupe by company+position+applied_date).

### 7.7 Export & Backup
- **AC1** — Export CSV & JSON lengkap (Company + Application + Activity).
- **AC2** — Backup otomatis harian (retensi 30 hari) di sisi server.

### 7.8 Search & Filter Global
- **AC1** — Search by company/posisi/catatan.
- **AC2** — Keyboard shortcut (`/` untuk search, `n` untuk new).

### 7.9 Settings
- **AC1** — Ubah daftar enum (channel, industry, tipe) tanpa deploy.
- **AC2** — Tema terang/gelap.
- **AC3** — Bahasa (ID default).
- **AC4** — Konfigurasi follow-up threshold (default 14 hari).

---

## 8. Analitik — Definisi Metrik

Agar angka konsisten, metrik didefinisikan eksplisit:

- **Applied** = jumlah `Application` dengan `applied_date != null`.
- **Response rate** = (lamaran yang punya ≥1 aktivitas masuk selain "applied", yaitu screening/interview/challenge/offer/rejected) ÷ Applied.
- **Interview rate** = lamaran yang pernah mencapai stage ≥ `screening` ÷ Applied.
- **Offer rate** = lamaran dengan outcome `offer` atau `accepted` ÷ Applied.
- **Conversion per channel** = metrik di atas, dihitung per nilai `source` (dan/atau `applied_via`).
- **Median time-to-first-response** = median(`occurred_at` respons pertama − `applied_date`).
- **Stale application** = `outcome = active`, `stage ∈ {applied}`, dan `now − applied_date > threshold` (default 14 hari) tanpa aktivitas baru.
- **Weekly cadence** = jumlah `applied_date` per minggu ISO.

> Insight pembanding dari data lama (baseline): Techinasia → 7/9 ditolak; Website → 2 approved dari 7; LinkedIn → 6 lamaran belum ada hasil. Dashboard harus bisa mereproduksi insight semacam ini secara otomatis.

---

## 9. UX & Alur Layar

Navigasi utama (sidebar): **Dashboard · Applications · Board · Companies · Insights · Settings**.

**Alur kunci — "Apply sekarang":**
1. Klik `+ Lamaran` (atau `n`).
2. Ketik nama company → autocomplete / "buat baru".
3. Isi posisi + source + applied_date (default hari ini).
4. Simpan → muncul toast + kartu masuk ke Board kolom "Applied".
5. Opsional: buka detail, tambah job_url & catatan.

**Alur follow-up:**
1. Dashboard menampilkan "Perlu follow-up (N)".
2. Klik → tabel terfilter.
3. Pilih satu → detail → "Tambah catatan follow-up" (tercatat sebagai `Activity`).

**Prinsip UX:**
- Keyboard-first untuk power user; touch-friendly untuk mobile.
- Empty state informatif (termasuk tombol import saat data kosong).
- Batch operations tidak konfirmasi berlebihan, tapi hapus selalu konfirmasi/undo.

---

## 10. Non-Functional Requirements

- **Performa:** TTFB < 500ms (P95), tabel 500+ baris mulus (virtualized), dashboard load < 1.5s.
- **Responsif:** breakpoint mobile (≤640), tablet, desktop; board bisa horizontal scroll di mobile.
- **Aksesibilitas:** WCAG 2.1 AA — kontras, fokus keyboard, label ARIA, navigasi keyboard penuh.
- **Privasi & keamanan:** data pribadi; akses via autentikasi (passkey/passphrase + session cookie httpOnly); enkripsi in-transit (HTTPS) & at-rest di DB; tidak ada tracking pihak ketiga.
- **Reliabilitas:** backup harian otomatis; import transaksional (all-or-nothing per batch).
- **Portabilitas data:** export penuh kapan saja (anti-lock-in).
- **i18n-ready:** string UI dipisah dari kode; default Indonesia.
- **Observability:** error logging + basic uptime (opsional Sentry).

---

## 11. Stack & Arsitektur (usulan)

- **Frontend/Backend:** Next.js (App Router) + TypeScript.
- **UI:** Tailwind CSS + shadcn/ui; ikon lucide.
- **Tabel:** TanStack Table (virtualized).
- **Kanban:** dnd-kit.
- **Chart:** Recharts.
- **Validasi:** Zod (shared schema client/server).
- **DB:** PostgreSQL (Supabase/Neon) + **Drizzle ORM**; migrasi ter-version.
- **Auth:** single-user — passkey (WebAuthn) atau passphrase + session; tanpa sistem registrasi publik.
- **Deploy:** Vercel (app) + Postgres terkelola; alternatif self-host (Docker) dengan Postgres lokal.
- **Testing:** Vitest (unit), Playwright (E2E untuk alur inti: quick-add, import, drag stage).
- **Struktur domain:** `companies`, `applications`, `activities` + modul `import`, `analytics`.

> Alternatif lebih ringan (jika tidak mau cloud DB): SQLite (Turso/libSQL) + Drizzle — cocok untuk single-user, lebih murah, tapi kurang ideal untuk serverless Vercel tanpa Turso.

---

## 12. Milestone (tanpa estimasi waktu)

- **M0 — Fondasi:** scaffold Next.js + TS + Tailwind + shadcn, skema DB + migrasi, auth single-user, CI.
- **M1 — Core CRUD:** Company & Application CRUD, quick-add, detail + Activity log, pipeline enum.
- **M2 — Views:** Table (filter/sort/group/inline edit) + Board (drag-drop) + search.
- **M3 — Import:** Notion CSV import dengan mapping, normalisasi, dedupe, preview.
- **M4 — Dashboard & Insights:** KPI, funnel, conversion per channel, follow-up list.
- **M5 — Polish:** export/backup, settings, dark mode, a11y, E2E tests.
- **M6 — (Fase 2):** kalender, reminders, dokumen/CV, PWA, analytics lanjutan.

**Definisi "MVP selesai":** data Notion lama terimpor 100%, bisa CRUD + lihat table & board, dashboard menampilkan conversion per channel, dan data bisa diexport.

---

## 13. Risiko & Mitigasi

| Risiko | Dampak | Mitigasi |
|---|---|---|
| Data lama kotor (baris kosong, typo, status ganda) | Import berantakan | Layar normalisasi + preview + dedupe; mapping status terdokumentasi (6.2) |
| Over-engineering (fitur AI/scraping kepagian) | MVP molor | Kunci scope ke Fase 1; fitur AI/extension di Fase 3 |
| Single-user auth dianggap "cukup aman" | Data pribadi bocor | Passkey/passphrase + cookie httpOnly + HTTPS; tidak ada URL publik |
| Enum terlalu kaku | Sulit adaptasi | Enum editable di Settings |
| Ketergantungan Notion | Lock-in | Import sekali + export penuh; tanpa live sync |

---

## 14. Pertanyaan Terbuka

1. **Hosting:** cloud (Vercel + Postgres) atau self-host lokal/VPS? (mempengaruhi pilihan DB & auth)
2. **Multi-user?** Tetap personal single-user, atau nanti mau bisa dibagikan (read-only)?
3. **AI/scraping:** perlukah parsing JD dari URL & tailoring CV di dekat-dekat MVP, atau tunda ke Fase 3?
4. **Arti "Approved"** di data lama: lolos screening, dapat offer, atau diterima kerja? (menentukan mapping final)
5. **Field rating** (branding/product/design/vibe/overall): tetap dipakai untuk keputusan, atau cukup arsip saja?
6. **Target platform utama:** desktop dulu atau mobile-first?

---

## 15. Success Metrics (untuk produk ini sendiri)

- **Input cepat:** median waktu tambah 1 lamaran < 30 detik.
- **Adopsi:** semua 56 perusahaan lama berhasil terimpor tanpa data loss (target 100%).
- **Konsistensi data:** ≥ 90% lamaran punya `stage`, `outcome`, dan `source` terisi.
- **Follow-up coverage:** 100% lamaran stale muncul di daftar follow-up.
- **Insight tersedia:** dashboard bisa menampilkan conversion per channel dalam < 1 menit buka app.
