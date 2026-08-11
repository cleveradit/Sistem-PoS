# Product Requirements Document (PRD)
## Aplikasi Point of Sale (POS) Multi-Tenant

| Item | Keterangan |
|---|---|
| Nama Produk | (TBD, working title: **PosKu**) |
| Versi Dokumen | 1.0 |
| Tanggal | 10 Agustus 2026 |
| Status | Draft untuk review |
| Penyusun | Radityo |
| Tipe Produk | SaaS multi-tenant, web based (responsive) + PWA kasir |

---

## 1. Ringkasan Eksekutif

Aplikasi POS multi-tenant yang dapat digunakan oleh banyak pemilik bisnis secara independen dalam satu instalasi platform. Setiap pemilik bisnis dapat memiliki lebih dari satu bisnis, setiap bisnis dapat memiliki banyak outlet, dan setiap outlet dapat memiliki banyak terminal kasir.

Prinsip utama produk ini adalah **superset fitur dengan kontrol aktivasi**. Platform menyediakan katalog fitur selengkap mungkin yang mencakup kebutuhan retail, F&B, jasa, dan grosir. Pemilik bisnis mengaktifkan hanya modul yang relevan lewat *feature flag*, sehingga antarmuka kasir tetap ringkas dan tidak membingungkan karyawan.

Tanpa mekanisme ini, produk universal akan gagal di dua arah: terlalu ramai untuk warung kecil, atau terlalu dangkal untuk restoran dan grosir.

---

## 2. Latar Belakang dan Pernyataan Masalah

### 2.1 Masalah

1. Aplikasi POS di pasar umumnya terspesialisasi pada satu vertical. Pemilik bisnis dengan lebih dari satu jenis usaha (misal punya toko kelontong sekaligus kedai kopi) harus berlangganan dua aplikasi berbeda dengan dua laporan terpisah.
2. Aplikasi POS "all in one" yang ada cenderung membebani pengguna kecil dengan menu yang tidak pernah dipakai, sehingga onboarding lambat dan tingkat churn tinggi.
3. Banyak POS lokal tidak memiliki jejak audit yang layak, sehingga pemilik tidak bisa membuktikan selisih kas atau kecurangan kasir.
4. Ketergantungan penuh pada koneksi internet membuat penjualan berhenti saat jaringan mati.

### 2.2 Peluang

Satu platform, banyak profil bisnis, satu sumber kebenaran laporan lintas bisnis untuk pemilik. Diferensiasi ada pada tiga hal: konfigurabilitas modul, ketahanan offline, dan kualitas audit trail.

---

## 3. Tujuan Produk dan Metrik Keberhasilan

### 3.1 Tujuan

| Kode | Tujuan |
|---|---|
| G-01 | Satu platform melayani minimal empat vertical (retail, F&B, jasa, grosir) tanpa fork kode |
| G-02 | Pemilik bisnis dapat menyelesaikan setup awal dan transaksi pertama dalam waktu di bawah 15 menit |
| G-03 | Kasir dapat menyelesaikan transaksi retail sederhana dalam maksimal 5 interaksi layar |
| G-04 | Penjualan tetap berjalan saat koneksi internet putus |
| G-05 | Pemilik memperoleh visibilitas penuh atas kas, stok, dan aktivitas karyawan lintas outlet |

### 3.2 Metrik Keberhasilan

| Metrik | Target |
|---|---|
| Time to first transaction (dari registrasi) | < 15 menit |
| Rata rata durasi transaksi retail (scan sampai struk) | < 30 detik |
| Uptime layanan inti | 99,5 persen per bulan |
| Transaksi offline tersinkron sempurna tanpa duplikasi | 100 persen |
| Tenant aktif yang mematikan minimal 1 modul | > 70 persen (indikator feature flag terpakai) |
| Churn bulanan tenant berbayar | < 5 persen |

---

## 4. Ruang Lingkup

### 4.1 Termasuk dalam Lingkup (v1 sampai v3)

- Manajemen tenant, bisnis, outlet, terminal
- Master data produk, layanan, varian, modifier, satuan, kategori
- Transaksi penjualan, retur, penukaran, void
- Manajemen persediaan dan pembelian
- Modul F&B (meja, open bill, KDS)
- Modul jasa (order berdurasi, DP, status pengerjaan)
- Pembayaran multi metode, termasuk QRIS dan EDC manual
- Shift kas dan rekonsiliasi
- Promo, diskon, voucher, loyalitas
- Pelanggan dan piutang
- Karyawan, hak akses, komisi
- Pengeluaran operasional dasar
- Laporan dan analitik
- Mode offline dan sinkronisasi
- Audit log
- Langganan dan penagihan platform

### 4.2 Di Luar Lingkup

- Akuntansi berpasangan penuh (general ledger, neraca, jurnal umum). Platform hanya menyediakan ekspor ke sistem akuntansi
- Payroll penuh (perhitungan PPh 21, BPJS)
- Manajemen produksi atau manufaktur (Bill of Material bertingkat). Hanya resep sederhana satu tingkat
- Aplikasi native iOS/Android pada v1. Menggunakan PWA
- E-Faktur dan pelaporan pajak otomatis ke DJP pada v1

---

## 5. Aktor dan Persona

### 5.1 Daftar Aktor

| Aktor | Cakupan | Deskripsi |
|---|---|---|
| **Super Admin (Platform)** | Global | Saya sebagai pengelola platform. Mengelola tenant, paket langganan, katalog fitur global, monitoring sistem, dukungan teknis |
| **Support Agent (Platform)** | Global, terbatas | Staf dukungan. Dapat melihat data tenant untuk troubleshooting dengan izin dan tanpa akses tulis ke transaksi |
| **Pemilik Bisnis (Tenant Owner)** | Tenant | Pemilik akun. Membuat bisnis, mengatur langganan, mengaktifkan modul, mengundang pengguna, akses penuh semua data di tenantnya |
| **Manajer Bisnis** | Bisnis | Mengelola satu atau lebih outlet di bawah satu bisnis. Tidak dapat mengubah langganan atau menghapus bisnis |
| **Supervisor Outlet** | Outlet | Otorisasi void, diskon manual, retur. Menutup shift dan menyetujui selisih kas |
| **Kasir** | Terminal | Melakukan transaksi, membuka dan menutup shift sendiri |
| **Staf Gudang** | Outlet | Penerimaan barang, stok opname, transfer antar outlet. Tidak melihat data keuangan |
| **Staf Produksi / Dapur** | Outlet | Melihat dan memperbarui status pesanan pada layar dapur (KDS) |
| **Pelanggan (opsional)** | Publik | Melihat status order, riwayat poin, struk digital lewat tautan publik |

### 5.2 Persona Ringkas

**Bu Sari, 42, pemilik toko kelontong.** Punya 1 outlet, 2 kasir. Butuh scan barcode, stok cepat, laporan harian di HP. Tidak butuh meja, modifier, atau KDS. Sensitif terhadap kompleksitas.

**Mas Dimas, 31, pemilik 3 kedai kopi.** Butuh meja, open bill, modifier topping, KDS, laporan per outlet, kontrol bahan baku lewat resep. Mengecek performa outlet dari HP setiap malam.

**Pak Anton, 50, pemilik bengkel dan laundry.** Dua bisnis berbeda dalam satu akun. Butuh order berdurasi, DP, notifikasi WA ke pelanggan, dan laporan gabungan.

**Rina, 22, kasir.** Bekerja shift. Butuh alur cepat, minim salah tekan, dan kejelasan saat tutup kas agar tidak disalahkan atas selisih.

---

## 6. Model Hirarki dan Multi-Tenancy

### 6.1 Hirarki Entitas

```
Platform
└── Tenant (akun pemilik bisnis, unit langganan dan penagihan)
    └── Business (unit bisnis, punya tipe: retail / fnb / jasa / grosir / campuran)
        └── Outlet (cabang fisik, punya alamat, jam operasional, zona waktu)
            ├── Terminal (device kasir, punya nomor urut struk sendiri)
            └── Storage Location (gudang atau rak, opsional)
```

### 6.2 Aturan Multi-Tenancy

| Kode | Kebutuhan |
|---|---|
| FR-MT-01 | Satu Tenant dapat memiliki banyak Business dengan tipe berbeda |
| FR-MT-02 | Setiap Business memiliki konfigurasi feature flag, mata uang, pajak, dan format struk sendiri |
| FR-MT-03 | Data antar Tenant terisolasi total. Kueri wajib difilter oleh tenant scope pada lapisan aplikasi |
| FR-MT-04 | Pengguna dapat diundang ke lebih dari satu Business dengan peran berbeda per Business |
| FR-MT-05 | Master data (produk, pelanggan, supplier) dimiliki pada level Business, bukan Outlet. Stok dan harga dapat dioverride per Outlet |
| FR-MT-06 | Pemilik dapat melihat laporan konsolidasi lintas Business dalam satu Tenant |
| FR-MT-07 | Penghapusan Tenant bersifat soft delete dengan masa tenggang 30 hari sebelum purge permanen |
| FR-MT-08 | Setiap Tenant dapat memiliki subdomain atau slug unik untuk tautan publik (struk digital, status order) |

