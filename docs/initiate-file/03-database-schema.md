# Database Schema
## Aplikasi Point of Sale (POS) Multi-Tenant

| Item | Keterangan |
|---|---|
| Dokumen Induk | PRD POS Multi-Tenant v1.0, Architecture v1.0 |
| Target RDBMS | PostgreSQL 16 |
| Versi | 1.0 |
| Tanggal | 10 Agustus 2026 |

---

## 1. Konvensi

### 1.1 Penamaan

| Aturan | Contoh |
|---|---|
| Nama tabel jamak, snake_case | `sale_items` |
| Kunci utama selalu `id` | `id` |
| Kunci asing `{tabel_tunggal}_id` | `product_id` |
| Boolean berawalan `is_` atau `has_` | `is_active`, `has_variants` |
| Timestamp berakhiran `_at` | `completed_at` |
| Kolom enum disimpan sebagai `VARCHAR` dengan CHECK, bukan tipe ENUM native | mudah ditambah tanpa migrasi berat |
| Tabel pivot memakai urutan alfabet | `product_modifier_group` |

### 1.2 Tipe Data Baku

| Kegunaan | Tipe | Alasan |
|---|---|---|
| Kunci utama teknis | `UUID` (v7) | Terurut waktu, dapat dibuat di klien untuk idempotensi offline |
| Nilai uang | `DECIMAL(15,2)` | Tidak pernah float. Presisi rupiah aman hingga triliunan |
| Persentase | `DECIMAL(7,4)` | Menampung 100,0000 persen dengan presisi empat desimal |
| Kuantitas | `DECIMAL(15,4)` | Mendukung penjualan berbasis berat dan konversi satuan |
| Timestamp | `TIMESTAMPTZ` | Wajib. Zona waktu outlet bervariasi |
| Data fleksibel | `JSONB` | Snapshot, pengaturan, metadata. Diindeks GIN jika dicari |
| Teks pendek | `VARCHAR(n)` dengan batas eksplisit | Mencegah data liar |

### 1.3 Kolom Wajib pada Tabel Transaksional

```sql
id             UUID PRIMARY KEY,
tenant_id      UUID NOT NULL REFERENCES tenants(id),
business_id    UUID NOT NULL REFERENCES businesses(id),
created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
updated_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
deleted_at     TIMESTAMPTZ NULL          -- hanya pada tabel yang mendukung soft delete
```

Index gabungan wajib: `(tenant_id, business_id)` sebagai prefix pada setiap index yang dipakai untuk penelusuran.

### 1.4 Aturan Integritas

1. Tabel transaksi final (`sales`, `sale_items`, `sale_payments`, `stock_movements`, `shifts` yang tertutup) **tidak menerima UPDATE** pada kolom nilai. Ditegakkan lewat trigger.
2. Soft delete hanya berlaku pada master data. Data transaksi tidak pernah dihapus.
3. Setiap nilai uang pada transaksi adalah **snapshot**, bukan referensi ke master.
4. `stock_movements` bersifat append only. `stocks.quantity` adalah cache yang dapat direkonstruksi.

### 1.5 Row Level Security

Diaktifkan pada tabel kritis:

```sql
ALTER TABLE sales ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON sales
  USING (tenant_id = current_setting('app.tenant_id', true)::uuid);
```

Aplikasi memanggil `SET LOCAL app.tenant_id = '...'` pada awal setiap transaksi. Ini lapis pertahanan kedua, bukan pengganti global scope di ORM.

---

## 2. Domain: Platform dan Langganan

### 2.1 `tenants`

Akun pemilik bisnis. Unit isolasi data dan penagihan.

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `name` | VARCHAR(150) | Nama akun atau perusahaan |
| `slug` | VARCHAR(60) | Unik, dipakai untuk subdomain dan tautan publik |
| `owner_user_id` | UUID | FK users. Pemilik utama |
| `status` | VARCHAR(20) | `trial`, `active`, `past_due`, `suspended`, `cancelled` |
| `country_code` | CHAR(2) | Default `ID` |
| `default_currency` | CHAR(3) | Default `IDR` |
| `default_timezone` | VARCHAR(50) | Default `Asia/Jakarta` |
| `trial_ends_at` | TIMESTAMPTZ | Null jika bukan trial |
| `suspended_at` | TIMESTAMPTZ | Null |
| `suspension_reason` | TEXT | Null |
| `purge_after` | TIMESTAMPTZ | Tanggal penghapusan permanen setelah soft delete |
| `settings` | JSONB | Preferensi tingkat tenant |
| `created_at`, `updated_at`, `deleted_at` | TIMESTAMPTZ | |

Index: `UNIQUE(slug)`, `(status)`, `(deleted_at)`.

### 2.2 `plans`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `code` | VARCHAR(40) | Unik. `free`, `starter`, `pro`, `enterprise` |
| `name` | VARCHAR(80) | |
| `description` | TEXT | |
| `price_monthly` | DECIMAL(15,2) | |
| `price_yearly` | DECIMAL(15,2) | |
| `max_businesses` | INT | Null berarti tanpa batas |
| `max_outlets` | INT | Null berarti tanpa batas |
| `max_users` | INT | |
| `max_products` | INT | |
| `data_retention_months` | INT | Retensi data transaksi |
| `is_public` | BOOLEAN | Paket khusus disembunyikan dari halaman harga |
| `sort_order` | INT | |
| `is_active` | BOOLEAN | |

### 2.3 `modules`

Katalog modul, dikelola Super Admin. Definisi selaras dengan `ModuleRegistry` di kode.

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `code` | VARCHAR(60) | Unik. `fnb.table`, `inventory.recipe` |
| `name` | VARCHAR(100) | |
| `description` | TEXT | Manfaat, ditampilkan di halaman jelajah fitur |
| `group` | VARCHAR(40) | `core`, `catalog`, `inventory`, `fnb`, `service`, dst |
| `dependencies` | JSONB | Array kode modul prasyarat |
| `conflicts` | JSONB | Array kode modul yang tidak boleh aktif bersamaan |
| `is_core` | BOOLEAN | Modul inti tidak dapat dimatikan |
| `sort_order` | INT | |

### 2.4 `plan_modules`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `plan_id` | UUID | FK plans |
| `module_id` | UUID | FK modules |

PK gabungan `(plan_id, module_id)`.

### 2.5 `subscriptions`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id` | UUID | FK tenants |
| `plan_id` | UUID | FK plans |
| `status` | VARCHAR(20) | `trialing`, `active`, `past_due`, `cancelled`, `expired` |
| `billing_cycle` | VARCHAR(10) | `monthly`, `yearly` |
| `current_period_start` | TIMESTAMPTZ | |
| `current_period_end` | TIMESTAMPTZ | |
| `cancel_at_period_end` | BOOLEAN | |
| `cancelled_at` | TIMESTAMPTZ | |
| `gateway_reference` | VARCHAR(120) | ID langganan di payment gateway |
| `module_overrides` | JSONB | Modul di luar paket yang diberikan manual oleh Super Admin |

Index: `(tenant_id, status)`, `(current_period_end)`.

### 2.6 `subscription_invoices`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id` | UUID | FK tenants |
| `subscription_id` | UUID | FK subscriptions |
| `invoice_number` | VARCHAR(40) | Unik |
| `period_start`, `period_end` | TIMESTAMPTZ | |
| `subtotal`, `tax_amount`, `total` | DECIMAL(15,2) | |
| `status` | VARCHAR(20) | `draft`, `open`, `paid`, `void`, `uncollectible` |
| `due_at`, `paid_at` | TIMESTAMPTZ | |
| `payment_method` | VARCHAR(40) | |
| `gateway_reference` | VARCHAR(120) | |
| `line_items` | JSONB | Rincian tagihan termasuk prorata |

---

## 3. Domain: Identity dan Akses

### 3.1 `users`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `name` | VARCHAR(120) | |
| `email` | VARCHAR(180) | Unik global |
| `email_verified_at` | TIMESTAMPTZ | |
| `password` | VARCHAR(255) | Argon2id. Null untuk akun yang hanya login PIN |
| `phone` | VARCHAR(25) | |
| `avatar_path` | VARCHAR(255) | |
| `is_super_admin` | BOOLEAN | Default false |
| `two_factor_secret` | TEXT | Terenkripsi |
| `two_factor_confirmed_at` | TIMESTAMPTZ | |
| `last_login_at` | TIMESTAMPTZ | |
| `locale` | VARCHAR(5) | `id`, `en` |
| `is_active` | BOOLEAN | |

Catatan: `users` **tidak** memiliki `tenant_id`. Satu user dapat tergabung di banyak tenant lewat `user_business_roles`. Ini keputusan penting agar konsultan atau akuntan dapat dilibatkan lintas tenant.

Index: `UNIQUE(email)`, `(is_super_admin)`.

### 3.2 `roles`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `business_id` | UUID | Null untuk peran bawaan sistem |
| `code` | VARCHAR(40) | `owner`, `manager`, `supervisor`, `cashier`, `warehouse`, `kitchen` |
| `name` | VARCHAR(80) | |
| `is_system` | BOOLEAN | Peran bawaan tidak dapat dihapus |
| `description` | TEXT | |

Index: `UNIQUE(business_id, code)`.

