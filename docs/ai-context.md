# AI Context & Development Mandates

All AI agents MUST read and follow these rules before proposing or implementing any changes.

> **Orientasi (paham project tanpa baca kode):** baca berurutan — dokumen ini (mandat) → [architecture.md](architecture.md) (peta modul) → [data-model.md](data-model.md) (struktur data) → [features/index.md](features/index.md) (detail per fitur).

---

## 1. Tech Stack (STRICT — no deviations)

Sumber: [initiate-file/02-architecture.md](initiate-file/02-architecture.md) §2. Perubahan stack wajib lewat entry baru di [decision-log.md](decision-log.md).

| Lapisan | Pilihan | Constraint |
|---|---|---|
| Bahasa & framework | PHP 8.3, Laravel 12 | Versi minor boleh naik, major tidak |
| Back office / panel admin | Filament 4 | CRUD master data, pengaturan, panel Super Admin |
| Terminal kasir | Inertia 2 + React 19 + TypeScript + shadcn/ui, dibungkus PWA | Bukan Livewire — kebutuhan offline & state kompleks |
| Basis data | PostgreSQL 16 | Bukan MySQL. JSONB, partial index, pg_trgm, dan RLS dipakai dan tidak opsional |
| Cache, sesi, antrean | Redis 7 + Laravel Horizon | — |
| Realtime | Laravel Reverb (WebSocket) | KDS, 86 list, status meja |
| Penyimpanan berkas | S3 compatible (MinIO / Cloudflare R2) | — |
| Pencarian | PostgreSQL `pg_trgm` | Meilisearch hanya jika katalog > 50.000 item |
| Penyimpanan lokal terminal | IndexedDB via Dexie + Service Worker (Workbox) | Master data & outbox offline |
| Pencetakan | Print Agent lokal (Go, single binary), fallback WebUSB / Web Bluetooth | Hanya ESC/POS standar |
| Kontainerisasi | Docker + Docker Compose | — |
| CI/CD | GitHub Actions | — |
| Reverse proxy | Traefik atau Nginx Proxy Manager | Routing subdomain tenant |
| Observability | Sentry, Laravel Pulse | — |

**Sudah ditolak, jangan diusulkan ulang:** MySQL 8, microservices sejak awal, Livewire untuk layar kasir.

---

## 2. Architecture Mandates

Sepuluh prinsip berikut bersifat absolut. Kode yang melanggar ditolak di review, bukan didiskusikan.

| Kode | Mandat | Konsekuensi konkret |
|---|---|---|
| AP-01 | **Modular monolith** | Satu deployable. Microservice hanya jika terbukti perlu |
| AP-02 | **Tenant isolation by default** | Tidak ada query tanpa scope tenant. Pelanggaran ditolak di test |
| AP-03 | **Ledger sebagai sumber kebenaran** | Stok dan kas direkonstruksi dari mutasi, bukan angka yang di-UPDATE |
| AP-04 | **Transaksi immutable** | Tidak ada UPDATE pada baris penjualan final. Koreksi lewat dokumen pembalik |
| AP-05 | **Snapshot pada titik transaksi** | Harga, nama, HPP, pajak disalin ke baris transaksi |
| AP-06 | **Offline first di terminal** | Kasir menulis ke lokal dulu, sinkronisasi menyusul |
| AP-07 | **Idempotensi di seluruh jalur tulis** | Setiap operasi tulis membawa kunci idempotensi dari klien |
| AP-08 | **Feature flag ditegakkan di server** | UI menyembunyikan, server menolak. Tidak pernah hanya UI |
| AP-09 | **Baca berat dipisah dari tulis** | Laporan memakai read model teragregasi |
| AP-10 | **Aksi sensitif meninggalkan jejak** | Audit log adalah kebutuhan produk, bukan sekadar teknis |

### 2.1 Struktur kode

