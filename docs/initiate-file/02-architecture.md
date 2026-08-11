# Architecture Document
## Aplikasi Point of Sale (POS) Multi-Tenant

| Item | Keterangan |
|---|---|
| Dokumen Induk | PRD POS Multi-Tenant v1.0 |
| Versi | 1.0 |
| Tanggal | 10 Agustus 2026 |
| Status | Draft untuk review |
| Penyusun | Radityo |

---

## 1. Prinsip Arsitektur

| Kode | Prinsip | Konsekuensi |
|---|---|---|
| AP-01 | **Modular monolith terlebih dahulu** | Satu deployable, batas modul ditegakkan di level kode. Microservice hanya jika terbukti perlu |
| AP-02 | **Tenant isolation by default** | Tidak ada query yang boleh berjalan tanpa scope tenant. Pelanggaran ditolak di test |
| AP-03 | **Ledger sebagai sumber kebenaran** | Stok dan kas direkonstruksi dari mutasi, bukan disimpan sebagai angka yang diubah langsung |
| AP-04 | **Transaksi bersifat immutable** | Tidak ada UPDATE pada baris penjualan yang sudah final. Koreksi lewat dokumen pembalik |
| AP-05 | **Snapshot pada titik transaksi** | Harga, nama, HPP, dan pajak disalin ke baris transaksi agar laporan historis stabil |
| AP-06 | **Offline first pada terminal kasir** | Kasir menulis ke penyimpanan lokal dahulu, sinkronisasi menyusul |
| AP-07 | **Idempotensi di seluruh jalur tulis** | Setiap operasi tulis membawa kunci idempotensi dari klien |
| AP-08 | **Feature flag ditegakkan di server** | UI menyembunyikan, server menolak. Tidak pernah hanya UI |
| AP-09 | **Baca berat dipisah dari tulis** | Laporan memakai model baca teragregasi, bukan agregasi langsung dari tabel transaksi |
| AP-10 | **Semua aksi sensitif meninggalkan jejak** | Audit log adalah kebutuhan produk, bukan sekadar teknis |

---

## 2. Keputusan Teknologi

| Lapisan | Pilihan | Alasan |
|---|---|---|
| Bahasa dan framework | PHP 8.3, Laravel 12 | Kecepatan pengembangan, ekosistem matang, sudah dikuasai tim |
| Panel admin dan back office | Filament 4 | CRUD kompleks (master data, pengaturan, panel Super Admin) tanpa membangun UI dari nol |
| Terminal kasir | Inertia 2 + React 19 + TypeScript + shadcn/ui, dibungkus PWA | Layar kasir butuh state kaya, responsif, dan kemampuan offline |
| Basis data | PostgreSQL 16 | JSONB untuk snapshot dan pengaturan, partial index, pg_trgm untuk pencarian produk, concurrency baik untuk append ledger, RLS sebagai lapis pertahanan kedua |
| Cache, sesi, antrean | Redis 7 + Laravel Horizon | Cache feature flag, antrean job berat, rate limiting |
| Realtime | Laravel Reverb (WebSocket) | KDS, daftar item habis (86 list), status meja |
| Penyimpanan berkas | S3 compatible (MinIO atau Cloudflare R2) | Gambar produk, lampiran nota, foto kondisi barang servis |
| Pencarian | PostgreSQL pg_trgm pada v1, Meilisearch jika katalog di atas 50.000 item | Hindari komponen tambahan sebelum terbukti perlu |
| Penyimpanan lokal terminal | IndexedDB via Dexie + Service Worker (Workbox) | Master data dan outbox transaksi offline |
| Pencetakan | Print Agent lokal (Go, single binary) + fallback WebUSB / Web Bluetooth | Printer thermal LAN dan USB tidak dapat diakses langsung dari browser secara andal |
| Kontainerisasi | Docker + Docker Compose | Konsisten dengan praktik deployment yang berjalan |
| CI/CD | GitHub Actions | Build image, uji, deploy ke VPS |
| Reverse proxy | Traefik atau Nginx Proxy Manager | TLS otomatis, routing subdomain tenant |
| Observability | Sentry, Laravel Pulse, uptime monitor eksternal | Deteksi error dan degradasi performa |

**Alternatif yang dipertimbangkan dan ditolak:** MySQL 8 (kalah pada JSONB, partial index, dan RLS, meski secara operasional lebih familiar), microservices sejak awal (biaya operasional tidak sepadan untuk tim kecil), Livewire untuk layar kasir (kurang cocok untuk kebutuhan offline dan state kompleks).

---

## 3. Diagram Konteks Sistem (C4 Level 1)