### 3.3 `permissions`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `code` | VARCHAR(80) | Unik. Format `{modul}.{objek}.{aksi}` |
| `name` | VARCHAR(120) | |
| `module_code` | VARCHAR(60) | Izin tersembunyi jika modulnya mati |
| `group` | VARCHAR(40) | Untuk pengelompokan di UI |
| `is_sensitive` | BOOLEAN | Dapat menjadi target supervisor override |

Contoh isi: `sales.transaction.create`, `sales.transaction.void`, `sales.discount.manual`, `sales.price.override`, `inventory.stock.adjust`, `reporting.profit.view`, `shift.close.force`, `settings.module.manage`.

### 3.4 `role_permissions`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `role_id` | UUID | FK roles |
| `permission_id` | UUID | FK permissions |
| `constraints` | JSONB | Batasan nilai, contoh `{"max_discount_percent": 5}` |

PK gabungan `(role_id, permission_id)`.

### 3.5 `user_business_roles`

Penetapan peran user pada satu business, dengan cakupan outlet opsional.

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `user_id` | UUID | FK users |
| `tenant_id` | UUID | FK tenants |
| `business_id` | UUID | FK businesses |
| `role_id` | UUID | FK roles |
| `outlet_scope` | JSONB | Array outlet_id. Null berarti semua outlet |
| `pin_hash` | VARCHAR(255) | PIN kasir, hash bcrypt. Unik per outlet |
| `pin_failed_attempts` | SMALLINT | |
| `pin_locked_until` | TIMESTAMPTZ | |
| `employee_code` | VARCHAR(30) | Kode karyawan untuk struk dan laporan |
| `is_active` | BOOLEAN | |
| `joined_at`, `left_at` | TIMESTAMPTZ | |

Index: `UNIQUE(user_id, business_id)`, `(business_id, role_id)`, `(tenant_id)`.

### 3.6 `invitations`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id` | UUID | |
| `email` | VARCHAR(180) | |
| `role_id` | UUID | |
| `outlet_scope` | JSONB | |
| `token` | VARCHAR(64) | Unik, di-hash |
| `invited_by` | UUID | FK users |
| `expires_at`, `accepted_at` | TIMESTAMPTZ | |

### 3.7 `devices`

Terminal atau perangkat yang terdaftar dan diotorisasi.

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id` | UUID | |
| `terminal_id` | UUID | FK terminals. Null jika perangkat belum ditugaskan |
| `token_hash` | VARCHAR(255) | Opaque device token |
| `device_name` | VARCHAR(100) | |
| `platform` | VARCHAR(40) | `android`, `ios`, `windows`, `web` |
| `app_version` | VARCHAR(20) | |
| `fingerprint` | VARCHAR(120) | |
| `last_seen_at` | TIMESTAMPTZ | |
| `last_sync_at` | TIMESTAMPTZ | |
| `pending_sync_count` | INT | Dilaporkan klien, untuk pemantauan |
| `is_active` | BOOLEAN | Dapat dicabut jarak jauh |

Index: `(business_id, is_active)`, `(last_sync_at)`.

### 3.8 `supervisor_authorizations`

Catatan setiap otorisasi supervisor, terpisah dari audit log umum karena dipakai pada laporan pengendalian.

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id` | UUID | |
| `terminal_id` | UUID | |
| `requested_by` | UUID | FK users. Kasir |
| `authorized_by` | UUID | FK users. Supervisor |
| `action` | VARCHAR(80) | Kode izin yang diminta |
| `context` | JSONB | `{sale_uuid, amount, item_id}` |
| `reason` | TEXT | Wajib |
| `granted_at` | TIMESTAMPTZ | |
| `token_jti` | VARCHAR(64) | Untuk mencegah pemakaian ulang |

Index: `(business_id, granted_at)`, `(authorized_by)`, `(action)`.

---

## 4. Domain: Tenancy dan Organisasi

### 4.1 `businesses`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id` | UUID | FK tenants |
| `name` | VARCHAR(150) | |
| `slug` | VARCHAR(60) | Unik dalam tenant |
| `type` | VARCHAR(20) | `retail`, `fnb`, `service`, `wholesale`, `mixed`, `minimal` |
| `logo_path` | VARCHAR(255) | |
| `currency` | CHAR(3) | |
| `timezone` | VARCHAR(50) | |
| `tax_number` | VARCHAR(40) | NPWP |
| `address`, `phone`, `email` | | |
| `inventory_valuation` | VARCHAR(10) | `average`, `fifo`. Terkunci setelah transaksi pertama |
| `allow_negative_stock` | BOOLEAN | |
| `day_closing_hour` | SMALLINT | Jam tutup buku, 0 sampai 23. Default 0 |
| `settings` | JSONB | Pengaturan lain yang jarang dikueri |
| `created_at`, `updated_at`, `deleted_at` | | |

Index: `(tenant_id)`, `UNIQUE(tenant_id, slug)`.

### 4.2 `business_modules`

Feature flag per business.

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id` | UUID | |
| `module_code` | VARCHAR(60) | |
| `is_enabled` | BOOLEAN | |
| `enabled_at`, `disabled_at` | TIMESTAMPTZ | |
| `changed_by` | UUID | FK users |
| `config` | JSONB | Konfigurasi spesifik modul |

Index: `UNIQUE(business_id, module_code)`.

### 4.3 `outlets`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id` | UUID | |
| `code` | VARCHAR(20) | Unik dalam business |
| `name` | VARCHAR(120) | |
| `address`, `city`, `postal_code`, `phone` | | |
| `latitude`, `longitude` | DECIMAL(10,7) | Opsional |
| `timezone` | VARCHAR(50) | Override business |
| `operating_hours` | JSONB | Per hari, mendukung jam ganda |
| `receipt_header`, `receipt_footer` | TEXT | |
| `receipt_settings` | JSONB | Blok mana ditampilkan, ukuran kertas |
| `rounding_mode` | VARCHAR(10) | `none`, `up`, `down`, `nearest` |
| `rounding_to` | INT | 100, 500, 1000 |
| `service_charge_rate` | DECIMAL(7,4) | |
| `service_charge_taxable` | BOOLEAN | |
| `is_active` | BOOLEAN | |

Index: `(business_id, is_active)`, `UNIQUE(business_id, code)`.

### 4.4 `tax_rates`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id` | UUID | |
| `outlet_id` | UUID | Null berarti berlaku untuk semua outlet |
| `code` | VARCHAR(20) | `ppn`, `pb1` |
| `name` | VARCHAR(60) | |
| `rate` | DECIMAL(7,4) | 11.0000 untuk 11 persen |
| `is_inclusive` | BOOLEAN | Harga sudah termasuk pajak |
| `applies_to` | VARCHAR(20) | `all`, `category`, `product` |
| `is_active` | BOOLEAN | |
| `effective_from`, `effective_to` | DATE | Untuk perubahan tarif terjadwal |

### 4.5 `terminals`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id` | UUID | |
| `code` | VARCHAR(10) | Prefix nomor struk, contoh `K01` |
| `name` | VARCHAR(80) | |
| `is_active` | BOOLEAN | Harus diaktifkan manajer sebelum dipakai |
| `activated_by`, `activated_at` | | |
| `last_shift_id` | UUID | |
| `settings` | JSONB | Tata letak grid, printer default |

Index: `UNIQUE(outlet_id, code)`.

### 4.6 `printers`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id` | UUID | |
| `name` | VARCHAR(80) | |
| `role` | VARCHAR(20) | `receipt`, `kitchen`, `bar`, `label`, `report` |
| `connection_type` | VARCHAR(20) | `usb`, `lan`, `bluetooth` |
| `address` | VARCHAR(120) | IP atau identifier perangkat |
| `paper_width` | SMALLINT | 58 atau 80 |
| `station_categories` | JSONB | Kategori produk yang dirutekan ke printer ini |
| `cash_drawer_enabled` | BOOLEAN | |
| `is_active` | BOOLEAN | |

### 4.7 `document_sequences`

Penomoran dokumen yang dihasilkan di server.

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id` | UUID | |
| `outlet_id` | UUID | Nullable |
| `document_type` | VARCHAR(30) | `purchase_order`, `transfer`, `adjustment`, `service_order` |
| `period` | VARCHAR(8) | `202608` untuk reset bulanan, kosong jika tanpa reset |
| `prefix` | VARCHAR(20) | |
| `last_number` | INT | Dikunci dengan `SELECT FOR UPDATE` |

Index: `UNIQUE(business_id, outlet_id, document_type, period)`.

---

## 5. Domain: Katalog

### 5.1 `categories`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id` | UUID | |
| `parent_id` | UUID | Self reference, maksimal 3 level |
| `name` | VARCHAR(100) | |
| `code` | VARCHAR(30) | |
| `color` | VARCHAR(7) | Warna tombol pada grid kasir |
| `image_path` | VARCHAR(255) | |
| `kitchen_station` | VARCHAR(40) | Rute tiket dapur untuk modul F&B |
| `sort_order` | INT | |
| `is_active` | BOOLEAN | |

Index: `(business_id, parent_id)`, `(business_id, is_active, sort_order)`.