### 6.3 Strategi Isolasi Data

Pendekatan: **single database, shared schema, dengan kolom `tenant_id` dan `business_id` pada seluruh tabel transaksional**, ditegakkan lewat global scope di lapisan ORM dan row level policy.

Alasan: biaya operasional rendah, migrasi tunggal, dan cukup untuk skala target. Tenant enterprise di masa depan dapat dipindahkan ke database terpisah tanpa mengubah kode aplikasi jika koneksi database ditentukan secara dinamis per tenant sejak awal.

Kebutuhan wajib:
- Semua query builder default menerapkan scope tenant otomatis
- Setiap operasi lintas tenant hanya dapat dilakukan oleh Super Admin dan wajib tercatat di audit log
- Uji otomatis wajib mencakup skenario kebocoran data antar tenant

---

## 7. Sistem Feature Flag (Inti Diferensiasi Produk)

### 7.1 Konsep

Setiap kemampuan produk dibungkus dalam **Module** yang dapat diaktifkan atau dimatikan per Business. Modul yang mati akan:

1. Menyembunyikan menu navigasi terkait
2. Menyembunyikan field terkait pada form (misal field "Modifier" hilang dari form produk)
3. Menonaktifkan endpoint API terkait (mengembalikan 403, bukan sekadar sembunyi di UI)
4. Menyembunyikan laporan terkait

### 7.2 Tiga Lapis Kontrol

| Lapis | Pengendali | Fungsi |
|---|---|---|
| **Plan Entitlement** | Super Admin | Menentukan modul mana yang tersedia untuk paket langganan tertentu. Modul di luar paket ditampilkan terkunci dengan ajakan upgrade |
| **Business Toggle** | Pemilik Bisnis | Mengaktifkan atau mematikan modul yang tersedia di paketnya, per Business |
| **Role Permission** | Pemilik / Manajer | Menentukan siapa yang boleh mengakses fitur yang sudah aktif |

Urutan evaluasi: `Plan Entitlement AND Business Toggle AND Role Permission`.

### 7.3 Preset Tipe Bisnis

Saat membuat Business, pengguna memilih tipe. Sistem menerapkan preset modul, yang tetap dapat diubah manual sesudahnya.

| Modul | Retail | F&B | Jasa | Grosir | Minimal |
|---|:--:|:--:|:--:|:--:|:--:|
| Penjualan dasar | ON | ON | ON | ON | ON |
| Barcode & SKU | ON | OFF | OFF | ON | OFF |
| Varian produk | ON | OFF | OFF | ON | OFF |
| Modifier & add-on | OFF | ON | OFF | OFF | OFF |
| Manajemen meja | OFF | ON | OFF | OFF | OFF |
| Open bill | OFF | ON | OFF | OFF | OFF |
| Kitchen Display (KDS) | OFF | ON | OFF | OFF | OFF |
| Manajemen stok | ON | ON | OFF | ON | OFF |
| Resep / bahan baku | OFF | ON | OFF | OFF | OFF |
| Multi satuan & konversi | OFF | OFF | OFF | ON | OFF |
| Harga bertingkat | OFF | OFF | OFF | ON | OFF |
| Order berdurasi & status | OFF | OFF | ON | OFF | OFF |
| DP / pelunasan | OFF | OFF | ON | ON | OFF |
| Pickup & delivery | OFF | ON | ON | ON | OFF |
| Pelanggan & loyalitas | ON | ON | ON | ON | OFF |
| Piutang | OFF | OFF | ON | ON | OFF |
| Pembelian & supplier | ON | ON | OFF | ON | OFF |
| Shift kas | ON | ON | ON | ON | ON |
| Komisi karyawan | OFF | OFF | ON | OFF | OFF |
| Batch & kedaluwarsa | OFF | ON | OFF | ON | OFF |
| Serial number | OFF | OFF | OFF | OFF | OFF |
| Pengeluaran operasional | ON | ON | ON | ON | OFF |

Preset "Minimal" ditujukan untuk warung atau pedagang tunggal yang hanya butuh mencatat penjualan.

### 7.4 Kebutuhan Fungsional Feature Flag

| Kode | Kebutuhan | Prioritas |
|---|---|---|
| FR-FF-01 | Katalog modul dikelola Super Admin, berisi kode, nama, deskripsi, dependensi, dan modul konflik | Must |
| FR-FF-02 | Modul dapat memiliki dependensi. Mengaktifkan "Resep" otomatis mengaktifkan "Manajemen stok" dengan konfirmasi ke pengguna | Must |
| FR-FF-03 | Mematikan modul yang menjadi dependensi modul aktif lain harus ditolak dengan pesan penjelas | Must |
| FR-FF-04 | Mematikan modul tidak menghapus data. Data disembunyikan dan kembali utuh saat modul diaktifkan lagi | Must |
| FR-FF-05 | Sistem memberi peringatan sebelum mematikan modul yang masih punya data aktif (misal ada 12 open bill berjalan) | Must |
| FR-FF-06 | Perubahan flag tercatat di audit log dengan pelaku dan waktu | Must |
| FR-FF-07 | Konfigurasi modul dapat diubah kapan saja tanpa perlu restart atau deploy | Must |
| FR-FF-08 | Tersedia halaman "Jelajahi Fitur" yang memperlihatkan modul terkunci beserta manfaat dan paket yang dibutuhkan | Should |
| FR-FF-09 | Pemilik dapat menyalin konfigurasi modul dari satu Business ke Business lain | Could |
| FR-FF-10 | Super Admin dapat mengaktifkan modul untuk tenant tertentu di luar paket (override manual, misal untuk uji coba) | Should |

---

## 8. Kebutuhan Fungsional per Modul

Penomoran: `FR-[KODE MODUL]-[NOMOR]`. Prioritas menggunakan MoSCoW.

### 8.1 Autentikasi dan Manajemen Pengguna (`AUTH`)

| Kode | Kebutuhan | Prioritas |
|---|---|---|
| FR-AUTH-01 | Registrasi mandiri pemilik bisnis dengan email dan verifikasi | Must |
| FR-AUTH-02 | Login dengan email dan kata sandi | Must |
| FR-AUTH-03 | Login kasir cepat dengan PIN 4 sampai 6 digit pada terminal yang sudah terotorisasi | Must |
| FR-AUTH-04 | Reset kata sandi lewat email dengan token kedaluwarsa | Must |
| FR-AUTH-05 | Two Factor Authentication (TOTP) opsional untuk peran Pemilik dan Manajer | Should |
| FR-AUTH-06 | Undang pengguna lewat email dengan penetapan peran dan cakupan outlet | Must |
| FR-AUTH-07 | Nonaktifkan pengguna tanpa menghapus riwayat transaksinya | Must |
| FR-AUTH-08 | Satu akun pengguna dapat tergabung di banyak Business dengan peran berbeda, dan dapat berpindah konteks | Must |
| FR-AUTH-09 | Sesi kasir otomatis terkunci setelah periode idle yang dapat dikonfigurasi | Should |
| FR-AUTH-10 | Riwayat perangkat dan sesi aktif, dengan kemampuan mencabut sesi jarak jauh | Should |
| FR-AUTH-11 | Login sosial (Google) | Could |

### 8.2 Manajemen Peran dan Hak Akses (`RBAC`)

| Kode | Kebutuhan | Prioritas |
|---|---|---|
| FR-RBAC-01 | Peran bawaan: Pemilik, Manajer, Supervisor, Kasir, Staf Gudang, Staf Dapur | Must |
| FR-RBAC-02 | Peran kustom yang dapat dibuat pemilik dengan pemilihan izin granular | Should |
| FR-RBAC-03 | Izin bersifat granular per aksi, bukan per menu. Contoh: `sale.void`, `sale.discount.manual`, `report.profit.view` | Must |
| FR-RBAC-04 | Cakupan akses dapat dibatasi hingga level Outlet tertentu | Must |
| FR-RBAC-05 | Aksi sensitif yang tidak diizinkan bagi kasir dapat dilewati lewat **otorisasi supervisor** (supervisor memasukkan PIN di terminal kasir tanpa perlu logout) | Must |
| FR-RBAC-06 | Setiap otorisasi supervisor tercatat: siapa, aksi apa, transaksi mana, alasan | Must |
| FR-RBAC-07 | Batas nominal diskon manual per peran dapat dikonfigurasi (misal kasir maksimal 5 persen) | Should |