```
                        ┌──────────────────────┐
                        │   Super Admin        │
                        │   (Platform Owner)   │
                        └──────────┬───────────┘
                                   │ kelola tenant, paket, modul
                                   ▼
┌───────────────┐        ┌──────────────────────────┐        ┌────────────────────┐
│ Pemilik Bisnis│───────▶│                          │◀───────│  Payment Gateway   │
│ & Manajer     │        │      PLATFORM POS        │        │  (QRIS dinamis)    │
└───────────────┘        │      MULTI-TENANT        │        └────────────────────┘
                         │                          │
┌───────────────┐        │                          │        ┌────────────────────┐
│ Kasir &       │───────▶│                          │───────▶│  WhatsApp Gateway  │
│ Supervisor    │        │                          │        │  & Email (SMTP)    │
└───────────────┘        │                          │        └────────────────────┘
                         │                          │
┌───────────────┐        │                          │        ┌────────────────────┐
│ Staf Gudang & │───────▶│                          │───────▶│  Object Storage    │
│ Staf Dapur    │        └────────┬─────────────────┘        │  (S3 / R2)         │
└───────────────┘                 │                          └────────────────────┘
                                  │ struk digital, lacak order
                                  ▼
                        ┌──────────────────────┐        ┌────────────────────┐
                        │  Pelanggan Akhir     │        │  Printer Thermal   │
                        │  (tautan publik)     │        │  & Laci Kas        │
                        └──────────────────────┘        └────────────────────┘
```

---

## 4. Diagram Container (C4 Level 2)

```
┌────────────────────────────────────────────────────────────────────────────┐
│                              KLIEN                                          │
├──────────────────┬──────────────────┬──────────────────┬───────────────────┤
│  POS Terminal    │  Back Office     │  Super Admin     │  Public Page      │
│  (PWA)           │  (Filament)      │  (Filament)      │  (Blade ringan)   │
│  React + TS      │  Server rendered │  Server rendered │  struk & lacak    │
│  IndexedDB       │                  │                  │                   │
│  Service Worker  │                  │                  │                   │
└────────┬─────────┴────────┬─────────┴────────┬─────────┴─────────┬─────────┘
         │ REST + Inertia   │ HTTP             │ HTTP              │ HTTP
         │ WebSocket        │                  │                   │
┌────────▼──────────────────▼──────────────────▼───────────────────▼─────────┐
│                        APLIKASI LARAVEL 12                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ HTTP Layer: Route, Middleware (TenantResolver, ModuleGate, Perm)     │  │
│  ├──────────────────────────────────────────────────────────────────────┤  │
│  │ Modules (bounded context)                                            │  │
│  │  Identity │ Tenancy │ Catalog │ Inventory │ Sales │ Payment │ Shift   │  │
│  │  Fnb │ Service │ Customer │ Promotion │ Purchasing │ Employee        │  │
│  │  Expense │ Reporting │ Billing │ Platform                            │  │
│  ├──────────────────────────────────────────────────────────────────────┤  │
│  │ Shared Kernel: Money, TenantContext, ModuleRegistry, AuditRecorder,   │  │
│  │                SequenceGenerator, EventBus                            │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└──────┬────────────────┬─────────────────┬────────────────┬─────────────────┘
       │                │                 │                │
┌──────▼──────┐  ┌──────▼──────┐  ┌───────▼───────┐  ┌─────▼──────────┐
│ PostgreSQL  │  │  Redis      │  │  Horizon      │  │  Reverb        │
│ 16          │  │  cache/queue│  │  worker pool  │  │  WebSocket     │
└─────────────┘  └─────────────┘  └───────────────┘  └────────────────┘
       │
┌──────▼──────────────────────────────────────────────────────────────┐
│ Layanan Eksternal: S3/R2, Payment Gateway, WhatsApp GW, SMTP        │
└─────────────────────────────────────────────────────────────────────┘

              ┌────────────────────────────────────┐
              │  Print Agent (Go, di jaringan lokal│
              │  outlet). Menerima job cetak dari  │
              │  PWA lewat HTTP lokal / WebSocket, │
              │  meneruskan ke printer ESC/POS     │
              └────────────────────────────────────┘
```

---

## 5. Struktur Kode Modular

```
app/
├── Core/                          # shared kernel, tidak boleh bergantung ke modul
│   ├── Money/                     # value object nilai uang
│   ├── Tenancy/                   # TenantContext, global scope, resolver
│   ├── Modules/                   # ModuleRegistry, ModuleGate, dependency resolver
│   ├── Sequencing/                # generator nomor dokumen
│   ├── Auditing/                  # AuditRecorder, trait Auditable
│   ├── Idempotency/               # penanganan kunci idempotensi
│   └── Support/
│
modules/
├── Identity/
│   ├── Domain/
│   │   ├── Entities/              # User, Role, Permission
│   │   ├── ValueObjects/
│   │   ├── Events/
│   │   └── Repositories/          # interface saja
│   ├── Application/
│   │   ├── Commands/              # InviteUser, AssignRole, VerifySupervisorPin
│   │   ├── Queries/
│   │   └── DTOs/
│   ├── Infrastructure/
│   │   ├── Eloquent/              # model + implementasi repository
│   │   ├── Migrations/
│   │   └── Providers/
│   └── Presentation/
│       ├── Http/                  # controller, request, resource
│       ├── Filament/              # resource back office
│       └── routes.php
│
├── Tenancy/          # Tenant, Business, Outlet, Terminal, BusinessModule
├── Catalog/          # Product, Variant, Category, Modifier, Price, Unit
├── Inventory/        # Stock, StockMovement, Opname, Transfer, Recipe, Batch
├── Sales/            # Sale, SaleItem, Return, Void, HeldSale
├── Payment/          # PaymentMethod, SalePayment, Gateway adapter
├── Shift/            # Shift, CashMovement, ZReport
├── Fnb/              # Table, TableSession, KitchenTicket, KDS
├── Service/          # ServiceOrder, StatusFlow, Assignment, Tracking
├── Customer/         # Customer, Group, Loyalty, Receivable
├── Promotion/        # Promotion, Rule engine, Coupon, Voucher
├── Purchasing/       # Supplier, PurchaseOrder, GoodsReceipt, Payable
├── Employee/         # Employee, Attendance, Commission
├── Expense/          # ExpenseCategory, Expense
├── Reporting/        # read model, aggregator job, export
├── Billing/          # Plan, Subscription, Invoice, Entitlement
└── Platform/         # panel Super Admin, announcement, impersonation
```