### 5.2 `products`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id` | UUID | |
| `category_id` | UUID | |
| `type` | VARCHAR(20) | `goods`, `service`, `bundle`, `open_item` |
| `name` | VARCHAR(180) | |
| `sku` | VARCHAR(60) | Unik dalam business jika diisi |
| `description` | TEXT | |
| `image_path` | VARCHAR(255) | |
| `base_price` | DECIMAL(15,2) | Harga jual default |
| `base_cost` | DECIMAL(15,2) | Harga beli terakhir. Referensi, bukan HPP |
| `average_cost` | DECIMAL(15,4) | HPP berjalan, diperbarui saat penerimaan barang |
| `has_variants` | BOOLEAN | |
| `track_stock` | BOOLEAN | False untuk layanan |
| `track_batch` | BOOLEAN | |
| `track_serial` | BOOLEAN | |
| `is_sold_by_weight` | BOOLEAN | |
| `base_unit_id` | UUID | FK units |
| `tax_id` | UUID | Null berarti ikut pajak default outlet |
| `is_tax_exempt` | BOOLEAN | |
| `service_duration_minutes` | INT | Untuk modul jasa |
| `commission_type` | VARCHAR(10) | `none`, `percent`, `fixed` |
| `commission_value` | DECIMAL(15,2) | |
| `min_stock_alert` | DECIMAL(15,4) | Default level business |
| `is_favorite` | BOOLEAN | Tampil di grid cepat kasir |
| `sort_order` | INT | |
| `is_active` | BOOLEAN | |
| `archived_at` | TIMESTAMPTZ | Diarsipkan, tidak dijual lagi |
| `created_at`, `updated_at`, `deleted_at` | | |

Index:
```sql
CREATE INDEX idx_products_business_active ON products (business_id, is_active, sort_order);
CREATE UNIQUE INDEX idx_products_sku ON products (business_id, sku) WHERE sku IS NOT NULL AND deleted_at IS NULL;
CREATE INDEX idx_products_name_trgm ON products USING gin (name gin_trgm_ops);
CREATE INDEX idx_products_category ON products (category_id) WHERE deleted_at IS NULL;
```

### 5.3 `product_variants`

Selalu ada minimal satu baris per produk, termasuk produk tanpa varian (varian default). Ini menyederhanakan seluruh query stok dan penjualan karena selalu menunjuk ke `variant_id`.

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `product_id` | UUID | |
| `sku` | VARCHAR(60) | |
| `name` | VARCHAR(180) | Contoh "Merah / XL" |
| `attributes` | JSONB | `{"warna":"Merah","ukuran":"XL"}` |
| `price` | DECIMAL(15,2) | Null berarti ikut `products.base_price` |
| `cost` | DECIMAL(15,2) | |
| `average_cost` | DECIMAL(15,4) | |
| `image_path` | VARCHAR(255) | |
| `is_default` | BOOLEAN | |
| `is_active` | BOOLEAN | |
| `sort_order` | INT | |

Index: `(product_id, is_active)`, `UNIQUE(business_id, sku) WHERE sku IS NOT NULL`.

### 5.4 `product_barcodes`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id` | UUID | |
| `variant_id` | UUID | FK product_variants |
| `barcode` | VARCHAR(60) | |
| `unit_id` | UUID | Barcode dapat menunjuk satuan berbeda (barcode dus) |
| `is_primary` | BOOLEAN | |

Index: `UNIQUE(business_id, barcode)`, `(variant_id)`.

### 5.5 `units`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id` | UUID | |
| `code` | VARCHAR(20) | `pcs`, `dus`, `kg`, `liter` |
| `name` | VARCHAR(60) | |
| `is_fractional` | BOOLEAN | Boleh desimal (kg, liter) |

### 5.6 `product_units`

Konversi multi satuan.

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `variant_id` | UUID | |
| `unit_id` | UUID | |
| `conversion_factor` | DECIMAL(15,4) | Berapa satuan dasar dalam satu satuan ini |
| `price` | DECIMAL(15,2) | Harga jual pada satuan ini |
| `is_base` | BOOLEAN | Faktor 1 |
| `is_purchase_default` | BOOLEAN | |
| `is_sale_default` | BOOLEAN | |

Index: `UNIQUE(variant_id, unit_id)`.

### 5.7 `price_lists` dan `product_prices`

Menangani harga per outlet, per kanal, per grup pelanggan, dan harga bertingkat kuantitas dalam satu mekanisme.

`price_lists`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id` | UUID | |
| `name` | VARCHAR(80) | Contoh "Harga GoFood", "Harga Reseller" |
| `outlet_id` | UUID | Null berarti semua outlet |
| `channel` | VARCHAR(20) | `dine_in`, `take_away`, `delivery`, `online`, null |
| `customer_group_id` | UUID | Null berarti semua |
| `priority` | INT | Makin besar makin diutamakan saat beberapa cocok |
| `valid_from`, `valid_to` | TIMESTAMPTZ | |
| `is_active` | BOOLEAN | |

`product_prices`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `price_list_id` | UUID | |
| `variant_id` | UUID | |
| `unit_id` | UUID | Nullable |
| `min_quantity` | DECIMAL(15,4) | Default 1. Untuk harga grosir bertingkat |
| `price` | DECIMAL(15,2) | |

Index: `(price_list_id, variant_id, min_quantity)`, `(variant_id)`.

**Resolusi harga** (dijalankan di `PricingEngine`, urutan berhenti pada kecocokan pertama dengan prioritas tertinggi):

```
1. price_lists yang cocok (outlet + channel + customer_group + berlaku hari ini)
   diurutkan priority DESC
2. dalam price list tersebut, cari product_prices dengan min_quantity terbesar
   yang tidak melebihi qty transaksi
3. jika tidak ada, pakai product_variants.price
4. jika null, pakai products.base_price
```

### 5.8 `modifier_groups`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id` | UUID | |
| `name` | VARCHAR(80) | "Level Gula", "Topping" |
| `selection_type` | VARCHAR(10) | `single`, `multiple` |
| `min_selection` | SMALLINT | 0 berarti opsional |
| `max_selection` | SMALLINT | Null berarti tanpa batas |
| `is_required` | BOOLEAN | |
| `sort_order` | INT | |
| `is_active` | BOOLEAN | |

### 5.9 `modifier_options`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `modifier_group_id` | UUID | |
| `name` | VARCHAR(80) | |
| `price_adjustment` | DECIMAL(15,2) | Dapat negatif |
| `linked_variant_id` | UUID | Opsional, agar topping mengurangi stok bahan |
| `quantity_used` | DECIMAL(15,4) | Jumlah bahan yang dipakai |
| `is_default` | BOOLEAN | |
| `is_available` | BOOLEAN | Untuk penandaan habis |
| `sort_order` | INT | |

### 5.10 `product_modifier_group`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `product_id` | UUID | |
| `modifier_group_id` | UUID | |
| `sort_order` | INT | |

PK gabungan.

### 5.11 `bundle_items`

Komposisi produk bertipe `bundle`.

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id` | UUID | |
| `bundle_product_id` | UUID | Produk bertipe bundle |
| `variant_id` | UUID | Komponen |
| `quantity` | DECIMAL(15,4) | |
| `price_allocation` | DECIMAL(15,2) | Alokasi harga untuk laporan per produk |

### 5.12 `recipes` dan `recipe_items`

Komposisi bahan baku satu tingkat.

`recipes`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id` | UUID | |
| `variant_id` | UUID | Item jual |
| `yield_quantity` | DECIMAL(15,4) | Berapa porsi dihasilkan |
| `is_active` | BOOLEAN | |

`recipe_items`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `recipe_id` | UUID | |
| `ingredient_variant_id` | UUID | Bahan baku |
| `quantity` | DECIMAL(15,4) | |
| `unit_id` | UUID | |
| `is_optional` | BOOLEAN | |

Saat penjualan, jika modul `inventory.recipe` aktif dan produk memiliki resep, mutasi stok dibuat untuk **bahan baku**, bukan untuk produk jadi.

---

## 6. Domain: Persediaan

### 6.1 `stocks`

Cache saldo stok. Dapat direkonstruksi penuh dari `stock_movements`.

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id` | UUID | |
| `variant_id` | UUID | |
| `quantity` | DECIMAL(15,4) | Saldo saat ini |
| `reserved_quantity` | DECIMAL(15,4) | Terikat pada open bill atau order jasa |
| `average_cost` | DECIMAL(15,4) | HPP rata rata per outlet |
| `min_stock` | DECIMAL(15,4) | Override level produk |
| `max_stock` | DECIMAL(15,4) | |
| `last_movement_at` | TIMESTAMPTZ | |

Index: `UNIQUE(outlet_id, variant_id)`, `(business_id, quantity)` untuk laporan stok menipis.

### 6.2 `stock_movements`

Buku besar stok. **Append only.** Sumber kebenaran.

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id` | UUID | |
| `variant_id` | UUID | |
| `batch_id` | UUID | Nullable |
| `type` | VARCHAR(30) | `sale`, `sale_return`, `purchase`, `purchase_return`, `adjustment`, `transfer_out`, `transfer_in`, `opname`, `recipe_consumption`, `waste`, `initial` |
| `direction` | VARCHAR(3) | `in`, `out` |
| `quantity` | DECIMAL(15,4) | Selalu positif. Arah ditentukan `direction` |
| `unit_cost` | DECIMAL(15,4) | HPP per unit pada saat mutasi |
| `total_cost` | DECIMAL(15,2) | |
| `balance_after` | DECIMAL(15,4) | Saldo setelah mutasi, untuk audit dan rekonstruksi cepat |
| `reference_type` | VARCHAR(40) | Nama entitas sumber |
| `reference_id` | UUID | ID entitas sumber |
| `notes` | TEXT | |
| `created_by` | UUID | |
| `occurred_at` | TIMESTAMPTZ | Waktu kejadian bisnis, bisa berbeda dari `created_at` untuk transaksi offline |
| `created_at` | TIMESTAMPTZ | |