### 8.3 Manajemen Bisnis dan Outlet (`ORG`)

| Kode | Kebutuhan | Prioritas |
|---|---|---|
| FR-ORG-01 | CRUD Business dengan tipe, logo, mata uang, zona waktu, dan format nomor | Must |
| FR-ORG-02 | CRUD Outlet dengan alamat, kontak, jam operasional, dan kontak penanggung jawab | Must |
| FR-ORG-03 | Registrasi terminal kasir dengan nama, kode, dan prefix nomor struk unik | Must |
| FR-ORG-04 | Terminal harus diaktifkan oleh Manajer sebelum dapat dipakai bertransaksi | Must |
| FR-ORG-05 | Konfigurasi pajak per Business dan override per Outlet (PPN 11 persen, PB1 10 persen, atau nonaktif) | Must |
| FR-ORG-06 | Pilihan pajak inklusif (harga sudah termasuk pajak) atau eksklusif | Must |
| FR-ORG-07 | Service charge dengan persentase yang dapat dikonfigurasi dan opsi dikenai pajak atau tidak | Should |
| FR-ORG-08 | Pembulatan nominal akhir (ke 100, 500, 1000) dengan opsi pembulatan ke atas, bawah, atau terdekat | Should |
| FR-ORG-09 | Kustomisasi struk: logo, header, footer, tampilkan atau sembunyikan NPWP, pesan promosi | Must |
| FR-ORG-10 | Penonaktifan sementara Outlet (misal renovasi) tanpa kehilangan data | Could |

### 8.4 Master Data Produk dan Layanan (`PRD`)

| Kode | Kebutuhan | Prioritas |
|---|---|---|
| FR-PRD-01 | CRUD produk dengan nama, kode/SKU, kategori, harga jual, harga beli, gambar, deskripsi | Must |
| FR-PRD-02 | Tipe item: **Barang** (dilacak stok), **Layanan** (tanpa stok), **Paket/Bundle** (kumpulan item), **Item Terbuka** (harga diisi saat transaksi) | Must |
| FR-PRD-03 | Kategori bertingkat maksimal 3 level | Should |
| FR-PRD-04 | Varian produk berbasis atribut (ukuran, warna, rasa) yang menghasilkan SKU turunan dengan harga dan stok masing masing | Must |
| FR-PRD-05 | Modifier group (contoh: level gula, topping) dengan aturan minimal dan maksimal pilihan, serta harga tambahan per opsi | Must |
| FR-PRD-06 | Modifier dapat berupa pilihan tunggal (radio) atau ganda (checkbox) | Must |
| FR-PRD-07 | Barcode ganda per produk (satu produk bisa punya beberapa barcode dari supplier berbeda) | Should |
| FR-PRD-08 | Generate dan cetak label barcode dengan template ukuran yang dapat dipilih | Should |
| FR-PRD-09 | Multi satuan dengan faktor konversi (contoh: 1 dus = 24 pcs) dan harga per satuan | Should |
| FR-PRD-10 | Harga bertingkat berdasarkan kuantitas (grosir) atau tipe pelanggan (member, reseller) | Should |
| FR-PRD-11 | Harga berbeda per Outlet dengan fallback ke harga Business | Should |
| FR-PRD-12 | Harga berbeda per kanal penjualan (dine in, take away, ojek online) | Should |
| FR-PRD-13 | Resep atau komposisi satu tingkat: satu item jual mengurangi beberapa item bahan baku | Should |
| FR-PRD-14 | Pelacakan batch dan tanggal kedaluwarsa untuk item tertentu | Should |
| FR-PRD-15 | Pelacakan serial number untuk item bernilai tinggi (elektronik) | Could |
| FR-PRD-16 | Item dengan penjualan berbasis berat (integrasi timbangan atau input manual desimal) | Could |
| FR-PRD-17 | Import dan export produk lewat CSV atau XLSX dengan validasi baris dan laporan kesalahan | Must |
| FR-PRD-18 | Penandaan produk favorit dan pengurutan manual pada grid kasir | Should |
| FR-PRD-19 | Arsip produk (tidak dijual lagi) tanpa menghapus riwayat transaksi | Must |
| FR-PRD-20 | Duplikasi produk sebagai template untuk mempercepat input | Could |

### 8.5 Persediaan / Inventory (`INV`)

| Kode | Kebutuhan | Prioritas |
|---|---|---|
| FR-INV-01 | Stok dicatat per kombinasi Outlet dan Produk/Varian | Must |
| FR-INV-02 | Kartu stok (stock ledger) yang mencatat setiap mutasi: penjualan, retur, pembelian, penyesuaian, transfer, pemakaian resep | Must |
| FR-INV-03 | Penyesuaian stok manual dengan alasan wajib (rusak, hilang, kedaluwarsa, koreksi) | Must |
| FR-INV-04 | Stok opname dengan mode terjadwal dan mode parsial per kategori atau lokasi | Must |
| FR-INV-05 | Opname mendukung penghitungan buta (blind count) di mana penghitung tidak melihat stok sistem | Should |
| FR-INV-06 | Transfer stok antar Outlet dengan status: dikirim, diterima, sebagian, ditolak | Should |
| FR-INV-07 | Peringatan stok minimum per produk per outlet, ditampilkan di dasbor dan notifikasi | Must |
| FR-INV-08 | Metode penilaian persediaan: Average Cost (default) atau FIFO, dipilih per Business dan tidak dapat diubah setelah ada transaksi | Should |
| FR-INV-09 | Pencatatan HPP pada saat penjualan untuk perhitungan laba kotor | Must |
| FR-INV-10 | Opsi izinkan atau larang penjualan saat stok nol (stok minus) per Business | Must |
| FR-INV-11 | Laporan stok bergerak lambat dan stok mati | Should |
| FR-INV-12 | Laporan barang mendekati kedaluwarsa (jika modul batch aktif) | Should |
| FR-INV-13 | Rekonsiliasi otomatis stok setelah sinkronisasi transaksi offline | Must |

### 8.6 Transaksi Penjualan (`SALE`)

Ini adalah alur paling kritis. Kecepatan dan kejelasan lebih penting daripada kelengkapan.

| Kode | Kebutuhan | Prioritas |
|---|---|---|
| FR-SALE-01 | Tambah item ke keranjang lewat scan barcode, pencarian teks, atau tap pada grid produk | Must |
| FR-SALE-02 | Ubah kuantitas, hapus item, kosongkan keranjang | Must |
| FR-SALE-03 | Catatan per item (contoh: "tanpa es") dan catatan per transaksi | Must |
| FR-SALE-04 | Diskon per item dan per transaksi, dalam nominal atau persentase | Must |
| FR-SALE-05 | Diskon manual di atas ambang batas peran memerlukan otorisasi supervisor | Must |
| FR-SALE-06 | Perhitungan pajak dan service charge otomatis sesuai konfigurasi Outlet | Must |
| FR-SALE-07 | Simpan transaksi sementara (hold / parked bill) dengan label, lalu panggil kembali | Must |
| FR-SALE-08 | Void item sebelum pembayaran, dan void transaksi setelah pembayaran dengan otorisasi dan alasan wajib | Must |
| FR-SALE-09 | Retur penjualan sebagian atau penuh, dengan pengembalian dana tunai, kredit toko, atau tukar barang | Must |
| FR-SALE-10 | Nomor transaksi unik berurutan per terminal, tahan terhadap kondisi offline | Must |
| FR-SALE-11 | Pemilihan pelanggan pada transaksi (opsional atau wajib sesuai konfigurasi) | Must |
| FR-SALE-12 | Penetapan kasir dan pramuniaga (sales person) berbeda pada satu transaksi untuk keperluan komisi | Should |
| FR-SALE-13 | Pemilihan kanal penjualan (dine in, take away, delivery, ojek online) yang memengaruhi harga dan pajak | Should |
| FR-SALE-14 | Item dengan harga terbuka (open price) dan item cepat (quick item) tanpa master produk | Should |
| FR-SALE-15 | Mode antrean cepat: satu tap langsung bayar tunai pas | Should |
| FR-SALE-16 | Pratinjau struk sebelum cetak dan opsi cetak ulang dengan penanda "COPY" | Must |
| FR-SALE-17 | Struk digital lewat tautan, dikirim via WhatsApp atau email | Should |
| FR-SALE-18 | Pencarian riwayat transaksi berdasarkan nomor, tanggal, pelanggan, atau kasir | Must |
| FR-SALE-19 | Keyboard shortcut untuk operasi utama pada terminal desktop | Should |
| FR-SALE-20 | Tampilan layar pelanggan (customer display) pada perangkat kedua | Could |

### 8.7 Pembayaran (`PAY`)