### 5.1 Aturan Ketergantungan Antar Modul

1. Modul **tidak boleh** memanggil model Eloquent modul lain secara langsung.
2. Komunikasi lintas modul lewat tiga jalur saja: **domain event**, **application service publik**, atau **read model bersama**.
3. `Core` tidak boleh bergantung pada modul mana pun.
4. Arah ketergantungan yang diizinkan (searah, tanpa siklus):

```
Sales ──▶ Catalog, Inventory, Customer, Promotion, Payment, Shift, Tenancy
Fnb ──▶ Sales, Catalog
Service ──▶ Sales, Customer, Employee
Inventory ──▶ Catalog, Tenancy
Purchasing ──▶ Inventory, Catalog
Reporting ──▶ (membaca read model saja, tidak ada modul yang bergantung padanya)
Billing ──▶ Tenancy
Semua ──▶ Core, Identity, Tenancy
```

5. Penegakan otomatis lewat **Deptrac** atau **PHPArkitect** di pipeline CI. Pelanggaran menggagalkan build.

---

## 6. Multi-Tenancy

### 6.1 Strategi

**Single database, shared schema**, dengan kolom `tenant_id` dan `business_id` pada seluruh tabel transaksional.

Pertahanan berlapis:

| Lapis | Mekanisme |
|---|---|
| 1. Konteks permintaan | Middleware `ResolveTenant` menetapkan `TenantContext` dari sesi, subdomain, atau API key |
| 2. Model | `BelongsToTenant` trait memasang global scope dan mengisi `tenant_id` otomatis saat create |
| 3. Basis data | PostgreSQL Row Level Security aktif pada tabel kritis, memakai `SET LOCAL app.tenant_id` per koneksi |
| 4. Pengujian | Test wajib yang membuat dua tenant lalu memverifikasi tidak ada kebocoran pada setiap endpoint |
| 5. Cache | Seluruh kunci cache diawali `t:{tenant_id}:b:{business_id}:` |

### 6.2 Resolusi Konteks

```
Request masuk
  → ResolveTenant middleware
      ├── Sesi web    : dari user aktif + business yang dipilih (disimpan di sesi)
      ├── API publik  : dari api_key → business_id → tenant_id
      └── Terminal PWA: dari device token → terminal → outlet → business → tenant
  → TenantContext::set(tenantId, businessId, outletId, terminalId)
  → EnsureSubscriptionActive middleware
  → ModuleGate middleware  (cek modul yang dibutuhkan route)
  → Permission middleware  (cek izin peran)
  → Controller
```

`TenantContext` adalah singleton per request. Job antrean **tidak** mewarisi context secara otomatis, sehingga setiap job wajib membawa `tenant_id` eksplisit di payload dan memanggil `TenantContext::set()` di awal `handle()`. Ini kesalahan paling umum pada aplikasi multi-tenant dan harus dijaga oleh base class `TenantAwareJob`.

### 6.3 Jalur Migrasi ke Isolasi Lebih Kuat

Koneksi database ditentukan lewat `TenantConnectionResolver` sejak awal, meski v1 selalu mengembalikan koneksi yang sama. Ketika ada tenant enterprise yang butuh database terpisah, cukup ubah resolver tanpa menyentuh kode modul.

---

## 7. Arsitektur Feature Flag

### 7.1 Komponen

```
ModuleRegistry (definisi statis di kode)
   ├── kode modul, nama, deskripsi, grup
   ├── dependensi (array kode modul)
   └── konflik (array kode modul)

PlanEntitlement (data, dikelola Super Admin)
   └── modul apa yang tersedia untuk paket X

BusinessModule (data, dikelola Pemilik)
   └── modul apa yang diaktifkan untuk business Y

ModuleGate (runtime)
   └── isEnabled(code) = registry.exists
                       AND plan.allows(code)
                       AND business.enabled(code)
```

### 7.2 Penegakan