Index:
```sql
CREATE INDEX idx_sm_variant_outlet ON stock_movements (outlet_id, variant_id, occurred_at DESC);
CREATE INDEX idx_sm_reference ON stock_movements (reference_type, reference_id);
CREATE INDEX idx_sm_business_occurred ON stock_movements (business_id, occurred_at DESC);
```

Kandidat partisi per bulan pada `occurred_at` setelah melewati 10 juta baris.

### 6.3 `stock_adjustments`

Dokumen penyesuaian, membungkus beberapa `stock_movements`.

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id` | UUID | |
| `document_number` | VARCHAR(40) | |
| `reason` | VARCHAR(30) | `damaged`, `lost`, `expired`, `correction`, `promotion`, `internal_use` |
| `notes` | TEXT | |
| `total_cost_impact` | DECIMAL(15,2) | |
| `status` | VARCHAR(20) | `draft`, `posted` |
| `created_by`, `approved_by` | UUID | |
| `posted_at` | TIMESTAMPTZ | |

`stock_adjustment_items`: `id`, `adjustment_id`, `variant_id`, `system_quantity`, `actual_quantity`, `difference`, `unit_cost`, `notes`.

### 6.4 `stock_opnames`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id` | UUID | |
| `document_number` | VARCHAR(40) | |
| `scope_type` | VARCHAR(20) | `full`, `category`, `location` |
| `scope_reference` | JSONB | Daftar kategori atau lokasi |
| `is_blind_count` | BOOLEAN | Penghitung tidak melihat stok sistem |
| `status` | VARCHAR(20) | `draft`, `counting`, `review`, `completed`, `cancelled` |
| `started_at`, `completed_at` | TIMESTAMPTZ | |
| `counted_by`, `approved_by` | UUID | |
| `total_variance_value` | DECIMAL(15,2) | |

`stock_opname_items`: `id`, `opname_id`, `variant_id`, `system_quantity` (dibekukan saat mulai), `counted_quantity`, `variance`, `unit_cost`, `variance_value`, `notes`, `counted_at`, `counted_by`.

### 6.5 `stock_transfers`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id` | UUID | |
| `document_number` | VARCHAR(40) | |
| `from_outlet_id`, `to_outlet_id` | UUID | |
| `status` | VARCHAR(20) | `draft`, `in_transit`, `partial`, `received`, `rejected`, `cancelled` |
| `shipped_at`, `received_at` | TIMESTAMPTZ | |
| `shipped_by`, `received_by` | UUID | |
| `notes` | TEXT | |

`stock_transfer_items`: `id`, `transfer_id`, `variant_id`, `quantity_sent`, `quantity_received`, `unit_cost`, `notes`.

Mutasi stok dibuat dua kali: `transfer_out` saat dikirim, `transfer_in` saat diterima. Selisih antara keduanya tampil sebagai barang dalam perjalanan.

### 6.6 `product_batches`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id` | UUID | |
| `variant_id` | UUID | |
| `batch_number` | VARCHAR(60) | |
| `expiry_date` | DATE | |
| `manufactured_date` | DATE | |
| `quantity` | DECIMAL(15,4) | Sisa dalam batch |
| `unit_cost` | DECIMAL(15,4) | |
| `supplier_id` | UUID | |
| `received_at` | TIMESTAMPTZ | |

Index: `(outlet_id, variant_id, expiry_date)`, `(business_id, expiry_date) WHERE quantity > 0`.

### 6.7 `product_serials`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id` | UUID | |
| `variant_id` | UUID | |
| `serial_number` | VARCHAR(80) | |
| `status` | VARCHAR(20) | `in_stock`, `sold`, `returned`, `damaged` |
| `purchase_reference_id` | UUID | |
| `sale_item_id` | UUID | |
| `warranty_until` | DATE | |

Index: `UNIQUE(business_id, serial_number)`.

---

## 7. Domain: Penjualan

### 7.1 `sales`

Tabel paling kritis. Immutable setelah `status = completed`.

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK. UUID v7 dibuat di klien |
| `tenant_id`, `business_id`, `outlet_id`, `terminal_id` | UUID | |
| `shift_id` | UUID | |
| `document_number` | VARCHAR(40) | `INV-K01-260810-0042` |
| `type` | VARCHAR(20) | `sale`, `return`, `exchange` |
| `status` | VARCHAR(20) | `draft`, `held`, `open_bill`, `completed`, `voided` |
| `channel` | VARCHAR(20) | `dine_in`, `take_away`, `delivery`, `online` |
| `customer_id` | UUID | Nullable |
| `customer_snapshot` | JSONB | Nama dan telepon saat transaksi |
| `cashier_id` | UUID | User yang mengoperasikan |
| `sales_person_id` | UUID | Untuk komisi, bisa berbeda dari kasir |
| `table_id` | UUID | Modul F&B |
| `guest_count` | SMALLINT | |
| `subtotal` | DECIMAL(15,2) | Setelah diskon item, sebelum diskon nota |
| `item_discount_total` | DECIMAL(15,2) | |
| `order_discount_total` | DECIMAL(15,2) | |
| `discount_total` | DECIMAL(15,2) | Jumlah keduanya |
| `taxable_base` | DECIMAL(15,2) | Dasar pengenaan pajak |
| `service_charge_amount` | DECIMAL(15,2) | |
| `tax_amount` | DECIMAL(15,2) | |
| `tax_breakdown` | JSONB | Rincian per jenis pajak |
| `rounding_amount` | DECIMAL(15,2) | Dapat negatif |
| `grand_total` | DECIMAL(15,2) | |
| `paid_amount` | DECIMAL(15,2) | |
| `change_amount` | DECIMAL(15,2) | |
| `due_amount` | DECIMAL(15,2) | Sisa yang menjadi piutang |
| `cost_total` | DECIMAL(15,2) | Total HPP snapshot, untuk laba kotor |
| `gross_profit` | DECIMAL(15,2) | Kolom terhitung untuk kecepatan laporan |
| `notes` | TEXT | |
| `void_reason` | TEXT | |
| `voided_by`, `voided_at` | | |
| `original_sale_id` | UUID | Untuk retur, menunjuk transaksi asal |
| `is_offline_created` | BOOLEAN | |
| `has_negative_stock_flag` | BOOLEAN | Ditandai saat sinkronisasi membuat stok minus |
| `price_drift` | JSONB | Selisih harga klien versus server, jika ada |
| `transacted_at` | TIMESTAMPTZ | Waktu kejadian di klien |
| `completed_at` | TIMESTAMPTZ | |
| `synced_at` | TIMESTAMPTZ | Waktu diterima server |
| `business_date` | DATE | Tanggal buku, memperhitungkan `day_closing_hour` |
| `created_at`, `updated_at` | | |

Index:
```sql
CREATE INDEX idx_sales_business_date ON sales (business_id, business_date DESC, status);
CREATE INDEX idx_sales_outlet_date ON sales (outlet_id, business_date DESC);
CREATE UNIQUE INDEX idx_sales_docnum ON sales (terminal_id, document_number);
CREATE INDEX idx_sales_customer ON sales (customer_id, completed_at DESC) WHERE customer_id IS NOT NULL;
CREATE INDEX idx_sales_shift ON sales (shift_id);
CREATE INDEX idx_sales_cashier ON sales (cashier_id, business_date);
CREATE INDEX idx_sales_open ON sales (outlet_id, status) WHERE status IN ('held','open_bill');
```

`business_date` dihitung saat penyimpanan, bukan saat kueri. Ini penting agar laporan usaha yang tutup pukul 03.00 tidak terpotong tengah malam.

### 7.2 `sale_items`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `sale_id` | UUID | |
| `line_number` | SMALLINT | Urutan tampil |
| `variant_id` | UUID | Nullable untuk open item |
| `product_snapshot` | JSONB | `{name, sku, category_name, unit}` |
| `quantity` | DECIMAL(15,4) | |
| `unit_id` | UUID | |
| `unit_conversion_factor` | DECIMAL(15,4) | Untuk konversi ke satuan dasar saat mutasi stok |
| `unit_price` | DECIMAL(15,2) | Snapshot harga jual |
| `original_price` | DECIMAL(15,2) | Harga sebelum override, untuk laporan |
| `price_source` | VARCHAR(30) | `base`, `variant`, `price_list`, `manual` |
| `modifier_total` | DECIMAL(15,2) | |
| `gross_amount` | DECIMAL(15,2) | (unit_price + modifier) × qty |
| `discount_amount` | DECIMAL(15,2) | |
| `discount_type` | VARCHAR(10) | `fixed`, `percent` |
| `discount_value` | DECIMAL(15,2) | Nilai yang diinput |
| `discount_source` | VARCHAR(20) | `manual`, `promotion`, `coupon`, `member` |
| `promotion_id` | UUID | Nullable |
| `net_amount` | DECIMAL(15,2) | |
| `tax_amount` | DECIMAL(15,2) | |
| `unit_cost` | DECIMAL(15,4) | Snapshot HPP |
| `total_cost` | DECIMAL(15,2) | |
| `batch_id`, `serial_id` | UUID | Nullable |
| `notes` | TEXT | "tanpa es" |
| `kitchen_status` | VARCHAR(20) | `pending`, `fired`, `preparing`, `ready`, `served` |
| `fired_at` | TIMESTAMPTZ | |
| `voided_at`, `voided_by`, `void_reason` | | Void per item sebelum pembayaran |