| Kode | Kebutuhan | Prioritas |
|---|---|---|
| FR-PAY-01 | Metode pembayaran yang dapat dikonfigurasi: tunai, QRIS, kartu debit/kredit (EDC), transfer bank, e-wallet, voucher, poin, piutang | Must |
| FR-PAY-02 | Pembayaran terpisah (split payment) dengan lebih dari satu metode dalam satu transaksi | Must |
| FR-PAY-03 | Perhitungan kembalian otomatis dan saran nominal uang tunai (pecahan umum) | Must |
| FR-PAY-04 | Pencatatan nomor referensi atau 4 digit terakhir kartu untuk pembayaran non tunai | Should |
| FR-PAY-05 | Biaya admin per metode pembayaran (misal MDR kartu 2 persen) yang tercatat untuk laporan | Should |
| FR-PAY-06 | Integrasi QRIS dinamis lewat payment gateway dengan konfirmasi otomatis | Should |
| FR-PAY-07 | Pembayaran uang muka (DP) dan pelunasan bertahap dengan riwayat cicilan | Should |
| FR-PAY-08 | Pembayaran dengan piutang pelanggan, tunduk pada batas kredit | Should |
| FR-PAY-09 | Tip atau uang jasa yang dapat dialokasikan ke karyawan tertentu | Could |
| FR-PAY-10 | Pembatalan pembayaran (refund) dengan jejak audit dan otorisasi | Must |

### 8.8 Shift Kas dan Rekonsiliasi (`SHIFT`)

| Kode | Kebutuhan | Prioritas |
|---|---|---|
| FR-SHIFT-01 | Buka shift dengan pencatatan modal awal laci kas | Must |
| FR-SHIFT-02 | Kasir tidak dapat bertransaksi sebelum shift dibuka | Must |
| FR-SHIFT-03 | Pencatatan kas masuk dan kas keluar di tengah shift dengan keterangan (setoran, ambil kembalian, bayar parkir) | Must |
| FR-SHIFT-04 | Tutup shift dengan input hitungan fisik uang per pecahan, lalu sistem menampilkan selisih | Must |
| FR-SHIFT-05 | Selisih di atas ambang tertentu memerlukan persetujuan supervisor dan alasan | Must |
| FR-SHIFT-06 | Laporan tutup shift (X report dan Z report) yang dapat dicetak | Must |
| FR-SHIFT-07 | Satu terminal hanya boleh punya satu shift terbuka pada satu waktu | Must |
| FR-SHIFT-08 | Shift yang terlupa ditutup akan diberi tanda dan dapat ditutup paksa oleh Manajer dengan catatan | Should |
| FR-SHIFT-09 | Riwayat shift dengan detail transaksi, metode pembayaran, dan selisih per kasir | Must |

### 8.9 Modul F&B (`FNB`)

| Kode | Kebutuhan | Prioritas |
|---|---|---|
| FR-FNB-01 | Denah meja dengan area (indoor, outdoor, lantai 2) dan status: kosong, terisi, dipesan, perlu dibersihkan | Should |
| FR-FNB-02 | Open bill per meja: pesanan bertambah seiring waktu, dibayar di akhir | Must |
| FR-FNB-03 | Pindah meja dan gabung meja dengan jejak audit | Should |
| FR-FNB-04 | Split bill berdasarkan item atau berdasarkan jumlah orang | Should |
| FR-FNB-05 | Kirim pesanan ke dapur (fire order) tanpa menutup bill, dengan pencetakan tiket dapur | Must |
| FR-FNB-06 | Kitchen Display System: daftar pesanan realtime, urut waktu, dengan status baru, diproses, siap, diantar | Should |
| FR-FNB-07 | Routing tiket ke stasiun berbeda berdasarkan kategori (dapur panas, bar, dessert) | Should |
| FR-FNB-08 | Penanda waktu tunggu dan peringatan pesanan yang terlalu lama di KDS | Should |
| FR-FNB-09 | Pencatatan jumlah tamu (pax) per meja untuk laporan average check per pax | Should |
| FR-FNB-10 | Menu berdasarkan waktu (menu sarapan aktif 06.00 sampai 10.00) | Could |
| FR-FNB-11 | Penandaan item habis (86 list) yang langsung menonaktifkan item di semua terminal | Should |
| FR-FNB-12 | Reservasi meja sederhana dengan nama, waktu, dan jumlah tamu | Could |

### 8.10 Modul Jasa dan Order Berdurasi (`SRV`)

Ditujukan untuk laundry, bengkel, salon, servis elektronik, percetakan.

| Kode | Kebutuhan | Prioritas |
|---|---|---|
| FR-SRV-01 | Order memiliki alur status yang dapat dikonfigurasi per Business (contoh: diterima, dikerjakan, selesai, diambil) | Must |
| FR-SRV-02 | Estimasi waktu selesai otomatis berdasarkan durasi layanan | Should |
| FR-SRV-03 | Uang muka saat order dibuat dan pelunasan saat pengambilan | Must |
| FR-SRV-04 | Cetak nota order dan label penanda barang pelanggan | Must |
| FR-SRV-05 | Halaman lacak status publik untuk pelanggan lewat tautan atau QR pada nota | Should |
| FR-SRV-06 | Notifikasi otomatis ke pelanggan saat status berubah menjadi selesai | Should |
| FR-SRV-07 | Penugasan teknisi atau pekerja per order, sebagai dasar komisi | Should |
| FR-SRV-08 | Pencatatan detail objek servis (merek, tipe, nomor polisi, kondisi awal, foto) | Should |
| FR-SRV-09 | Layanan berbasis kuantitas terukur (kilogram, meter, jam) | Must |
| FR-SRV-10 | Penjadwalan janji temu dengan kalender dan slot waktu | Could |
| FR-SRV-11 | Penjemputan dan pengantaran dengan alamat, biaya, dan status pengantaran | Could |

### 8.11 Promo, Diskon, dan Loyalitas (`PROMO`)

| Kode | Kebutuhan | Prioritas |
|---|---|---|
| FR-PROMO-01 | Promo terjadwal berdasarkan tanggal, hari, dan jam (happy hour) | Should |
| FR-PROMO-02 | Tipe promo: potongan nominal, potongan persentase, harga khusus, beli X gratis Y, bundling | Should |
| FR-PROMO-03 | Syarat promo: minimum belanja, produk atau kategori tertentu, tipe pelanggan, outlet tertentu | Should |
| FR-PROMO-04 | Aturan penumpukan promo (stackable atau eksklusif) dan prioritas penerapan | Should |
| FR-PROMO-05 | Kupon dengan kode unik, batas pemakaian total dan per pelanggan | Should |
| FR-PROMO-06 | Voucher bernilai nominal yang dapat dijual dan ditebus | Could |
| FR-PROMO-07 | Program poin: rasio perolehan, rasio penukaran, masa berlaku poin | Should |
| FR-PROMO-08 | Tingkatan member (Bronze, Silver, Gold) dengan manfaat berbeda | Could |
| FR-PROMO-09 | Laporan efektivitas promo: jumlah pemakaian, nilai diskon, dampak pada omzet | Should |

### 8.12 Pelanggan dan Piutang (`CUST`)

| Kode | Kebutuhan | Prioritas |
|---|---|---|
| FR-CUST-01 | CRUD pelanggan: nama, telepon, email, alamat, tanggal lahir, catatan | Must |
| FR-CUST-02 | Pencarian pelanggan cepat lewat nomor telepon atau nama di layar kasir | Must |
| FR-CUST-03 | Pendaftaran pelanggan baru langsung dari layar kasir tanpa keluar dari transaksi | Must |
| FR-CUST-04 | Riwayat transaksi, total belanja, frekuensi, dan produk favorit per pelanggan | Should |
| FR-CUST-05 | Grup atau tipe pelanggan (umum, member, reseller) yang memengaruhi harga | Should |
| FR-CUST-06 | Batas kredit dan tempo pembayaran per pelanggan | Should |
| FR-CUST-07 | Kartu piutang: daftar tagihan, pembayaran, dan saldo per pelanggan | Should |
| FR-CUST-08 | Laporan umur piutang (aging: 0-30, 31-60, 61-90, di atas 90 hari) | Should |
| FR-CUST-09 | Pengingat jatuh tempo otomatis lewat WhatsApp atau email | Could |
| FR-CUST-10 | Import pelanggan lewat CSV | Should |
| FR-CUST-11 | Penggabungan data pelanggan duplikat | Could |

### 8.13 Pembelian dan Supplier (`PUR`)