| Titik | Mekanisme |
|---|---|
| Route | Middleware `module:fnb.table` pada grup route |
| Filament | `canAccess()` pada Resource memanggil `ModuleGate` |
| API | Middleware yang sama, mengembalikan `403 MODULE_DISABLED` |
| Form dan UI | Prop `enabledModules` dikirim ke frontend, komponen dirender bersyarat |
| Job dan listener | Guard di awal handler, karena flag bisa berubah setelah job diantrekan |

### 7.3 Cache dan Invalidasi

Konfigurasi modul per business di-cache di Redis dengan kunci `t:{tenant}:b:{business}:modules` tanpa TTL. Invalidasi dilakukan secara eksplisit saat `BusinessModuleChanged` event terbit. Event yang sama juga menyiarkan pesan WebSocket ke terminal aktif agar UI menyegarkan navigasi tanpa perlu login ulang.

### 7.4 Resolver Dependensi

Saat pemilik mengaktifkan modul:

```
enable(code):
  deps = registry.dependenciesOf(code)   # rekursif
  missing = deps yang belum aktif
  if missing tidak kosong:
      tampilkan konfirmasi "Modul ini butuh: A, B. Aktifkan sekaligus?"
  aktifkan code + missing dalam satu transaksi database

disable(code):
  dependents = modul aktif yang bergantung pada code
  if dependents tidak kosong:
      tolak, jelaskan modul mana yang menghalangi
  activeData = hitung data aktif terkait (open bill, order berjalan)
  if activeData > 0:
      tampilkan peringatan berisi jumlah, minta konfirmasi
  nonaktifkan (data tetap disimpan, hanya disembunyikan)
```

---

## 8. Otorisasi dan Supervisor Override

### 8.1 Model Izin

Izin bersifat granular per aksi dengan format `{modul}.{objek}.{aksi}`, contoh `sales.transaction.void`, `sales.discount.manual`, `reporting.profit.view`.

Peran adalah kumpulan izin. Penetapan peran terjadi pada level **user + business**, dengan cakupan opsional ke daftar outlet. Satu user dapat memiliki peran berbeda di business berbeda.

### 8.2 Alur Supervisor Override

```
Kasir memicu aksi terbatas
  → Client memanggil POST /api/authorizations
      body: { action, context: {sale_uuid, amount}, supervisor_pin }
  → Server memverifikasi PIN terhadap user di outlet yang sama
  → Server memeriksa apakah user tersebut punya izin yang diminta
  → Jika lolos, server menerbitkan authorization_token (JWT pendek, TTL 90 detik,
    terikat pada action + context hash + terminal)
  → Client mengirim ulang permintaan aksi disertai authorization_token
  → Server memvalidasi token, menjalankan aksi
  → AuditLog mencatat: actor = kasir, authorized_by = supervisor, reason, context
```

Token berumur pendek dan terikat konteks agar tidak dapat dipakai ulang untuk transaksi lain. PIN tidak pernah dikirim bersama permintaan aksi utamanya.

---

## 9. Alur Transaksi Penjualan

### 9.1 Sequence Online

```
Kasir      PWA           API            Domain              DB           Queue
  │         │              │               │                 │             │
  ├─scan───▶│              │               │                 │             │
  │         ├─cari lokal──▶│ (IndexedDB)   │                 │             │
  │◀────────┤ item masuk keranjang         │                 │             │
  │         │              │               │                 │             │
  ├─bayar──▶│              │               │                 │             │
  │         ├─POST /sales──▶│              │                 │             │
  │         │  Idempotency-Key: {uuid}     │                 │             │
  │         │              ├─cek idempotensi──────────────────▶│           │
  │         │              ├─CreateSaleCommand────▶│          │             │
  │         │              │               ├─validasi modul & izin          │
  │         │              │               ├─hitung ulang harga, diskon,   │
  │         │              │               │  pajak, service charge SERVER-SIDE
  │         │              │               ├─snapshot harga & HPP          │
  │         │              │               ├─BEGIN TX────────▶│             │
  │         │              │               ├─insert sales, sale_items       │
  │         │              │               ├─insert sale_payments           │
  │         │              │               ├─insert stock_movements         │
  │         │              │               ├─update stocks (cache)          │
  │         │              │               ├─COMMIT──────────▶│             │
  │         │              │               ├─event SaleCompleted───────────▶│
  │         │◀─201 + struk─┤               │                 │             │
  │◀─cetak──┤              │               │                 │             │
  │         │              │               │        listener asinkron:      │
  │         │              │               │        - poin loyalitas        │
  │         │              │               │        - agregat harian        │
  │         │              │               │        - notifikasi stok minim │
  │         │              │               │        - webhook keluar        │
```

**Aturan penting:** total transaksi selalu dihitung ulang di server. Nilai dari klien hanya dipakai sebagai pembanding. Jika selisih melebihi toleransi pembulatan, transaksi ditolak dengan kode `TOTAL_MISMATCH` dan detail rincian dikembalikan agar klien dapat menyelaraskan.

### 9.2 Perhitungan Total (Urutan Baku)