Index: `(sale_id, line_number)`, `(variant_id, created_at)`, `(promotion_id)`.

### 7.3 `sale_item_modifiers`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `sale_item_id` | UUID | |
| `modifier_option_id` | UUID | Nullable jika opsi sudah dihapus |
| `modifier_snapshot` | JSONB | `{group_name, option_name}` |
| `price_adjustment` | DECIMAL(15,2) | |
| `quantity` | DECIMAL(15,4) | |

### 7.4 `sale_payments`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `sale_id` | UUID | |
| `payment_method_id` | UUID | |
| `method_snapshot` | JSONB | `{name, type}` |
| `amount` | DECIMAL(15,2) | |
| `tendered_amount` | DECIMAL(15,2) | Uang diterima, untuk tunai |
| `change_amount` | DECIMAL(15,2) | |
| `fee_amount` | DECIMAL(15,2) | MDR atau biaya admin |
| `reference_number` | VARCHAR(80) | Nomor approval EDC atau referensi transfer |
| `card_last_four` | CHAR(4) | Hanya 4 digit terakhir |
| `gateway_reference` | VARCHAR(120) | |
| `gateway_status` | VARCHAR(30) | `pending`, `settled`, `failed`, `expired` |
| `paid_at` | TIMESTAMPTZ | |
| `is_refunded` | BOOLEAN | |
| `refunded_amount` | DECIMAL(15,2) | |

Index: `(sale_id)`, `(business_id, paid_at)`, `(payment_method_id, paid_at)`.

### 7.5 `payment_methods`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id` | UUID | |
| `outlet_id` | UUID | Null berarti semua outlet |
| `code` | VARCHAR(30) | |
| `name` | VARCHAR(60) | "Tunai", "QRIS BCA", "Debit Mandiri" |
| `type` | VARCHAR(20) | `cash`, `card`, `qris`, `transfer`, `ewallet`, `voucher`, `points`, `credit` |
| `requires_reference` | BOOLEAN | |
| `opens_cash_drawer` | BOOLEAN | |
| `fee_type` | VARCHAR(10) | `none`, `percent`, `fixed` |
| `fee_value` | DECIMAL(15,4) | |
| `fee_borne_by` | VARCHAR(10) | `merchant`, `customer` |
| `gateway_provider` | VARCHAR(30) | `midtrans`, `xendit`, null |
| `is_active` | BOOLEAN | |
| `sort_order` | INT | |

### 7.6 `sale_returns`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id` | UUID | |
| `original_sale_id` | UUID | |
| `return_sale_id` | UUID | Transaksi retur yang dibuat di tabel `sales` |
| `document_number` | VARCHAR(40) | |
| `reason` | VARCHAR(30) | `defective`, `wrong_item`, `customer_change_mind`, `expired` |
| `refund_type` | VARCHAR(20) | `cash`, `store_credit`, `exchange`, `original_method` |
| `refund_amount` | DECIMAL(15,2) | |
| `restock` | BOOLEAN | Apakah barang dikembalikan ke stok |
| `notes` | TEXT | |
| `processed_by`, `authorized_by` | UUID | |
| `processed_at` | TIMESTAMPTZ | |

`sale_return_items`: `id`, `return_id`, `original_sale_item_id`, `variant_id`, `quantity`, `unit_price`, `refund_amount`, `restock`, `condition`.

### 7.7 `held_sales`

Transaksi yang disimpan sementara. Terpisah dari `sales` agar tabel utama tetap bersih dari data tidak final.

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id`, `terminal_id` | UUID | |
| `label` | VARCHAR(60) | "Pak Budi", "Meja 5" |
| `cashier_id` | UUID | |
| `customer_id` | UUID | |
| `payload` | JSONB | Isi keranjang lengkap |
| `item_count` | SMALLINT | |
| `estimated_total` | DECIMAL(15,2) | |
| `held_at` | TIMESTAMPTZ | |
| `expires_at` | TIMESTAMPTZ | Dibersihkan otomatis setelah tutup shift |

---

## 8. Domain: Shift dan Kas

### 8.1 `shifts`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id`, `terminal_id` | UUID | |
| `shift_number` | VARCHAR(40) | |
| `cashier_id` | UUID | |
| `status` | VARCHAR(20) | `open`, `closed`, `force_closed` |
| `opening_balance` | DECIMAL(15,2) | Modal awal laci |
| `opened_at` | TIMESTAMPTZ | |
| `closed_at` | TIMESTAMPTZ | |
| `expected_cash` | DECIMAL(15,2) | Modal + tunai masuk - tunai keluar |
| `counted_cash` | DECIMAL(15,2) | Hasil hitung fisik |
| `difference` | DECIMAL(15,2) | Positif berarti lebih, negatif berarti kurang |
| `difference_reason` | TEXT | |
| `denominations` | JSONB | `{"100000":5,"50000":12,...}` |
| `sales_count` | INT | |
| `sales_total` | DECIMAL(15,2) | |
| `payment_summary` | JSONB | Total per metode pembayaran |
| `void_count`, `void_total` | | |
| `discount_total` | DECIMAL(15,2) | |
| `return_count`, `return_total` | | |
| `closed_by` | UUID | Bisa berbeda dari kasir jika ditutup paksa |
| `approved_by` | UUID | Supervisor yang menyetujui selisih |
| `business_date` | DATE | |

Index: `UNIQUE(terminal_id) WHERE status = 'open'` (memastikan satu shift terbuka per terminal), `(outlet_id, business_date)`, `(cashier_id, opened_at DESC)`.

### 8.2 `cash_movements`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id`, `shift_id` | UUID | |
| `type` | VARCHAR(20) | `cash_in`, `cash_out`, `bank_deposit`, `petty_cash`, `opening`, `closing` |
| `amount` | DECIMAL(15,2) | Selalu positif |
| `direction` | VARCHAR(3) | `in`, `out` |
| `reason` | VARCHAR(120) | |
| `reference_type`, `reference_id` | | Misal menunjuk ke `expenses` |
| `attachment_path` | VARCHAR(255) | Foto nota |
| `created_by`, `authorized_by` | UUID | |
| `occurred_at` | TIMESTAMPTZ | |

---

## 9. Domain: F&B

### 9.1 `dining_areas`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id` | UUID | |
| `name` | VARCHAR(60) | "Indoor", "Rooftop" |
| `sort_order` | INT | |
| `is_active` | BOOLEAN | |

### 9.2 `dining_tables`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id`, `area_id` | UUID | |
| `code` | VARCHAR(20) | "A1", "T-05" |
| `name` | VARCHAR(60) | |
| `capacity` | SMALLINT | |
| `status` | VARCHAR(20) | `available`, `occupied`, `reserved`, `cleaning`, `disabled` |
| `current_sale_id` | UUID | Open bill yang sedang berjalan |
| `position_x`, `position_y` | INT | Koordinat pada denah |
| `shape` | VARCHAR(20) | `square`, `round`, `rectangle` |
| `is_active` | BOOLEAN | |

Index: `UNIQUE(outlet_id, code)`, `(outlet_id, status)`.

### 9.3 `table_sessions`

Riwayat pemakaian meja, untuk laporan durasi dan perputaran meja.

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id`, `table_id` | UUID | |
| `sale_id` | UUID | |
| `guest_count` | SMALLINT | |
| `opened_by` | UUID | |
| `opened_at`, `closed_at` | TIMESTAMPTZ | |
| `duration_minutes` | INT | Terhitung saat tutup |
| `moved_from_table_id` | UUID | Jejak pindah meja |
| `merged_into_sale_id` | UUID | Jejak gabung meja |

### 9.4 `kitchen_tickets`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id` | UUID | |
| `sale_id` | UUID | |
| `ticket_number` | VARCHAR(20) | Nomor pendek untuk dipanggil |
| `station` | VARCHAR(40) | `kitchen`, `bar`, `dessert` |
| `status` | VARCHAR(20) | `new`, `preparing`, `ready`, `served`, `cancelled` |
| `table_code` | VARCHAR(20) | Snapshot |
| `channel` | VARCHAR(20) | |
| `fired_at` | TIMESTAMPTZ | |
| `started_at`, `ready_at`, `served_at` | TIMESTAMPTZ | |
| `prep_duration_seconds` | INT | Terhitung, untuk laporan kecepatan dapur |
| `printed_at` | TIMESTAMPTZ | |
| `notes` | TEXT | |