| Kode | Kebutuhan | Prioritas |
|---|---|---|
| FR-PUR-01 | CRUD supplier dengan kontak, tempo pembayaran, dan catatan | Should |
| FR-PUR-02 | Purchase Order dengan status: draft, dikirim, sebagian diterima, selesai, dibatalkan | Should |
| FR-PUR-03 | Penerimaan barang yang menambah stok dan memperbarui harga beli | Must |
| FR-PUR-04 | Penerimaan sebagian dengan sisa yang tetap tercatat | Should |
| FR-PUR-05 | Biaya tambahan pembelian (ongkos kirim, bea) yang dialokasikan ke HPP | Could |
| FR-PUR-06 | Retur pembelian ke supplier | Should |
| FR-PUR-07 | Hutang usaha: daftar tagihan supplier, pembayaran, dan saldo | Should |
| FR-PUR-08 | Saran pembelian otomatis berdasarkan stok minimum dan rata rata penjualan | Could |
| FR-PUR-09 | Riwayat harga beli per supplier untuk perbandingan | Could |

### 8.14 Karyawan dan Operasional (`EMP`)

| Kode | Kebutuhan | Prioritas |
|---|---|---|
| FR-EMP-01 | Data karyawan: nama, kontak, jabatan, outlet penempatan, tanggal masuk | Should |
| FR-EMP-02 | Absensi masuk dan pulang lewat PIN pada terminal | Should |
| FR-EMP-03 | Jadwal shift kerja mingguan | Could |
| FR-EMP-04 | Komisi berbasis persentase penjualan, nominal per item, atau per layanan | Should |
| FR-EMP-05 | Laporan performa karyawan: total penjualan, jumlah transaksi, rata rata nilai transaksi, komisi | Should |
| FR-EMP-06 | Papan peringkat penjualan antar karyawan (opsional, dapat dimatikan) | Could |

### 8.15 Pengeluaran Operasional (`EXP`)

| Kode | Kebutuhan | Prioritas |
|---|---|---|
| FR-EXP-01 | Kategori pengeluaran yang dapat dikustomisasi (listrik, sewa, gaji, bahan) | Should |
| FR-EXP-02 | Pencatatan pengeluaran dengan tanggal, nominal, kategori, outlet, dan lampiran foto nota | Should |
| FR-EXP-03 | Pengeluaran berulang yang dibuat otomatis (sewa bulanan) | Could |
| FR-EXP-04 | Pengeluaran dari kas kasir langsung tercatat sebagai kas keluar shift | Should |
| FR-EXP-05 | Laporan laba rugi sederhana: omzet dikurangi HPP dikurangi pengeluaran | Should |

### 8.16 Laporan dan Analitik (`RPT`)

| Kode | Kebutuhan | Prioritas |
|---|---|---|
| FR-RPT-01 | Dasbor ringkas: omzet hari ini, jumlah transaksi, rata rata nilai transaksi, perbandingan dengan periode sebelumnya | Must |
| FR-RPT-02 | Laporan penjualan per periode, per outlet, per kasir, per kanal | Must |
| FR-RPT-03 | Laporan penjualan per produk, kategori, dan varian, dengan pengurutan berdasarkan kuantitas atau nilai | Must |
| FR-RPT-04 | Laporan laba kotor per produk berdasarkan HPP tercatat | Should |
| FR-RPT-05 | Laporan metode pembayaran dan rekonsiliasi kas | Must |
| FR-RPT-06 | Laporan pajak dan service charge terkumpul | Should |
| FR-RPT-07 | Laporan diskon, void, dan retur beserta pelaku otorisasi (laporan pengendalian kecurangan) | Must |
| FR-RPT-08 | Laporan jam sibuk (heatmap penjualan per jam dan per hari) | Should |
| FR-RPT-09 | Laporan konsolidasi lintas Business dalam satu Tenant | Should |
| FR-RPT-10 | Perbandingan performa antar Outlet | Should |
| FR-RPT-11 | Ekspor semua laporan ke XLSX, CSV, dan PDF | Must |
| FR-RPT-12 | Laporan terjadwal yang dikirim otomatis lewat email setiap hari atau minggu | Could |
| FR-RPT-13 | Semua laporan menghormati cakupan akses pengguna (kasir hanya melihat datanya sendiri) | Must |
| FR-RPT-14 | Filter rentang tanggal preset (hari ini, kemarin, 7 hari, bulan ini, kustom) | Must |

### 8.17 Perangkat Keras dan Pencetakan (`HW`)

| Kode | Kebutuhan | Prioritas |
|---|---|---|
| FR-HW-01 | Cetak struk ke printer thermal 58mm dan 80mm dengan perintah ESC/POS | Must |
| FR-HW-02 | Koneksi printer via USB, LAN, dan Bluetooth | Must |
| FR-HW-03 | Multi printer per outlet dengan penetapan peran (struk pelanggan, tiket dapur, tiket bar) | Should |
| FR-HW-04 | Buka laci kas otomatis lewat pin printer saat transaksi tunai selesai | Should |
| FR-HW-05 | Dukungan barcode scanner mode keyboard wedge (tanpa driver khusus) | Must |
| FR-HW-06 | Template struk yang dapat dikustomisasi (urutan blok, ukuran font, tampilkan atau sembunyikan elemen) | Should |
| FR-HW-07 | Cetak ulang otomatis saat printer gagal, dengan antrean cetak | Should |
| FR-HW-08 | Integrasi timbangan digital untuk produk berbasis berat | Could |
| FR-HW-09 | Fallback: jika printer tidak tersedia, tampilkan QR struk digital di layar | Should |

### 8.18 Mode Offline dan Sinkronisasi (`OFF`)

| Kode | Kebutuhan | Prioritas |
|---|---|---|
| FR-OFF-01 | Terminal menyimpan salinan lokal master data (produk, harga, pelanggan, promo) | Must |
| FR-OFF-02 | Transaksi baru dapat dibuat, dibayar, dan dicetak saat offline | Must |
| FR-OFF-03 | Setiap transaksi memiliki identitas unik yang dibuat di klien (UUID) untuk mencegah duplikasi saat sinkron | Must |
| FR-OFF-04 | Sinkronisasi otomatis saat koneksi pulih, dengan indikator jumlah transaksi tertunda | Must |
| FR-OFF-05 | Strategi penyelesaian konflik yang jelas: transaksi penjualan bersifat append only dan tidak pernah ditolak server. Konflik stok diselesaikan dengan penyesuaian setelahnya, bukan dengan menolak transaksi | Must |
| FR-OFF-06 | Pembatasan fitur saat offline harus jelas di UI (contoh: QRIS dinamis dan cek poin realtime tidak tersedia) | Must |
| FR-OFF-07 | Peringatan jika terminal offline lebih dari ambang waktu tertentu (data master berpotensi basi) | Should |
| FR-OFF-08 | Nomor struk offline menggunakan prefix terminal agar tidak bentrok | Must |
| FR-OFF-09 | Log sinkronisasi yang dapat diperiksa jika terjadi selisih | Should |

### 8.19 Notifikasi (`NOTIF`)

| Kode | Kebutuhan | Prioritas |
|---|---|---|
| FR-NOTIF-01 | Notifikasi dalam aplikasi untuk stok menipis, shift ditutup dengan selisih, dan order selesai | Should |
| FR-NOTIF-02 | Notifikasi email untuk ringkasan harian dan peringatan penting | Should |
| FR-NOTIF-03 | Integrasi WhatsApp untuk struk digital, status order, dan pengingat piutang | Should |
| FR-NOTIF-04 | Push notification pada PWA untuk pemilik (omzet harian, peringatan) | Could |
| FR-NOTIF-05 | Preferensi notifikasi per pengguna dan per jenis kejadian | Should |

### 8.20 Integrasi (`INT`)

| Kode | Kebutuhan | Prioritas |
|---|---|---|
| FR-INT-01 | Payment gateway untuk QRIS dinamis (Midtrans, Xendit, atau setara) | Should |
| FR-INT-02 | Webhook keluar untuk kejadian penting (transaksi selesai, stok menipis, order status berubah) | Could |
| FR-INT-03 | REST API publik dengan API key per Business, terdokumentasi, dan dibatasi rate limit | Should |
| FR-INT-04 | Ekspor jurnal ke format yang dapat diimpor Accurate atau Jurnal.id | Could |
| FR-INT-05 | Integrasi marketplace makanan (GoFood, GrabFood) untuk menarik pesanan masuk | Won't (v1) |
| FR-INT-06 | Integrasi e-commerce (Tokopedia, Shopee) untuk sinkronisasi stok | Won't (v1) |

### 8.21 Audit dan Keamanan Operasional (`AUD`)