```
1. subtotal_item      = qty × harga_satuan_snapshot
2. modifier           = jumlah harga modifier × qty
3. diskon_item        = diskon per baris (nominal atau persen)
4. subtotal           = Σ (subtotal_item + modifier - diskon_item)
5. diskon_transaksi   = diskon level nota (manual + promo otomatis)
6. dasar_pengenaan    = subtotal - diskon_transaksi
7. service_charge     = dasar_pengenaan × rate_service
8. pajak              = (dasar_pengenaan + service_charge_kena_pajak) × rate_pajak
                        atau ekstraksi jika harga inklusif pajak
9. total_sebelum_bulat= dasar_pengenaan + service_charge + pajak
10. pembulatan        = sesuai konfigurasi outlet
11. grand_total       = total_sebelum_bulat + pembulatan
```

Urutan ini dikodekan dalam satu kelas `PricingEngine` yang dipakai bersama oleh server dan klien (klien memakai port TypeScript dengan uji kesetaraan hasil). Duplikasi logika diterima demi kemampuan offline, tetapi kesetaraan diverifikasi lewat golden test dataset yang sama di kedua sisi.

---

## 10. Arsitektur Offline dan Sinkronisasi

### 10.1 Penyimpanan Lokal

| Store (IndexedDB) | Isi | Sifat |
|---|---|---|
| `products` | Katalog, varian, harga, modifier, barcode | Read only, hasil sinkronisasi |
| `customers` | Pelanggan yang sering dipakai (paging + pencarian lazy) | Read mostly |
| `promotions` | Promo aktif dan aturannya | Read only |
| `settings` | Pajak, pembulatan, modul aktif, template struk | Read only |
| `outbox` | Transaksi yang belum terkirim | Tulis lokal, dihapus setelah dikonfirmasi server |
| `sales_local` | Riwayat transaksi terminal untuk cetak ulang | Retensi 7 hari |
| `sync_meta` | Cursor sinkronisasi per entitas | Internal |

### 10.2 Pola Outbox

```
Transaksi selesai di klien
  → tulis ke sales_local (status: pending)
  → tulis ke outbox dengan uuid v7 + payload lengkap + idempotency_key
  → cetak struk (tidak menunggu jaringan)
  → SyncWorker (background):
        while outbox tidak kosong dan online:
            ambil batch (maks 20, urut waktu)
            POST /api/sync/sales  dengan array payload
            untuk tiap hasil:
                 201 / 200 duplicate  → tandai synced, hapus dari outbox
                 422 validasi         → tandai failed, munculkan ke UI, jangan retry otomatis
                 409 konflik bisnis   → tandai needs_review
                 5xx / network        → biarkan di outbox, backoff eksponensial
```

### 10.3 Aturan Konflik

| Situasi | Perlakuan |
|---|---|
| Transaksi terkirim dua kali | Server menolak duplikat berdasarkan `uuid`, mengembalikan 200 dengan data transaksi yang sudah ada. Bukan error |
| Stok menjadi minus setelah sinkron | Transaksi **tetap diterima**. Sistem membuat `stock_movement` dan menandai `negative_stock_flag`. Pemilik menerima notifikasi untuk penyesuaian |
| Harga di server sudah berubah | Transaksi diterima dengan harga snapshot dari klien. Selisih dicatat pada `price_drift` untuk audit |
| Produk sudah diarsipkan di server | Transaksi diterima. Nama dan harga memakai snapshot klien |
| Promo sudah kedaluwarsa saat sinkron | Diterima jika masih berlaku pada `transacted_at` klien, ditolak jika jam klien menyimpang lebih dari toleransi |
| Kupon melebihi kuota pemakaian | Diterima, tetapi ditandai `over_quota` untuk ditinjau. Penjualan tidak boleh dibatalkan sepihak |

**Prinsip yang mendasari semua ini:** penjualan sudah terjadi di dunia nyata dan uang sudah diterima. Server tidak pernah membatalkan fakta itu. Yang dilakukan sistem adalah menerima, menandai, dan memberi jalan koreksi.

### 10.4 Sinkronisasi Master Data (Delta)

```
GET /api/sync/pull?since={cursor}&entities=products,customers,promotions
  → server mengembalikan perubahan sejak cursor, termasuk penghapusan (tombstone)
  → klien menerapkan perubahan, memperbarui cursor
  → dijalankan saat: aplikasi dibuka, setiap 5 menit, saat koneksi pulih,
    dan saat menerima sinyal WebSocket "master_data_changed"
```

Cursor berupa `updated_at` presisi mikrodetik ditambah `id` sebagai tie breaker, bukan nomor halaman.

### 10.5 Batasan Saat Offline

| Fitur | Status Offline |
|---|---|
| Penjualan tunai dan non tunai manual | Tersedia |
| Cetak struk | Tersedia (lewat Print Agent lokal) |
| Buka dan tutup shift | Tersedia |
| Diskon manual dan otorisasi supervisor | Tersedia (PIN diverifikasi terhadap hash yang disinkronkan) |
| Promo otomatis | Tersedia (aturan sudah tersimpan lokal) |
| QRIS dinamis | Tidak tersedia, tombol nonaktif dengan penjelasan |
| Cek dan tebus poin | Tidak tersedia. Poin tetap diperoleh, dihitung saat sinkron |
| Pendaftaran pelanggan baru | Tersedia, dibuat dengan uuid lokal lalu direkonsiliasi |
| Piutang | Tidak tersedia (butuh saldo terkini) |
| Laporan | Terbatas pada data terminal itu sendiri |