Index: `(outlet_id, station, status, fired_at)`.

`kitchen_ticket_items`: `id`, `ticket_id`, `sale_item_id`, `product_name`, `quantity`, `modifiers_text`, `notes`, `status`.

### 9.5 `table_reservations`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id`, `table_id` | UUID | |
| `customer_name`, `customer_phone` | | |
| `guest_count` | SMALLINT | |
| `reserved_at` | TIMESTAMPTZ | |
| `duration_minutes` | INT | |
| `status` | VARCHAR(20) | `booked`, `seated`, `no_show`, `cancelled` |
| `notes` | TEXT | |

---

## 10. Domain: Jasa

### 10.1 `service_status_flows`

Alur status yang dapat dikonfigurasi per business.

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id` | UUID | |
| `name` | VARCHAR(60) | "Alur Laundry" |
| `is_default` | BOOLEAN | |
| `statuses` | JSONB | Array berurutan: `[{code, label, color, is_final, notify_customer, deducts_stock}]` |

### 10.2 `service_orders`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id` | UUID | |
| `document_number` | VARCHAR(40) | |
| `sale_id` | UUID | Transaksi penjualan terkait |
| `customer_id` | UUID | |
| `customer_snapshot` | JSONB | |
| `flow_id` | UUID | FK service_status_flows |
| `current_status` | VARCHAR(40) | |
| `priority` | VARCHAR(10) | `normal`, `express` |
| `received_at` | TIMESTAMPTZ | |
| `estimated_ready_at` | TIMESTAMPTZ | |
| `ready_at`, `picked_up_at` | TIMESTAMPTZ | |
| `assigned_to` | UUID | Teknisi atau pekerja |
| `object_details` | JSONB | `{brand, model, plate_number, initial_condition}` |
| `total_amount` | DECIMAL(15,2) | |
| `down_payment` | DECIMAL(15,2) | |
| `remaining_amount` | DECIMAL(15,2) | |
| `tracking_token` | VARCHAR(40) | Unik, untuk halaman lacak publik |
| `delivery_type` | VARCHAR(20) | `pickup`, `delivery`, `self` |
| `delivery_address` | TEXT | |
| `delivery_fee` | DECIMAL(15,2) | |
| `notes` | TEXT | |

Index: `(outlet_id, current_status, received_at)`, `UNIQUE(tracking_token)`, `(customer_id)`, `(assigned_to, current_status)`.

### 10.3 `service_order_items`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `service_order_id` | UUID | |
| `variant_id` | UUID | |
| `product_snapshot` | JSONB | |
| `quantity` | DECIMAL(15,4) | Kg, meter, jam, atau pcs |
| `unit_id` | UUID | |
| `unit_price` | DECIMAL(15,2) | |
| `discount_amount` | DECIMAL(15,2) | |
| `net_amount` | DECIMAL(15,2) | |
| `assigned_to` | UUID | Pekerja per item, untuk komisi |
| `status` | VARCHAR(40) | Status per item jika diperlukan |
| `notes` | TEXT | |

### 10.4 `service_order_status_histories`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `service_order_id` | UUID | |
| `from_status`, `to_status` | VARCHAR(40) | |
| `changed_by` | UUID | |
| `notes` | TEXT | |
| `notified_customer` | BOOLEAN | |
| `changed_at` | TIMESTAMPTZ | |

### 10.5 `service_order_attachments`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `service_order_id` | UUID | |
| `type` | VARCHAR(20) | `before`, `after`, `document` |
| `file_path` | VARCHAR(255) | |
| `caption` | VARCHAR(180) | |
| `uploaded_by` | UUID | |

---

## 11. Domain: Pelanggan

### 11.1 `customer_groups`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id` | UUID | |
| `name` | VARCHAR(60) | "Umum", "Member", "Reseller" |
| `default_discount_percent` | DECIMAL(7,4) | |
| `price_list_id` | UUID | |
| `credit_limit` | DECIMAL(15,2) | Default untuk anggota grup |
| `points_multiplier` | DECIMAL(7,4) | |
| `is_default` | BOOLEAN | |

### 11.2 `customers`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id` | UUID | |
| `group_id` | UUID | |
| `code` | VARCHAR(30) | Nomor member |
| `name` | VARCHAR(150) | |
| `phone` | VARCHAR(25) | |
| `email` | VARCHAR(180) | |
| `birth_date` | DATE | |
| `gender` | VARCHAR(10) | |
| `address`, `city`, `postal_code` | | |
| `tax_number` | VARCHAR(40) | |
| `credit_limit` | DECIMAL(15,2) | |
| `credit_used` | DECIMAL(15,2) | Cache saldo piutang |
| `payment_term_days` | SMALLINT | |
| `points_balance` | DECIMAL(15,2) | Cache, sumber kebenaran di `point_transactions` |
| `store_credit_balance` | DECIMAL(15,2) | Saldo dari retur |
| `total_spent` | DECIMAL(15,2) | Cache untuk segmentasi |
| `transaction_count` | INT | |
| `last_transaction_at` | TIMESTAMPTZ | |
| `notes` | TEXT | |
| `tags` | JSONB | |
| `is_active` | BOOLEAN | |
| `created_at`, `updated_at`, `deleted_at` | | |

Index: `(business_id, phone)`, `(business_id, name)` dengan trgm, `UNIQUE(business_id, code) WHERE code IS NOT NULL`, `(group_id)`.

### 11.3 `point_transactions`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `customer_id` | UUID | |
| `type` | VARCHAR(20) | `earned`, `redeemed`, `expired`, `adjusted`, `reversed` |
| `points` | DECIMAL(15,2) | Positif atau negatif |
| `balance_after` | DECIMAL(15,2) | |
| `reference_type`, `reference_id` | | |
| `expires_at` | TIMESTAMPTZ | |
| `notes` | TEXT | |
| `occurred_at` | TIMESTAMPTZ | |

### 11.4 `receivables`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id`, `customer_id` | UUID | |
| `sale_id` | UUID | |
| `service_order_id` | UUID | |
| `document_number` | VARCHAR(40) | |
| `amount` | DECIMAL(15,2) | Nilai awal |
| `paid_amount` | DECIMAL(15,2) | |
| `remaining_amount` | DECIMAL(15,2) | |
| `status` | VARCHAR(20) | `open`, `partial`, `paid`, `overdue`, `written_off` |
| `due_date` | DATE | |
| `issued_at`, `settled_at` | TIMESTAMPTZ | |

Index: `(customer_id, status)`, `(business_id, due_date) WHERE status IN ('open','partial','overdue')`.

### 11.5 `receivable_payments`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `receivable_id` | UUID | |
| `payment_method_id` | UUID | |
| `amount` | DECIMAL(15,2) | |
| `reference_number` | VARCHAR(80) | |
| `shift_id` | UUID | Agar masuk rekonsiliasi kas jika dibayar tunai |
| `received_by` | UUID | |
| `paid_at` | TIMESTAMPTZ | |
| `notes` | TEXT | |

---

## 12. Domain: Promosi

### 12.1 `promotions`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id` | UUID | |
| `name` | VARCHAR(120) | |
| `type` | VARCHAR(30) | `fixed_discount`, `percent_discount`, `special_price`, `buy_x_get_y`, `bundle`, `free_shipping` |
| `scope` | VARCHAR(20) | `item`, `order` |
| `discount_value` | DECIMAL(15,2) | |
| `max_discount_amount` | DECIMAL(15,2) | Batas atas untuk diskon persentase |
| `min_purchase_amount` | DECIMAL(15,2) | |
| `min_quantity` | DECIMAL(15,4) | |
| `buy_quantity`, `get_quantity` | DECIMAL(15,4) | Untuk tipe beli X gratis Y |
| `get_variant_id` | UUID | Item yang digratiskan |
| `is_stackable` | BOOLEAN | Boleh digabung promo lain |
| `priority` | INT | Urutan penerapan |
| `starts_at`, `ends_at` | TIMESTAMPTZ | |
| `active_days` | JSONB | `[1,2,3,4,5]` untuk Senin sampai Jumat |
| `active_time_start`, `active_time_end` | TIME | Happy hour |
| `usage_limit_total` | INT | |
| `usage_limit_per_customer` | INT | |
| `usage_count` | INT | Cache |
| `applies_to_channels` | JSONB | |
| `outlet_scope` | JSONB | |
| `customer_group_scope` | JSONB | |
| `requires_coupon` | BOOLEAN | |
| `is_active` | BOOLEAN | |

Index: `(business_id, is_active, starts_at, ends_at)`.

### 12.2 `promotion_targets`

Produk atau kategori yang menjadi sasaran promo.

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `promotion_id` | UUID | |
| `target_type` | VARCHAR(20) | `product`, `variant`, `category`, `all` |
| `target_id` | UUID | |
| `is_exclusion` | BOOLEAN | True berarti dikecualikan dari promo |

### 12.3 `coupons`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `promotion_id` | UUID | |
| `code` | VARCHAR(40) | |
| `usage_limit` | INT | |
| `usage_count` | INT | |
| `customer_id` | UUID | Null berarti dapat dipakai siapa saja |
| `valid_from`, `valid_to` | TIMESTAMPTZ | |
| `is_active` | BOOLEAN | |