| Kode | Kebutuhan | Prioritas |
|---|---|---|
| FR-AUD-01 | Log seluruh aksi sensitif: void, retur, diskon manual, ubah harga, ubah stok, ubah hak akses, ubah feature flag, login gagal | Must |
| FR-AUD-02 | Log mencatat pelaku, waktu, alamat IP, perangkat, nilai lama, dan nilai baru | Must |
| FR-AUD-03 | Log bersifat append only dan tidak dapat dihapus oleh pengguna tenant | Must |
| FR-AUD-04 | Halaman audit dengan pencarian dan filter, tersedia untuk Pemilik | Must |
| FR-AUD-05 | Retensi log minimal 12 bulan untuk paket berbayar | Should |
| FR-AUD-06 | Laporan aktivitas mencurigakan: void berlebihan, diskon berlebihan, selisih kas berulang oleh kasir tertentu | Should |

### 8.22 Pengaturan dan Kustomisasi (`SET`)

| Kode | Kebutuhan | Prioritas |
|---|---|---|
| FR-SET-01 | Pengaturan bahasa antarmuka (Bahasa Indonesia dan Inggris) | Should |
| FR-SET-02 | Format mata uang, pemisah ribuan, dan jumlah desimal | Must |
| FR-SET-03 | Zona waktu per Outlet, memengaruhi pemotongan laporan harian | Must |
| FR-SET-04 | Jam tutup buku harian yang dapat diatur (misal hari berakhir pukul 03.00 untuk usaha malam) | Should |
| FR-SET-05 | Tema terang dan gelap | Could |
| FR-SET-06 | Kustomisasi tata letak grid kasir (ukuran tombol, tampilkan gambar atau tidak) | Should |
| FR-SET-07 | Backup dan ekspor seluruh data tenant atas permintaan pemilik | Should |

### 8.23 Langganan dan Penagihan Platform (`BILL`)

| Kode | Kebutuhan | Prioritas |
|---|---|---|
| FR-BILL-01 | Paket langganan berjenjang dengan batasan: jumlah outlet, jumlah pengguna, jumlah produk, retensi data, modul tersedia | Must |
| FR-BILL-02 | Masa uji coba gratis dengan durasi yang dapat dikonfigurasi | Should |
| FR-BILL-03 | Upgrade dan downgrade paket dengan perhitungan prorata | Should |
| FR-BILL-04 | Perilaku saat langganan kedaluwarsa: akses baca tetap ada, transaksi baru diblokir. Data tidak dihapus | Must |
| FR-BILL-05 | Riwayat tagihan dan unduh faktur | Should |
| FR-BILL-06 | Pembayaran otomatis lewat payment gateway dan pembayaran manual dengan konfirmasi admin | Should |
| FR-BILL-07 | Penegakan batas paket saat pembuatan entitas baru dengan pesan ajakan upgrade | Must |

### 8.24 Panel Super Admin (`ADM`)

| Kode | Kebutuhan | Prioritas |
|---|---|---|
| FR-ADM-01 | Daftar seluruh tenant dengan status, paket, tanggal daftar, dan aktivitas terakhir | Must |
| FR-ADM-02 | Detail tenant: jumlah bisnis, outlet, pengguna, volume transaksi | Must |
| FR-ADM-03 | Suspend, aktifkan kembali, dan hapus tenant | Must |
| FR-ADM-04 | Kelola katalog modul, paket, dan harga | Must |
| FR-ADM-05 | Impersonasi pengguna tenant untuk dukungan teknis, memerlukan persetujuan tercatat dan selalu masuk audit log | Should |
| FR-ADM-06 | Pengumuman dalam aplikasi ke seluruh tenant atau segmen tertentu | Should |
| FR-ADM-07 | Metrik platform: tenant aktif, MRR, churn, modul terpopuler, error rate | Should |
| FR-ADM-08 | Monitoring kesehatan sistem: antrean job, kegagalan sinkronisasi, kegagalan cetak | Should |
| FR-ADM-09 | Feature flag global untuk merilis fitur secara bertahap (canary release) | Could |

---

## 9. Alur Pengguna Utama

### 9.1 Onboarding Pemilik Bisnis

```
Registrasi
  → Verifikasi email
  → Buat Business (isi nama, pilih tipe bisnis)
  → Sistem menerapkan preset modul
  → Tinjau modul aktif (dapat diubah, dapat dilewati)
  → Buat Outlet pertama (otomatis dibuat "Outlet Utama")
  → Tambah produk (manual, import CSV, atau lewati dengan item terbuka)
  → Buka shift
  → Transaksi pertama
```

Prinsip: setiap langkah setelah pembuatan Business harus dapat dilewati. Pengguna wajib bisa mencapai transaksi pertama tanpa mengisi apa pun selain nama bisnis.

### 9.2 Alur Transaksi Retail

```
Kasir login PIN → Buka shift (jika belum) → Scan barcode
  → Item masuk keranjang → Ulangi
  → [Opsional] Pilih pelanggan
  → [Opsional] Terapkan diskon (cek batas peran)
  → Tekan Bayar → Pilih metode → Input nominal
  → Sistem hitung kembalian → Konfirmasi
  → Cetak struk + buka laci → Transaksi selesai
```

### 9.3 Alur Transaksi F&B Dine In

```
Pilih meja → Input jumlah tamu → Tambah item + modifier
  → Kirim ke dapur (cetak tiket / muncul di KDS)
  → Meja berstatus terisi, bill terbuka
  → [Tambah pesanan susulan, kirim lagi]
  → Pelanggan minta bill → Cetak bill sementara
  → Bayar (dapat split) → Cetak struk
  → Meja berstatus perlu dibersihkan → kosong
```

### 9.4 Alur Order Jasa

```
Buat order → Pilih pelanggan (atau daftar baru)
  → Tambah layanan + kuantitas terukur
  → Catat detail objek + foto kondisi
  → Sistem hitung estimasi selesai
  → Terima DP → Cetak nota + label
  → [Karyawan perbarui status: dikerjakan → selesai]
  → Notifikasi ke pelanggan
  → Pelanggan datang → Pelunasan → Cetak struk → Status diambil
```

### 9.5 Alur Otorisasi Supervisor

```
Kasir melakukan aksi terbatas (contoh: void)
  → Sistem menampilkan dialog otorisasi
  → Supervisor memasukkan PIN di terminal yang sama
  → Sistem memverifikasi izin supervisor
  → Alasan wajib diisi
  → Aksi dijalankan, tercatat atas nama kasir dengan otorisasi supervisor
  → Sesi kasir tidak berubah
```

---

## 10. Kebutuhan Non Fungsional

### 10.1 Performa

| Kode | Kebutuhan |
|---|---|
| NFR-PERF-01 | Waktu tambah item ke keranjang di bawah 100 ms pada perangkat kelas menengah |
| NFR-PERF-02 | Waktu muat awal layar kasir di bawah 3 detik pada koneksi 3G |
| NFR-PERF-03 | Waktu simpan transaksi di bawah 500 ms saat online, dan seketika saat offline |
| NFR-PERF-04 | Laporan dengan rentang 1 bulan tampil di bawah 5 detik. Rentang lebih panjang diproses asinkron dengan notifikasi |
| NFR-PERF-05 | Mendukung katalog hingga 50.000 produk per Business tanpa degradasi pencarian |
| NFR-PERF-06 | Mendukung 50 transaksi per menit per outlet pada jam sibuk |

### 10.2 Skalabilitas dan Ketersediaan

| Kode | Kebutuhan |
|---|---|
| NFR-SCALE-01 | Arsitektur mendukung penambahan instance aplikasi secara horizontal tanpa perubahan kode |
| NFR-SCALE-02 | Pekerjaan berat (laporan, import, sinkronisasi massal) dijalankan lewat antrean terpisah |
| NFR-SCALE-03 | Uptime layanan inti 99,5 persen per bulan |
| NFR-SCALE-04 | Backup basis data harian dengan retensi 30 hari dan uji pemulihan berkala |
| NFR-SCALE-05 | Target RPO 24 jam dan RTO 4 jam |

### 10.3 Keamanan

| Kode | Kebutuhan |
|---|---|
| NFR-SEC-01 | Seluruh lalu lintas menggunakan HTTPS/TLS |
| NFR-SEC-02 | Kata sandi disimpan dengan algoritma hashing modern (bcrypt atau argon2) |
| NFR-SEC-03 | PIN kasir disimpan ter-hash, bukan plaintext, dan dibatasi percobaan gagal |
| NFR-SEC-04 | Rate limiting pada endpoint autentikasi dan API publik |
| NFR-SEC-05 | Perlindungan terhadap OWASP Top 10 |
| NFR-SEC-06 | Tidak menyimpan nomor kartu kredit lengkap dan CVV dalam kondisi apa pun |
| NFR-SEC-07 | Data sensitif dalam penyimpanan lokal terminal dienkripsi |
| NFR-SEC-08 | Uji otomatis wajib memverifikasi isolasi antar tenant pada setiap rilis |

### 10.4 Usability