---

## 11. Penomoran Dokumen

Kebutuhan: nomor unik, berurutan, terbaca manusia, dan tahan offline.

```
Format: {PREFIX}-{KODE_TERMINAL}-{YYMMDD}-{URUT}
Contoh: INV-K01-260810-0042
```

- Urutan dijaga per kombinasi terminal dan tanggal, dihasilkan **di klien** saat offline.
- Server tidak menolak nomor dari klien, hanya memverifikasi keunikan pada kombinasi `terminal_id + document_number`.
- Untuk dokumen yang dibuat di server (PO, transfer stok), nomor dihasilkan lewat tabel `document_sequences` dengan penguncian baris (`SELECT ... FOR UPDATE`) di dalam transaksi.
- `uuid` (v7, terurut waktu) tetap menjadi kunci teknis. `document_number` hanya untuk manusia.

---

## 12. Arsitektur Pencetakan

Browser tidak dapat berbicara langsung dengan printer thermal LAN secara andal, dan dialog cetak sistem merusak alur kasir.

```
PWA Kasir
   │ 1. membangun payload struk terstruktur (JSON, bukan gambar)
   │
   ├──▶ Print Agent lokal (http://localhost:9100)
   │        - binary Go, berjalan di perangkat kasir atau satu PC di outlet
   │        - mengubah JSON menjadi perintah ESC/POS
   │        - antrean cetak dengan retry
   │        - mendukung printer USB, LAN (port 9100), dan Bluetooth
   │        - membuka laci kas lewat pulse pin
   │        - mendaftarkan diri ke server agar status printer terpantau
   │
   ├──▶ Fallback 1: Web Bluetooth / WebUSB langsung dari PWA (Android, Chrome)
   │
   └──▶ Fallback 2: render HTML struk, tampilkan QR untuk struk digital
```

Template struk disimpan sebagai struktur blok (header, item, total, footer) dengan properti tampil atau sembunyi, bukan sebagai string bebas. Ini memungkinkan render konsisten ke ESC/POS, HTML, dan PDF dari satu sumber.

---

## 13. Arsitektur Pelaporan

Agregasi langsung dari `sales` dan `sale_items` akan melambat setelah beberapa ratus ribu baris.

```
SaleCompleted / SaleVoided / SaleReturned event
   → job UpdateDailyAggregate (antrean terpisah, prioritas rendah)
        → upsert daily_sales_summaries   (per business, outlet, tanggal)
        → upsert product_daily_sales     (per produk, outlet, tanggal)
        → upsert payment_daily_summaries (per metode, outlet, tanggal)
        → upsert cashier_daily_summaries (per user, outlet, tanggal)

Dasbor dan laporan periodik  → baca dari tabel agregat (cepat)
Laporan detail transaksi     → baca dari tabel transaksi dengan filter dan paging
Rekonsiliasi harian (cron)   → hitung ulang agregat H-1 untuk menangkap
                               transaksi offline yang datang terlambat
```

Ekspor besar (di atas 5.000 baris) diproses lewat antrean dan dikirim sebagai tautan unduh, bukan dirender sinkron.

---

## 14. Kejadian Domain (Domain Events)

| Event | Diterbitkan Oleh | Konsumen |
|---|---|---|
| `SaleCompleted` | Sales | Inventory (mutasi stok), Customer (poin), Reporting (agregat), Notification, Webhook |
| `SaleVoided` | Sales | Inventory (pembalikan), Reporting, Audit |
| `SaleReturned` | Sales | Inventory, Customer (koreksi poin), Reporting |
| `StockLevelLow` | Inventory | Notification, Purchasing (saran PO) |
| `StockMovementRecorded` | Inventory | Reporting |
| `ShiftClosed` | Shift | Reporting, Notification (jika ada selisih) |
| `ShiftDiscrepancyDetected` | Shift | Notification ke pemilik, Audit |
| `ServiceOrderStatusChanged` | Service | Notification (WhatsApp ke pelanggan) |
| `KitchenTicketFired` | Fnb | Reverb broadcast ke KDS |
| `BusinessModuleChanged` | Tenancy | Cache invalidation, Reverb broadcast ke terminal |
| `SubscriptionExpired` | Billing | Tenancy (kunci transaksi baru), Notification |
| `SupervisorAuthorizationGranted` | Identity | Audit |

Semua listener yang tidak memengaruhi respons pengguna berjalan asinkron lewat antrean. Listener yang harus konsisten dengan transaksi (mutasi stok) berjalan **di dalam transaksi database yang sama**, bukan lewat antrean.

---

## 15. Strategi Caching