Index: `UNIQUE(business_id, code)`.

### 12.4 `promotion_usages`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `promotion_id` | UUID | |
| `coupon_id` | UUID | |
| `sale_id` | UUID | |
| `customer_id` | UUID | |
| `discount_amount` | DECIMAL(15,2) | |
| `is_over_quota` | BOOLEAN | Ditandai saat sinkronisasi offline melewati kuota |
| `used_at` | TIMESTAMPTZ | |

### 12.5 `vouchers`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id` | UUID | |
| `code` | VARCHAR(40) | |
| `face_value` | DECIMAL(15,2) | |
| `remaining_value` | DECIMAL(15,2) | Mendukung pemakaian sebagian |
| `sold_sale_id` | UUID | Transaksi saat voucher dijual |
| `customer_id` | UUID | |
| `status` | VARCHAR(20) | `active`, `partially_used`, `used`, `expired`, `void` |
| `expires_at` | TIMESTAMPTZ | |

---

## 13. Domain: Pembelian

### 13.1 `suppliers`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id` | UUID | |
| `code` | VARCHAR(30) | |
| `name` | VARCHAR(150) | |
| `contact_person`, `phone`, `email`, `address` | | |
| `tax_number` | VARCHAR(40) | |
| `payment_term_days` | SMALLINT | |
| `payable_balance` | DECIMAL(15,2) | Cache |
| `notes` | TEXT | |
| `is_active` | BOOLEAN | |

### 13.2 `purchase_orders`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id`, `supplier_id` | UUID | |
| `document_number` | VARCHAR(40) | |
| `status` | VARCHAR(20) | `draft`, `sent`, `partial`, `received`, `cancelled` |
| `order_date` | DATE | |
| `expected_date` | DATE | |
| `subtotal`, `discount_amount`, `tax_amount`, `shipping_cost`, `other_cost`, `grand_total` | DECIMAL(15,2) | |
| `notes` | TEXT | |
| `created_by`, `approved_by` | UUID | |

`purchase_order_items`: `id`, `purchase_order_id`, `variant_id`, `quantity`, `unit_id`, `conversion_factor`, `unit_cost`, `discount_amount`, `tax_amount`, `subtotal`, `received_quantity`, `notes`.

### 13.3 `goods_receipts`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id`, `supplier_id` | UUID | |
| `purchase_order_id` | UUID | Nullable, penerimaan tanpa PO diperbolehkan |
| `document_number` | VARCHAR(40) | |
| `supplier_invoice_number` | VARCHAR(60) | |
| `received_date` | DATE | |
| `subtotal`, `discount_amount`, `tax_amount`, `shipping_cost`, `grand_total` | DECIMAL(15,2) | |
| `landed_cost_allocation` | VARCHAR(20) | `none`, `by_value`, `by_quantity` |
| `payment_status` | VARCHAR(20) | `unpaid`, `partial`, `paid` |
| `due_date` | DATE | |
| `received_by` | UUID | |
| `notes` | TEXT | |

`goods_receipt_items`: `id`, `goods_receipt_id`, `purchase_order_item_id`, `variant_id`, `quantity`, `unit_id`, `conversion_factor`, `unit_cost`, `landed_cost_per_unit`, `final_unit_cost`, `batch_number`, `expiry_date`, `subtotal`.

Penerimaan barang memicu: mutasi stok `purchase`, pembaruan `average_cost`, pembuatan batch jika modul aktif, dan pembuatan `payables`.

### 13.4 `purchase_returns`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id`, `supplier_id` | UUID | |
| `goods_receipt_id` | UUID | |
| `document_number` | VARCHAR(40) | |
| `reason` | VARCHAR(30) | |
| `total_amount` | DECIMAL(15,2) | |
| `refund_type` | VARCHAR(20) | `cash`, `credit_note`, `replacement` |
| `status` | VARCHAR(20) | |
| `returned_at` | TIMESTAMPTZ | |

### 13.5 `payables` dan `payable_payments`

`payables`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `supplier_id` | UUID | |
| `goods_receipt_id` | UUID | |
| `document_number` | VARCHAR(40) | |
| `amount`, `paid_amount`, `remaining_amount` | DECIMAL(15,2) | |
| `status` | VARCHAR(20) | `open`, `partial`, `paid`, `overdue` |
| `due_date` | DATE | |

`payable_payments`: `id`, `payable_id`, `amount`, `payment_method`, `reference_number`, `paid_at`, `paid_by`, `notes`.

---

## 14. Domain: Karyawan dan Pengeluaran

### 14.1 `employees`

Data ketenagakerjaan, terpisah dari `users` karena tidak semua karyawan punya akun login.

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id` | UUID | |
| `user_id` | UUID | Nullable |
| `code` | VARCHAR(30) | |
| `name` | VARCHAR(120) | |
| `phone`, `email`, `address` | | |
| `position` | VARCHAR(60) | |
| `employment_type` | VARCHAR(20) | `full_time`, `part_time`, `contract` |
| `joined_at`, `resigned_at` | DATE | |
| `base_salary` | DECIMAL(15,2) | Referensi, bukan payroll |
| `is_active` | BOOLEAN | |

### 14.2 `attendances`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id`, `employee_id` | UUID | |
| `date` | DATE | |
| `clock_in_at`, `clock_out_at` | TIMESTAMPTZ | |
| `clock_in_terminal_id`, `clock_out_terminal_id` | UUID | |
| `work_duration_minutes` | INT | |
| `status` | VARCHAR(20) | `present`, `late`, `absent`, `leave`, `holiday` |
| `notes` | TEXT | |

Index: `UNIQUE(employee_id, date)`.

### 14.3 `commission_rules`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id` | UUID | |
| `name` | VARCHAR(80) | |
| `applies_to` | VARCHAR(20) | `all`, `category`, `product`, `employee` |
| `target_id` | UUID | |
| `calculation_type` | VARCHAR(20) | `percent_of_sale`, `percent_of_profit`, `fixed_per_item` |
| `value` | DECIMAL(15,4) | |
| `min_sales_threshold` | DECIMAL(15,2) | |
| `priority` | INT | |
| `is_active` | BOOLEAN | |

### 14.4 `commissions`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `employee_id` | UUID | |
| `sale_id`, `sale_item_id`, `service_order_id` | UUID | |
| `rule_id` | UUID | |
| `base_amount` | DECIMAL(15,2) | |
| `commission_amount` | DECIMAL(15,2) | |
| `status` | VARCHAR(20) | `pending`, `approved`, `paid`, `cancelled` |
| `period` | VARCHAR(7) | `2026-08` |
| `paid_at` | TIMESTAMPTZ | |

Index: `(employee_id, period, status)`.

### 14.5 `expense_categories` dan `expenses`

`expense_categories`: `id`, `tenant_id`, `business_id`, `name`, `code`, `parent_id`, `is_active`.

`expenses`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id`, `category_id` | UUID | |
| `document_number` | VARCHAR(40) | |
| `description` | VARCHAR(255) | |
| `amount` | DECIMAL(15,2) | |
| `payment_method_id` | UUID | |
| `shift_id` | UUID | Jika dibayar dari kas kasir |
| `expense_date` | DATE | |
| `attachment_path` | VARCHAR(255) | |
| `is_recurring` | BOOLEAN | |
| `recurring_config` | JSONB | Frekuensi dan tanggal berikutnya |
| `created_by`, `approved_by` | UUID | |
| `notes` | TEXT | |

Index: `(business_id, expense_date)`, `(outlet_id, category_id, expense_date)`.

---

## 15. Domain: Pelaporan (Read Model)

Tabel agregat yang diperbarui lewat event dan direkonsiliasi harian.

### 15.1 `daily_sales_summaries`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `outlet_id` | UUID | |
| `business_date` | DATE | |
| `transaction_count` | INT | |
| `item_count` | DECIMAL(15,4) | |
| `gross_sales` | DECIMAL(15,2) | Sebelum diskon |
| `discount_total` | DECIMAL(15,2) | |
| `net_sales` | DECIMAL(15,2) | |
| `tax_total`, `service_charge_total`, `rounding_total` | DECIMAL(15,2) | |
| `grand_total` | DECIMAL(15,2) | |
| `cost_total` | DECIMAL(15,2) | |
| `gross_profit` | DECIMAL(15,2) | |
| `void_count`, `void_total` | | |
| `return_count`, `return_total` | | |
| `average_transaction_value` | DECIMAL(15,2) | |
| `guest_count` | INT | |
| `hourly_breakdown` | JSONB | Omzet per jam untuk heatmap |
| `recalculated_at` | TIMESTAMPTZ | |

Index: `UNIQUE(outlet_id, business_date)`.

### 15.2 `product_daily_sales`

`id`, `tenant_id`, `business_id`, `outlet_id`, `variant_id`, `business_date`, `quantity_sold`, `gross_amount`, `discount_amount`, `net_amount`, `cost_amount`, `gross_profit`, `transaction_count`.

Index: `UNIQUE(outlet_id, variant_id, business_date)`, `(business_id, business_date, net_amount DESC)`.

### 15.3 `payment_daily_summaries`

`id`, `tenant_id`, `business_id`, `outlet_id`, `payment_method_id`, `business_date`, `transaction_count`, `total_amount`, `fee_amount`, `refund_amount`, `net_amount`.

### 15.4 `cashier_daily_summaries`

`id`, `tenant_id`, `business_id`, `outlet_id`, `user_id`, `business_date`, `transaction_count`, `total_amount`, `discount_given`, `void_count`, `void_amount`, `return_count`, `cash_difference_total`, `average_transaction_value`.

Tabel ini yang menjadi dasar laporan pengendalian kecurangan.

---

## 16. Domain: Sistem

### 16.1 `audit_logs`

Append only. Tidak dapat dihapus oleh pengguna tenant.

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id` | UUID | |
| `outlet_id`, `terminal_id` | UUID | Nullable |
| `user_id` | UUID | Pelaku |
| `impersonated_by` | UUID | Diisi jika Super Admin sedang impersonasi |
| `event` | VARCHAR(60) | `sale.voided`, `module.disabled`, `price.changed` |
| `auditable_type` | VARCHAR(80) | |
| `auditable_id` | UUID | |
| `old_values` | JSONB | |
| `new_values` | JSONB | |
| `reason` | TEXT | |
| `ip_address` | INET | |
| `user_agent` | VARCHAR(255) | |
| `authorized_by` | UUID | Supervisor jika ada |
| `occurred_at` | TIMESTAMPTZ | |

