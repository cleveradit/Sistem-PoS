# AI Context & Development Mandates

All AI agents MUST read and follow these rules before proposing or implementing any changes.

> **Orientasi (paham project tanpa baca kode):** baca berurutan — dokumen ini (mandat) → [architecture.md](architecture.md) (peta modul) → [data-model.md](data-model.md) (struktur data) → [features/index.md](features/index.md) (detail per fitur).

---

## 1. Tech Stack (STRICT — no deviations)

<!-- ISI DI SINI: daftar bahasa, framework, library utama, dan constraint versi yang TIDAK BOLEH dilanggar. Contoh: Go 1.23, React 19, PostgreSQL 16, Redis 7. -->

---

## 2. Architecture Mandates

<!-- ISI DI SINI: aturan arsitektur absolut. Contoh: "All domain logic lives in internal/. Nothing domain-related in cmd/." -->
<!-- Contoh: "Cross-module reads are allowed. Cross-module writes are forbidden — call the owning module's Service instead." -->

---

## 3. Rules

<!-- ISI DI SINI: aturan teknis yang harus dipatuhi setiap AI agent. Format: Rule N — judul singkat, lalu 1-3 kalimat penjelasan. -->
<!-- Contoh:
**Rule 1 — Naming Convention:** All service files MUST be suffixed with `_service.go`.
**Rule 2 — Error Handling:** Never panic; always return wrapped errors with context.
-->

---

## 4. Local Development Environment

<!-- ISI DI SINI: cara menjalankan project di local. Sertakan:
- Prasyarat (Docker, Go version, Node version, dll.)
- Cara start services
- Cara rebuild
- Cara clear cache
- Catatan penting (mount, hot reload, dll.)
-->

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