`app/Core/` adalah shared kernel — **tidak boleh bergantung pada modul mana pun**. Isinya: `Money/`, `Tenancy/`, `Modules/`, `Sequencing/`, `Auditing/`, `Idempotency/`, `Support/`.

Setiap modul di `modules/{Nama}/` memakai empat lapis tetap: `Domain/` (entity, VO, event, interface repository) → `Application/` (command, query, DTO) → `Infrastructure/` (Eloquent, migration, provider) → `Presentation/` (Http, Filament, routes).

Daftar modul: Identity, Tenancy, Catalog, Inventory, Sales, Payment, Shift, Fnb, Service, Customer, Promotion, Purchasing, Employee, Expense, Reporting, Billing, Platform.

### 2.2 Ketergantungan antar modul

Modul **tidak boleh** memanggil model Eloquent modul lain secara langsung. Komunikasi lintas modul hanya lewat tiga jalur: **domain event**, **application service publik**, atau **read model bersama**.

Arah yang diizinkan (searah, tanpa siklus):

```
Sales      ──▶ Catalog, Inventory, Customer, Promotion, Payment, Shift, Tenancy
Fnb        ──▶ Sales, Catalog
Service    ──▶ Sales, Customer, Employee
Inventory  ──▶ Catalog, Tenancy
Purchasing ──▶ Inventory, Catalog
Billing    ──▶ Tenancy
Reporting  ──▶ (baca read model saja; tidak ada modul yang bergantung padanya)
Semua      ──▶ Core, Identity, Tenancy
```

Ditegakkan otomatis lewat Deptrac/PHPArkitect di CI. Pelanggaran menggagalkan build.

### 2.3 Multi-tenancy

Single database, shared schema, kolom `tenant_id` + `business_id` di seluruh tabel transaksional. Pertahanan berlapis: middleware `ResolveTenant` → trait `BelongsToTenant` (global scope + auto-fill) → PostgreSQL RLS via `SET LOCAL app.tenant_id` → uji kebocoran dua tenant per endpoint → prefix cache `t:{tenant_id}:b:{business_id}:`.

Urutan middleware baku: `ResolveTenant` → `EnsureSubscriptionActive` → `ModuleGate` → `Permission` → Controller.

Koneksi database selalu lewat `TenantConnectionResolver`, meski v1 selalu mengembalikan koneksi yang sama.

### 2.4 Feature flag

Penegakan wajib di lima titik: middleware route `module:fnb.table`, `canAccess()` pada Filament Resource, middleware API (balas `403 MODULE_DISABLED`), render bersyarat via prop `enabledModules`, dan guard di awal job/listener — karena flag bisa berubah setelah job diantrekan.

Cache di Redis pada kunci `t:{tenant}:b:{business}:modules` **tanpa TTL**; invalidasi eksplisit saat event `BusinessModuleChanged`.

### 2.5 Perhitungan total

Urutan 11 langkah (subtotal item → modifier → diskon item → subtotal → diskon transaksi → dasar pengenaan → service charge → pajak → total sebelum bulat → pembulatan → grand total) dikodekan **hanya** di kelas `PricingEngine`. Klien memakai port TypeScript-nya. Duplikasi logika diterima demi offline, tetapi kesetaraan diverifikasi lewat golden test dataset yang sama di kedua sisi. Detail: [initiate-file/02-architecture.md](initiate-file/02-architecture.md) §9.2.

### 2.6 Penomoran dokumen

Format `{PREFIX}-{KODE_TERMINAL}-{YYMMDD}-{URUT}` (mis. `INV-K01-260810-0042`), dihasilkan **di klien** agar tahan offline; server hanya memverifikasi keunikan `terminal_id + document_number`. Dokumen sisi server (PO, transfer stok) memakai tabel `document_sequences` dengan `SELECT ... FOR UPDATE`. Kunci teknis tetap `uuid` v7 — `document_number` hanya untuk manusia.

---

## 3. Rules