| Data | Lokasi | TTL | Invalidasi |
|---|---|---|---|
| Konfigurasi modul per business | Redis | tanpa TTL | Event `BusinessModuleChanged` |
| Izin peran per user per business | Redis | 1 jam | Saat peran diubah |
| Pengaturan business dan outlet | Redis | tanpa TTL | Saat pengaturan disimpan |
| Katalog produk untuk terminal | IndexedDB klien | delta sync | Sinyal WebSocket |
| Agregat dasbor hari berjalan | Redis | 60 detik | Kedaluwarsa alami |
| Entitlement paket | Redis | 1 jam | Saat langganan berubah |

Seluruh kunci diawali namespace tenant. Tidak ada cache global lintas tenant kecuali katalog modul statis.

---

## 16. Desain API

### 16.1 Lapisan

| API | Konsumen | Autentikasi | Format |
|---|---|---|---|
| Inertia | Back office (Filament dan React) | Sesi | Server driven |
| Terminal API `/api/pos/v1` | PWA kasir | Device token + user token | REST JSON |
| Sync API `/api/sync/v1` | PWA kasir | Device token | REST JSON, batch |
| Public API `/api/v1` | Integrasi pihak ketiga | API key per business | REST JSON |
| Webhook keluar | Sistem pelanggan | HMAC signature | POST JSON |

### 16.2 Konvensi

- Versi di path, bukan di header.
- Seluruh endpoint tulis menerima header `Idempotency-Key`. Respons diulang dari cache selama 24 jam untuk kunci yang sama.
- Format error konsisten:

```json
{
  "error": {
    "code": "MODULE_DISABLED",
    "message": "Modul Manajemen Meja tidak aktif untuk bisnis ini.",
    "details": { "module": "fnb.table" }
  }
}
```

- Paginasi memakai cursor, bukan offset, untuk daftar yang sering berubah.
- Rate limit: 120 permintaan per menit per terminal, 60 per menit per API key, dapat dinaikkan per paket.

---

## 17. Keamanan

| Aspek | Penerapan |
|---|---|
| Transport | TLS 1.2 ke atas, HSTS aktif |
| Kata sandi | Argon2id |
| PIN kasir | Hash bcrypt dengan pepper, dibatasi 5 percobaan lalu terkunci 15 menit |
| Token perangkat | Opaque token, dapat dicabut, terikat pada terminal dan sidik perangkat |
| Otorisasi supervisor | JWT TTL 90 detik, terikat aksi dan konteks |
| Data lokal terminal | IndexedDB dienkripsi dengan kunci turunan dari device token untuk field sensitif |
| Kartu pembayaran | Hanya 4 digit terakhir dan nomor referensi. Tidak pernah PAN penuh atau CVV |
| Injeksi | Query builder dan prepared statement. Raw query dilarang tanpa review |
| Otorisasi objek | Policy per model, ditambah scope tenant. Uji IDOR wajib pada endpoint yang menerima ID |
| Unggahan berkas | Validasi MIME, batas ukuran, disimpan di storage terpisah tanpa eksekusi |
| Impersonasi | Hanya Super Admin, wajib alasan, berdurasi terbatas, selalu tercatat, dan terlihat oleh pemilik tenant di audit log |
| Rahasia | Environment variable, tidak pernah masuk repositori. Rotasi terjadwal |

---

## 18. Deployment

### 18.1 Topologi

```
                    Internet
                       │
                  ┌────▼─────┐
                  │ Traefik  │  TLS, routing, rate limit tepi
                  └────┬─────┘
        ┌──────────────┼──────────────┬───────────────┐
   ┌────▼────┐   ┌─────▼─────┐  ┌─────▼─────┐  ┌──────▼──────┐
   │ app-web │   │ app-web   │  │  reverb   │  │  horizon    │
   │ (php-fpm│   │ (replika) │  │ websocket │  │  worker     │
   │ + nginx)│   │           │  │           │  │             │
   └────┬────┘   └─────┬─────┘  └─────┬─────┘  └──────┬──────┘
        └──────────────┴──────────────┴───────────────┘
                       │
        ┌──────────────┼──────────────┐
   ┌────▼─────┐  ┌─────▼─────┐  ┌─────▼─────┐
   │ postgres │  │   redis   │  │  minio    │
   │  16      │  │     7     │  │  (or R2)  │
   └──────────┘  └───────────┘  └───────────┘
        │
   ┌────▼──────────┐
   │ pgbackrest /  │  backup harian + WAL archiving
   │ cron dump     │
   └───────────────┘
```

### 18.2 Lingkungan

| Lingkungan | Tujuan | Data |
|---|---|---|
| `local` | Pengembangan, Docker Compose | Seeder |
| `staging` | Uji integrasi dan UAT | Salinan anonim dari produksi |
| `production` | Layanan nyata | Data nyata |

### 18.3 Pipeline CI/CD

```
push ke branch
  → lint (Pint, ESLint, TypeScript check)
  → analisis statis (PHPStan level 6, Deptrac batas modul)
  → unit test + feature test (termasuk uji isolasi tenant)
  → build image Docker, tag dengan SHA commit
  → push ke registry

merge ke main
  → deploy staging otomatis
  → migrasi database (dengan pemeriksaan migrasi destruktif)
  → smoke test
  → menunggu persetujuan manual
  → deploy production (rolling, health check sebelum memutar trafik)
  → migrasi production
  → notifikasi hasil
```