| Kode | Kebutuhan |
|---|---|
| NFR-UX-01 | Layar kasir dapat dioperasikan penuh dengan sentuhan pada layar 7 inci ke atas |
| NFR-UX-02 | Target sentuh minimal 44x44 piksel pada elemen transaksi |
| NFR-UX-03 | Kasir baru dapat menyelesaikan transaksi pertama tanpa pelatihan formal |
| NFR-UX-04 | Semua aksi destruktif memerlukan konfirmasi eksplisit |
| NFR-UX-05 | Pesan kesalahan menjelaskan penyebab dan langkah perbaikan dalam bahasa awam |
| NFR-UX-06 | Kontras warna memenuhi WCAG AA |

### 10.5 Kompatibilitas

| Kode | Kebutuhan |
|---|---|
| NFR-COMP-01 | Browser: Chrome, Edge, Safari, Firefox dua versi terakhir |
| NFR-COMP-02 | Perangkat kasir: Android 9 ke atas, iPadOS 15 ke atas, Windows 10 ke atas |
| NFR-COMP-03 | PWA dapat dipasang ke layar utama dengan dukungan offline |
| NFR-COMP-04 | Kompatibel dengan printer thermal ESC/POS umum di pasar Indonesia |

### 10.6 Kepatuhan dan Data

| Kode | Kebutuhan |
|---|---|
| NFR-DATA-01 | Data transaksi tidak dapat diubah setelah tersimpan. Koreksi dilakukan lewat transaksi pembalik |
| NFR-DATA-02 | Retensi data transaksi minimal 5 tahun untuk paket berbayar |
| NFR-DATA-03 | Pemilik dapat mengekspor seluruh datanya kapan saja (portabilitas data) |
| NFR-DATA-04 | Penghapusan akun mengikuti alur soft delete lalu purge, dengan pemberitahuan |
| NFR-DATA-05 | Pemrosesan data pribadi mengikuti UU PDP Indonesia |

---

## 11. Model Data Tingkat Tinggi

Entitas utama dan relasinya (bukan skema final):

```
tenants
  ├── subscriptions ── plans ── plan_modules
  ├── users ── user_business_roles ── roles ── permissions
  └── businesses
        ├── business_modules (feature flags)
        ├── outlets
        │     ├── terminals
        │     ├── shifts ── cash_movements
        │     ├── stocks ── stock_movements
        │     ├── tables (fnb)
        │     └── printers
        ├── categories ── products
        │     ├── product_variants ── barcodes
        │     ├── product_units
        │     ├── product_prices (per outlet / channel / tier)
        │     ├── modifier_groups ── modifier_options
        │     └── recipe_items
        ├── customers ── customer_groups
        │     └── receivables ── receivable_payments
        ├── suppliers ── purchase_orders ── purchase_order_items
        │     └── goods_receipts ── payables
        ├── promotions ── promotion_rules ── coupons
        ├── sales
        │     ├── sale_items ── sale_item_modifiers
        │     ├── sale_payments
        │     ├── sale_discounts
        │     └── sale_returns
        ├── service_orders ── service_order_statuses (jasa)
        ├── expenses ── expense_categories
        └── audit_logs
```

Catatan desain penting:

1. **Nilai uang** disimpan sebagai `DECIMAL(15,2)` atau integer dalam satuan terkecil. Tidak pernah float.
2. **Snapshot harga dan HPP** disalin ke `sale_items` saat transaksi. Perubahan harga master tidak boleh mengubah laporan historis.
3. **Snapshot nama produk** juga disimpan agar struk lama tetap terbaca setelah produk diarsipkan.
4. **`sales.uuid`** dibuat di klien untuk idempotensi sinkronisasi offline.
5. **`stock_movements`** adalah sumber kebenaran. Kolom `stocks.quantity` hanya cache yang dapat direkonstruksi.
6. Seluruh tabel transaksional memiliki `tenant_id` dan `business_id` terindeks.

---

## 12. Contoh User Story dan Kriteria Penerimaan

Format: `Sebagai [peran], saya ingin [kebutuhan], agar [manfaat]`.

### US-01 Mengaktifkan modul sesuai kebutuhan

> Sebagai **Pemilik Bisnis**, saya ingin mematikan modul yang tidak saya butuhkan, agar tampilan aplikasi tidak membingungkan karyawan saya.

**Kriteria Penerimaan**
- Diberikan saya berada di halaman Pengaturan Modul, ketika saya mematikan modul "Manajemen Meja", maka menu Meja hilang dari navigasi seluruh pengguna di Business tersebut dalam waktu kurang dari 5 detik.
- Ketika modul dimatikan sementara masih ada 3 open bill aktif, maka sistem menampilkan peringatan berisi jumlah data terdampak dan meminta konfirmasi.
- Ketika saya mengaktifkan kembali modul tersebut, maka seluruh data meja dan bill sebelumnya muncul kembali tanpa kehilangan.
- Ketika saya mencoba mematikan modul "Manajemen Stok" sementara modul "Resep" aktif, maka sistem menolak dan menjelaskan bahwa Resep bergantung padanya.
- Setiap perubahan tercatat di audit log lengkap dengan nama saya dan waktu.

### US-02 Transaksi saat internet mati

> Sebagai **Kasir**, saya ingin tetap bisa melayani pembeli saat internet mati, agar penjualan tidak berhenti.

**Kriteria Penerimaan**
- Ketika koneksi terputus, maka indikator status berubah menjadi "Offline" dan tetap terlihat jelas selama kondisi berlangsung.
- Ketika saya menyelesaikan transaksi tunai saat offline, maka struk tetap tercetak dan nomor struk tetap unik menggunakan prefix terminal.
- Ketika koneksi pulih, maka transaksi tertunda terkirim otomatis dan indikator menampilkan jumlah antrean yang tersisa.
- Ketika transaksi yang sama tersinkron dua kali karena gangguan jaringan, maka server menolak duplikat berdasarkan UUID dan hanya menyimpan satu catatan.
- Ketika saya membuka metode QRIS dinamis saat offline, maka metode tersebut tampil nonaktif dengan penjelasan.

### US-03 Menutup shift tanpa saling curiga

> Sebagai **Kasir**, saya ingin proses tutup kas yang transparan, agar saya tidak disalahkan atas selisih yang bukan kesalahan saya.

**Kriteria Penerimaan**
- Ketika saya menutup shift, maka sistem meminta hitungan fisik uang per pecahan.
- Setelah input selesai, maka sistem menampilkan kas seharusnya, kas aktual, dan selisih.
- Ketika selisih melebihi ambang yang dikonfigurasi, maka sistem meminta alasan dan otorisasi supervisor sebelum shift dapat ditutup.
- Setelah shift ditutup, maka laporan Z tercetak dan tidak dapat diubah lagi.

### US-04 Memantau banyak bisnis dari satu akun

> Sebagai **Pemilik Bisnis** dengan bengkel dan laundry, saya ingin melihat performa keduanya dari satu akun, agar saya tidak perlu berpindah aplikasi.

**Kriteria Penerimaan**
- Ketika saya login, maka saya melihat pemilih Business dan dapat berpindah tanpa login ulang.
- Ketika saya membuka dasbor konsolidasi, maka saya melihat omzet gabungan dan rincian per Business.
- Data master (produk, pelanggan) satu Business tidak muncul di Business lain.
- Setiap Business memiliki konfigurasi modul, pajak, dan format struk yang independen.

### US-05 Mencegah kecurangan diskon

> Sebagai **Pemilik Bisnis**, saya ingin membatasi diskon yang dapat diberikan kasir, agar margin saya terjaga.

**Kriteria Penerimaan**
- Ketika saya mengatur batas diskon kasir 5 persen, maka kasir tidak dapat menerapkan diskon di atas nilai tersebut tanpa otorisasi.
- Ketika kasir memasukkan diskon 10 persen, maka muncul dialog PIN supervisor dan kolom alasan wajib.
- Setelah otorisasi berhasil, maka transaksi tercatat dengan nama kasir dan nama supervisor pemberi izin.
- Laporan diskon menampilkan seluruh diskon manual beserta pemberi otorisasi dan alasannya.

---

## 13. Rencana Rilis Bertahap

### Fase 1: MVP (target 8 sampai 10 minggu)

Tujuan: satu tenant dapat berjualan sungguhan.

- Auth, RBAC dasar dengan peran bawaan
- Tenant, Business, Outlet, Terminal
- Sistem feature flag (kerangka + preset)
- Produk sederhana, kategori, barcode
- Transaksi penjualan, diskon, pajak, pembayaran tunai dan non tunai manual
- Cetak struk thermal
- Shift kas
- Stok dasar dan penyesuaian
- Laporan penjualan harian dan per produk
- Audit log