**Rule 1 — Tenant scope tidak pernah implisit di job.** `TenantContext` adalah singleton per request dan **tidak** diwarisi job antrean. Setiap job wajib membawa `tenant_id` eksplisit di payload dan memanggil `TenantContext::set()` di awal `handle()`. Turunkan dari base class `TenantAwareJob`. Ini kesalahan paling umum di aplikasi multi-tenant.

**Rule 2 — Dilarang menyentuh Eloquent modul lain.** Butuh data modul tetangga? Pakai application service publiknya, dengarkan domain event-nya, atau baca read model. Jangan `use Modules\Catalog\Infrastructure\Eloquent\Product` dari dalam `modules/Sales/`.

**Rule 3 — Setiap endpoint tulis menerima `Idempotency-Key`.** Respons diulang dari cache selama 24 jam untuk kunci yang sama. Endpoint tulis tanpa penanganan kunci ini dianggap belum selesai.

**Rule 4 — Jangan pernah UPDATE transaksi final.** Void, retur, dan koreksi dibuat sebagai dokumen baru yang membalik. Stok dan kas dibaca dengan menjumlahkan ledger, bukan dari kolom counter.

**Rule 5 — Snapshot, bukan join, untuk data historis.** Harga, nama produk, HPP, dan tarif pajak disalin ke baris transaksi saat dibuat. Laporan periode lampau tidak boleh berubah ketika master data diubah.

**Rule 6 — Feature flag dicek di server.** Menyembunyikan tombol di React bukan implementasi. Route/API/Filament/job harus menolak dengan `403 MODULE_DISABLED`.

**Rule 7 — Raw query dilarang tanpa review.** Pakai query builder atau prepared statement. Setiap endpoint yang menerima ID wajib punya policy per model plus uji IDOR.

**Rule 8 — Format error API konsisten:** `{"error": {"code", "message", "details"}}`. `code` berupa konstanta SCREAMING_SNAKE, `message` bahasa Indonesia untuk pengguna akhir.

**Rule 9 — Paginasi memakai cursor, bukan offset,** untuk daftar yang sering berubah. Versi API ada di path (`/api/pos/v1`), bukan di header.

**Rule 10 — Migrasi expand lalu contract.** Kolom baru ditambahkan nullable, kode ditulis kompatibel dua arah, kolom lama dihapus di rilis berikutnya. Tidak ada migrasi destruktif dalam satu deploy.

**Rule 11 — Uji isolasi tenant wajib.** Setiap endpoint baru butuh test yang membuat dua tenant dan memverifikasi tidak ada kebocoran. Test ini tidak boleh di-skip.

**Rule 12 — Aksi sensitif wajib tercatat.** Perubahan feature flag, void, diskon manual, override supervisor, impersonasi, dan penyesuaian stok masuk audit log lengkap dengan pelaku, waktu, dan alasan.

**Rule 13 — Rahasia hanya di environment variable.** Tidak pernah masuk repositori. Nomor kartu disimpan maksimal 4 digit terakhir + nomor referensi; tidak pernah PAN penuh atau CVV.

**Rule 14 — Gate CI yang harus lulus:** Pint, ESLint, TypeScript check, PHPStan level 6, Deptrac batas modul, unit + feature test termasuk uji isolasi tenant.

---

## 4. Local Development Environment

**Status: belum berlaku.** Repositori ini baru berisi dokumentasi — belum ada kode aplikasi, `composer.json`, maupun `docker-compose.yml`. Section ini diisi dengan perintah nyata setelah ticket scaffolding selesai; jangan menuliskan perintah yang belum pernah dijalankan.

Prasyarat yang sudah pasti dari stack di atas:

| Kebutuhan | Versi |
|---|---|
| Docker + Docker Compose | terbaru |
| PHP | 8.3 |
| Node.js | LTS aktif (untuk Vite + React 19) |
| PostgreSQL | 16 (via container) |
| Redis | 7 (via container) |