Aturan migrasi: **expand lalu contract**. Kolom baru ditambahkan sebagai nullable, kode ditulis kompatibel dua arah, kolom lama baru dihapus pada rilis berikutnya. Ini menghindari downtime dan memungkinkan rollback.

---

## 19. Observability

| Aspek | Alat | Yang Dipantau |
|---|---|---|
| Error aplikasi | Sentry | Exception, dengan tag `tenant_id` dan `business_id` |
| Performa | Laravel Pulse | Query lambat, job lambat, penggunaan cache |
| Antrean | Horizon | Panjang antrean, job gagal, waktu tunggu |
| Uptime | Monitor eksternal | Endpoint health, waktu respons |
| Bisnis | Dasbor internal | Transaksi per menit, kegagalan sinkronisasi, kegagalan cetak |
| Log | Struktur JSON ke stdout | Dikumpulkan oleh Docker logging driver |

Metrik khusus yang wajib ada karena sifat produk ini:

- Jumlah transaksi tertunda di outbox per terminal (mendeteksi terminal yang gagal sinkron diam diam)
- Selisih agregat harian versus hitung ulang (mendeteksi kerusakan data)
- Rasio transaksi dengan `negative_stock_flag`
- Waktu antara `transacted_at` klien dan `synced_at` server

---

## 20. Architecture Decision Records (Ringkas)

| ID | Keputusan | Status | Alasan Utama | Konsekuensi |
|---|---|---|---|---|
| ADR-001 | Modular monolith, bukan microservices | Diterima | Tim kecil, batas domain belum stabil | Batas modul harus dijaga lewat alat otomatis |
| ADR-002 | PostgreSQL 16 sebagai basis data utama | Diterima | JSONB, partial index, pg_trgm, RLS | Tim perlu membiasakan diri dari MySQL |
| ADR-003 | Shared schema dengan kolom tenant | Diterima | Biaya operasional rendah, migrasi tunggal | Risiko kebocoran harus ditutup dengan disiplin uji |
| ADR-004 | Offline first pada terminal, server tidak pernah menolak penjualan | Diterima | Penjualan adalah fakta yang sudah terjadi | Perlu mekanisme penandaan dan koreksi, bukan penolakan |
| ADR-005 | Logika harga diduplikasi ke TypeScript | Diterima | Syarat mutlak kemampuan offline | Wajib golden test dataset bersama untuk menjaga kesetaraan |
| ADR-006 | Print Agent lokal sebagai jalur cetak utama | Diterima | Browser tidak andal untuk printer thermal LAN | Perlu distribusi dan pembaruan binary tambahan |
| ADR-007 | Feature flag ditegakkan di server, bukan hanya UI | Diterima | UI dapat dimanipulasi | Setiap route dan job perlu guard eksplisit |
| ADR-008 | Read model teragregasi untuk laporan | Diterima | Agregasi langsung tidak akan menskala | Perlu job rekonsiliasi untuk data terlambat |
| ADR-009 | Filament untuk back office, React untuk kasir | Diterima | CRUD cepat di satu sisi, UX kaya di sisi lain | Dua paradigma frontend dalam satu proyek |
| ADR-010 | UUID v7 sebagai kunci teknis, nomor dokumen untuk manusia | Diterima | Idempotensi offline dan urutan waktu | Ukuran indeks lebih besar daripada bigint |

---

## 21. Risiko Teknis

| Risiko | Kemungkinan | Dampak | Mitigasi |
|---|---|---|---|
| Logika harga di klien dan server menyimpang | Sedang | Tinggi | Golden test dataset bersama, dijalankan di CI kedua sisi. Server selalu jadi otoritas akhir saat online |
| Kebocoran data antar tenant | Rendah | Sangat tinggi | Global scope, RLS, uji isolasi wajib per endpoint, review khusus untuk raw query |
| Job antrean kehilangan konteks tenant | Sedang | Tinggi | Base class `TenantAwareJob` yang memaksa `tenant_id` di konstruktor |
| Outbox menumpuk tanpa disadari | Sedang | Tinggi | Metrik jumlah antrean per terminal, peringatan di UI kasir dan notifikasi ke pemilik |
| Agregat laporan tidak sinkron dengan transaksi | Sedang | Sedang | Job rekonsiliasi harian yang menghitung ulang H-1 dan mencatat selisih |
| Kombinasi feature flag menghasilkan kondisi tak teruji | Tinggi | Sedang | Uji berbasis preset tipe bisnis, dependensi ketat, larangan kombinasi tak valid |
| Print Agent tidak terpasang atau kedaluwarsa | Sedang | Sedang | Deteksi versi, pembaruan otomatis, fallback struk digital |
| Migrasi destruktif menyebabkan downtime | Rendah | Tinggi | Pola expand-contract, pemeriksa migrasi di CI |
| Pertumbuhan tabel `stock_movements` dan `sale_items` | Tinggi | Sedang | Partisi tabel per bulan setelah melewati ambang, arsip data lama ke tabel dingin |