Tidak termasuk: F&B, jasa, promo, loyalitas, pembelian, offline penuh.

### Fase 2: Vertical Expansion (target 6 sampai 8 minggu)

- Modul F&B: meja, open bill, modifier, tiket dapur
- Modul Jasa: order berdurasi, DP, status, lacak publik
- Varian produk
- Pelanggan dan riwayat
- Retur dan void lengkap
- Laporan diperluas

### Fase 3: Operasional Lanjut (target 6 sampai 8 minggu)

- Pembelian, supplier, hutang
- Stok opname dan transfer antar outlet
- Resep dan bahan baku
- Promo dan loyalitas
- Piutang
- Pengeluaran dan laba rugi sederhana

### Fase 4: Ketahanan dan Skala (target 6 minggu)

- Mode offline penuh dan sinkronisasi
- KDS
- Multi satuan dan harga bertingkat
- Batch dan kedaluwarsa
- API publik dan webhook

### Fase 5: Monetisasi dan Ekosistem

- Paket langganan, penagihan, prorata
- Panel Super Admin lengkap
- Integrasi payment gateway QRIS
- Integrasi WhatsApp
- Laporan terjadwal

---

## 14. Risiko dan Mitigasi

| Risiko | Dampak | Mitigasi |
|---|---|---|
| Kompleksitas feature flag membuat pengujian meledak (kombinasi modul terlalu banyak) | Tinggi | Uji berdasarkan preset tipe bisnis, bukan semua kombinasi. Tetapkan dependensi ketat agar kombinasi tidak valid tidak mungkin terjadi |
| Kebocoran data antar tenant | Sangat tinggi | Global scope di lapisan ORM, uji otomatis isolasi wajib di CI, code review khusus untuk query manual |
| Sinkronisasi offline menyebabkan duplikasi atau stok kacau | Tinggi | UUID dari klien, operasi idempoten, stok direkonstruksi dari ledger bukan dari counter |
| Kompatibilitas printer thermal yang beragam | Sedang | Batasi pada ESC/POS standar, sediakan daftar printer teruji, fallback struk digital |
| Produk terlalu besar untuk dikerjakan sendiri | Tinggi | Rilis bertahap per fase, MVP sempit, tolak permintaan fitur di luar fase berjalan |
| Pengguna kecil tetap merasa aplikasi rumit | Sedang | Preset "Minimal", onboarding yang dapat dilewati, sembunyikan pengaturan lanjutan secara default |
| Perubahan harga master merusak laporan historis | Tinggi | Snapshot harga dan HPP pada baris transaksi |
| Persaingan dengan POS gratis dari pemain besar | Sedang | Fokus pada segmen multi bisnis dan multi vertical yang tidak dilayani pemain besar |

---

## 15. Asumsi

1. Mayoritas pengguna berada di Indonesia dan memakai Rupiah.
2. Perangkat kasir umumnya tablet Android atau laptop kelas menengah, bukan mesin POS khusus.
3. Koneksi internet tersedia sebagian besar waktu, dengan gangguan berkala.
4. Pengguna tidak memiliki latar belakang akuntansi.
5. Regulasi pajak yang relevan pada v1 hanya PPN dan PB1 dengan tarif yang dapat dikonfigurasi manual.

---

## 16. Pertanyaan Terbuka

| No | Pertanyaan | Perlu Diputuskan Sebelum |
|---|---|---|
| Q1 | Apakah harga langganan berbasis per outlet, per pengguna, atau per volume transaksi? | Fase 5 |
| Q2 | Apakah mode offline perlu tersedia di semua paket atau hanya paket atas? | Fase 4 |
| Q3 | Apakah metode penilaian persediaan FIFO benar benar dibutuhkan di pasar target, atau Average Cost sudah cukup? | Fase 3 |
| Q4 | Seberapa dalam integrasi WhatsApp: unofficial gateway (murah, berisiko diblokir) atau WhatsApp Business API resmi (mahal, stabil)? | Fase 5 |
| Q5 | Apakah KDS berupa halaman web di tablet, atau perlu aplikasi terpisah? | Fase 4 |
| Q6 | Bagaimana perlakuan data tenant yang berhenti berlangganan lebih dari 12 bulan? | Fase 5 |
| Q7 | Apakah perlu mendukung multi mata uang dalam satu tenant? | Sebelum desain skema harga |

---

## 17. Glosarium

| Istilah | Arti |
|---|---|
| Tenant | Akun pemilik bisnis, unit isolasi data dan langganan |
| Business | Unit usaha dengan tipe tertentu di bawah satu Tenant |
| Outlet | Cabang atau lokasi fisik penjualan |
| Terminal | Perangkat kasir terdaftar dengan penomoran struk sendiri |
| Feature Flag | Sakelar aktivasi modul per Business |
| Entitlement | Hak akses modul yang diberikan oleh paket langganan |
| Open Bill | Tagihan yang masih terbuka dan dapat ditambah pesanan |
| KDS | Kitchen Display System, layar pesanan untuk dapur |
| HPP | Harga Pokok Penjualan, biaya perolehan barang yang terjual |
| Z Report | Laporan penutupan shift yang bersifat final |
| Stock Ledger | Catatan kronologis seluruh mutasi stok |
| PB1 | Pajak Barang dan Jasa Tertentu, pajak daerah untuk restoran |
| Idempotensi | Sifat operasi yang menghasilkan efek sama meski dijalankan berulang |

---

## 18. Lampiran: Katalog Modul untuk Implementasi

Referensi kode modul yang dipakai di tabel `modules` dan `business_modules`.

| Kode Modul | Nama | Dependensi | Kelompok |
|---|---|---|---|
| `sales.core` | Penjualan Dasar | - | Inti (selalu aktif) |
| `shift.core` | Shift Kas | `sales.core` | Inti (selalu aktif) |
| `catalog.barcode` | Barcode dan SKU | - | Katalog |
| `catalog.variant` | Varian Produk | - | Katalog |
| `catalog.modifier` | Modifier dan Add On | - | Katalog |
| `catalog.unit` | Multi Satuan | - | Katalog |
| `catalog.pricing_tier` | Harga Bertingkat | - | Katalog |
| `catalog.channel_price` | Harga per Kanal | - | Katalog |
| `catalog.bundle` | Paket dan Bundling | - | Katalog |
| `inventory.core` | Manajemen Stok | - | Persediaan |
| `inventory.opname` | Stok Opname | `inventory.core` | Persediaan |
| `inventory.transfer` | Transfer Antar Outlet | `inventory.core` | Persediaan |
| `inventory.recipe` | Resep dan Bahan Baku | `inventory.core` | Persediaan |
| `inventory.batch` | Batch dan Kedaluwarsa | `inventory.core` | Persediaan |
| `inventory.serial` | Serial Number | `inventory.core` | Persediaan |
| `fnb.table` | Manajemen Meja | - | F&B |
| `fnb.openbill` | Open Bill | `fnb.table` | F&B |
| `fnb.kds` | Kitchen Display | `fnb.openbill` | F&B |
| `fnb.splitbill` | Split Bill | `fnb.openbill` | F&B |
| `service.order` | Order Berdurasi | - | Jasa |
| `service.tracking` | Lacak Status Publik | `service.order` | Jasa |
| `service.assignment` | Penugasan Teknisi | `service.order` | Jasa |
| `payment.split` | Pembayaran Terpisah | - | Pembayaran |
| `payment.dp` | Uang Muka dan Cicilan | - | Pembayaran |
| `payment.gateway` | QRIS Dinamis | - | Pembayaran |
| `customer.core` | Data Pelanggan | - | Pelanggan |
| `customer.loyalty` | Poin dan Member | `customer.core` | Pelanggan |
| `customer.receivable` | Piutang | `customer.core` | Pelanggan |
| `promo.core` | Promo dan Diskon Otomatis | - | Promo |
| `promo.coupon` | Kupon dan Voucher | `promo.core` | Promo |
| `purchase.core` | Pembelian dan Supplier | `inventory.core` | Pembelian |
| `purchase.payable` | Hutang Usaha | `purchase.core` | Pembelian |
| `employee.core` | Data Karyawan | - | Karyawan |
| `employee.attendance` | Absensi | `employee.core` | Karyawan |
| `employee.commission` | Komisi | `employee.core` | Karyawan |
| `expense.core` | Pengeluaran Operasional | - | Keuangan |
| `offline.mode` | Mode Offline | - | Sistem |
| `notif.whatsapp` | Notifikasi WhatsApp | - | Sistem |
| `api.public` | REST API Publik | - | Sistem |

---

## 19. Riwayat Revisi

| Versi | Tanggal | Perubahan | Penulis |
|---|---|---|---|
| 1.0 | 10 Agustus 2026 | Draft awal | Radityo |