Lingkungan yang direncanakan: `local` (Docker Compose, data seeder) · `staging` (salinan anonim produksi) · `production`.

---

## 5. Documentation Workflow

**Aturan 0 — Semua dokumentasi/konteks WAJIB di `docs/` (versioned di git), JANGAN di memory lokal AI.**
Developer berpindah-pindah PC, dan memory lokal AI tidak ikut di-commit sehingga tidak tersedia di mesin lain. Maka: keputusan, status ticket, hasil analisis, dan konteks apa pun yang perlu bertahan antar-sesi/antar-PC harus ditulis ke file `docs/` yang sesuai (planning, feature, decision-log, atau backlog) — bukan disimpan sebagai memory lokal. AI agent tidak boleh mengandalkan memory lokal sebagai sumber kebenaran untuk project ini.

Never write code without going through this flow first. Do not create `implementation_plan.md` as a standalone artifact — use the official planning files below.

**Step 1 — Backlog (`docs/backlog.md`)**
New bugs/ideas go here without ticket numbers. Status: `READY` or `BLOCKED`. Remove the item once it becomes a planning ticket.

**Step 2 — Planning (`docs/planning/TICKET-NAME.md`)**
Create the plan before writing any code. Use `docs/planning/_template-implementation-plan.md`. Register in `docs/planning/index.md`. Get explicit user approval before executing.

Status lifecycle: `DRAFT` → `REVIEW` → `READY` → `DONE`.
- `DRAFT`: decisions not locked, execution blocked.
- `REVIEW`: decisions locked, waiting for user manual approval — do NOT execute.
- `READY`: user gave explicit written approval in chat — execution allowed.
- `DONE`: implementation and verification complete.

Each plan must include: Business Decision Snapshot, Non-Negotiable Technical Contract (target files, method signatures, integration points, return shapes), Acceptance Test Matrix (min. 1 boundary + 1 failure case), and Out of Scope section.

When a ticket is DONE: move its file to `docs/planning/Ticket-Implemented/`, remove its entry from `docs/planning/index.md` (do not delete the index file itself). Check both locations when numbering new tickets to avoid duplicates.

**Step 3 — Feature Docs (`docs/features/`)**
Every shipped feature needs a doc here, terdaftar di [`docs/features/index.md`](features/index.md). Pakai template lean: `# Judul` → `**Status:** Live` → `## Ringkasan` (2–4 kalimat) → section relevan (Quick Reference / katalog sebagai tabel) → `## Gotchas` (hanya hal non-obvious) → `## Terkait` ([[cross-link]]). Bahasa Indonesia, istilah teknis Inggris.

Aturan gaya (wajib): tanpa emoji dekoratif di header; tanpa commit hash/branch/tanggal di Status; tanpa prosa motivasi/marketing; katalog method/field/kolom selalu berupa tabel; setiap klaim diverifikasi ke kode aktual (yang tak terverifikasi tidak ditulis). Doc tingkat struktural (peta modul, referensi data) berada satu level di atas: `docs/architecture.md` dan `docs/data-model.md`.

**Step 4 — Decision Log (`docs/decision-log.md`)**
Catat keputusan yang memenuhi minimal satu kriteria:
- Gotcha/trap teknis yang tidak bisa diturunkan dari membaca kode
- Trade-off bisnis dengan konsekuensi non-obvious
- Koreksi atas asumsi atau mandat yang sebelumnya salah

Jangan catat: cleanup code, perubahan UI minor, atau hal yang sudah terdokumentasi di `ai-context.md` sendiri. Entry yang SUPERSEDED harus dihapus, bukan dibiarkan.

Format per entry (4 field):

```markdown
## DEC-XXX — Judul singkat

**Decision:** Apa yang diputuskan.

**Why:** Kenapa — constraint, bug, atau asumsi yang salah.

**Impact:** Konsekuensi konkret yang tidak obvious dari kode.

**Tickets:** TICKET-XXX (opsional)
```