Index: `(business_id, occurred_at DESC)`, `(auditable_type, auditable_id)`, `(user_id, occurred_at DESC)`, `(event, occurred_at DESC)`.

Partisi per bulan disarankan sejak awal karena tabel ini tumbuh paling cepat.

### 16.2 `sync_logs`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID | PK |
| `tenant_id`, `business_id`, `terminal_id`, `device_id` | UUID | |
| `direction` | VARCHAR(10) | `push`, `pull` |
| `entity_type` | VARCHAR(40) | |
| `record_count` | INT | |
| `success_count`, `duplicate_count`, `failed_count` | INT | |
| `payload_size_bytes` | INT | |
| `duration_ms` | INT | |
| `errors` | JSONB | |
| `client_time`, `server_time` | TIMESTAMPTZ | Untuk deteksi penyimpangan jam |
| `time_drift_seconds` | INT | |

### 16.3 `idempotency_keys`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `key` | VARCHAR(80) | PK |
| `tenant_id` | UUID | |
| `endpoint` | VARCHAR(120) | |
| `request_hash` | VARCHAR(64) | Mendeteksi kunci sama dengan payload berbeda |
| `response_status` | SMALLINT | |
| `response_body` | JSONB | |
| `locked_at` | TIMESTAMPTZ | Mencegah pemrosesan paralel |
| `created_at` | TIMESTAMPTZ | |
| `expires_at` | TIMESTAMPTZ | 24 jam |

### 16.4 `notifications`

`id`, `tenant_id`, `business_id`, `user_id`, `type`, `title`, `body`, `data` (JSONB), `channel` (`in_app`, `email`, `whatsapp`, `push`), `status`, `sent_at`, `read_at`.

### 16.5 `api_keys`

`id`, `tenant_id`, `business_id`, `name`, `key_prefix`, `key_hash`, `scopes` (JSONB), `rate_limit_per_minute`, `last_used_at`, `expires_at`, `revoked_at`, `created_by`.

### 16.6 `webhooks` dan `webhook_deliveries`

`webhooks`: `id`, `tenant_id`, `business_id`, `url`, `events` (JSONB), `secret`, `is_active`, `failure_count`, `disabled_at`.

`webhook_deliveries`: `id`, `webhook_id`, `event`, `payload`, `response_status`, `response_body`, `attempt`, `delivered_at`, `next_retry_at`.

### 16.7 `announcements`

`id`, `title`, `body`, `type`, `target_plans` (JSONB), `target_tenants` (JSONB), `starts_at`, `ends_at`, `is_dismissible`, `created_by`.

---

## 17. Ringkasan Relasi Kunci

```
tenants 1─────n businesses 1─────n outlets 1─────n terminals
                     │                   │
                     │                   ├──n shifts 1──n cash_movements
                     │                   ├──n stocks
                     │                   ├──n dining_tables
                     │                   └──n printers
                     │
                     ├──n business_modules
                     ├──n products 1──n product_variants ──n product_barcodes
                     │        │                │
                     │        │                ├──n product_units
                     │        │                ├──n product_prices
                     │        │                └──n stock_movements
                     │        └──n product_modifier_group ──n modifier_groups ──n modifier_options
                     │
                     ├──n customers ──n receivables ──n receivable_payments
                     │        └──n point_transactions
                     │
                     ├──n suppliers ──n purchase_orders ──n goods_receipts ──n payables
                     │
                     ├──n promotions ──n coupons ──n promotion_usages
                     │
                     └──n sales 1──n sale_items 1──n sale_item_modifiers
                              ├──n sale_payments
                              ├──n sale_returns
                              ├──1 service_orders (opsional)
                              └──n kitchen_tickets (opsional)
```

---

## 18. Catatan Implementasi

### 18.1 Trigger Immutability

```sql
CREATE OR REPLACE FUNCTION prevent_sale_mutation() RETURNS trigger AS $$
BEGIN
  IF OLD.status = 'completed' AND (
       NEW.grand_total IS DISTINCT FROM OLD.grand_total OR
       NEW.subtotal   IS DISTINCT FROM OLD.subtotal OR
       NEW.tax_amount IS DISTINCT FROM OLD.tax_amount
     ) THEN
    RAISE EXCEPTION 'Transaksi yang sudah selesai tidak dapat diubah nilainya';
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

Kolom yang tetap boleh berubah setelah `completed`: `synced_at`, `has_negative_stock_flag`, `price_drift`, `void_reason` beserta kolom void terkait.

### 18.2 Rekonstruksi Stok

```sql
-- Verifikasi konsistensi cache stok terhadap ledger
SELECT s.outlet_id, s.variant_id, s.quantity AS cached,
       COALESCE(SUM(CASE WHEN m.direction = 'in' THEN m.quantity ELSE -m.quantity END), 0) AS computed
FROM stocks s
LEFT JOIN stock_movements m
       ON m.outlet_id = s.outlet_id AND m.variant_id = s.variant_id
GROUP BY s.outlet_id, s.variant_id, s.quantity
HAVING s.quantity <> COALESCE(SUM(CASE WHEN m.direction = 'in' THEN m.quantity ELSE -m.quantity END), 0);
```

Dijalankan sebagai job harian. Selisih dicatat dan dilaporkan, bukan diperbaiki diam diam.

### 18.3 Strategi Partisi

| Tabel | Ambang Partisi | Kunci |
|---|---|---|
| `sales` | 5 juta baris | RANGE pada `business_date`, per bulan |
| `sale_items` | 20 juta baris | RANGE pada `created_at`, per bulan |
| `stock_movements` | 10 juta baris | RANGE pada `occurred_at`, per bulan |
| `audit_logs` | Sejak awal | RANGE pada `occurred_at`, per bulan |

### 18.4 Retensi

| Data | Retensi | Perlakuan |
|---|---|---|
| Transaksi | Sesuai paket, minimal 5 tahun untuk berbayar | Arsip ke tabel dingin setelah 24 bulan |
| Audit log | 12 bulan aktif | Ekspor ke object storage lalu drop partisi |
| Sync log | 30 hari | Hapus |
| Idempotency key | 24 jam | Hapus |
| Held sales kedaluwarsa | Sampai tutup shift | Hapus |
| Notifikasi terbaca | 90 hari | Hapus |

### 18.5 Urutan Migrasi

```
01_extensions           (uuid-ossp, pg_trgm, btree_gin)
02_platform             (tenants, plans, modules, plan_modules, subscriptions, invoices)
03_identity             (users, roles, permissions, role_permissions, user_business_roles, invitations, devices)
04_tenancy              (businesses, business_modules, outlets, terminals, printers, tax_rates, document_sequences)
05_catalog              (units, categories, products, variants, barcodes, product_units, price_lists, product_prices)
06_catalog_modifier     (modifier_groups, modifier_options, product_modifier_group, bundle_items, recipes, recipe_items)
07_inventory            (stocks, stock_movements, adjustments, opnames, transfers, batches, serials)
08_customer             (customer_groups, customers, point_transactions, receivables, receivable_payments)
09_promotion            (promotions, promotion_targets, coupons, promotion_usages, vouchers)
10_sales                (payment_methods, sales, sale_items, sale_item_modifiers, sale_payments, sale_returns, held_sales)
11_shift                (shifts, cash_movements)
12_fnb                  (dining_areas, dining_tables, table_sessions, kitchen_tickets, reservations)
13_service              (service_status_flows, service_orders, items, histories, attachments)
14_purchasing           (suppliers, purchase_orders, goods_receipts, purchase_returns, payables)
15_employee_expense     (employees, attendances, commission_rules, commissions, expense_categories, expenses)
16_reporting            (daily summaries)
17_system               (audit_logs, sync_logs, idempotency_keys, notifications, api_keys, webhooks, announcements)
18_rls_policies         (aktivasi RLS dan policy per tabel)
19_triggers             (immutability, updated_at, balance_after)
```

Urutan ini menghormati ketergantungan foreign key dan dapat dijalankan bertahap sesuai fase rilis.
