# User Stories
## Aplikasi Point of Sale (POS) Multi-Tenant

| Item | Keterangan |
|---|---|
| Dokumen Induk | PRD POS Multi-Tenant v1.0 |
| Versi | 1.0 |
| Tanggal | 10 Agustus 2026 |
| Total Story | 78 |

---

## 1. Konvensi

### 1.1 Format

```
US-XXX  Judul singkat
Sebagai [peran], saya ingin [kebutuhan], agar [manfaat].

Kriteria Penerimaan (Given / When / Then)
Referensi FR, Prioritas, Estimasi, Fase
```

### 1.2 Skala Estimasi

Story point Fibonacci: 1 (sepele), 2 (kecil), 3 (sedang), 5 (besar), 8 (sangat besar), 13 (perlu dipecah).

### 1.3 Definition of Ready

Sebuah story siap dikerjakan bila: kriteria penerimaan lengkap dan dapat diuji, dependensi teridentifikasi, desain UI tersedia untuk story yang menyentuh antarmuka, dan estimasi disepakati.

### 1.4 Definition of Done

Kode ditulis dan direview, uji otomatis lulus termasuk uji isolasi tenant, analisis statis lulus, dokumentasi API diperbarui bila ada perubahan kontrak, feature flag terpasang bila story berada di dalam modul opsional, dan diverifikasi di staging.

### 1.5 Daftar Epik

| Epik | Nama | Story | Fase |
|---|---|---|---|
| EP-01 | Onboarding dan Multi-Tenancy | 7 | 1 |
| EP-02 | Feature Flag dan Konfigurasi Modul | 5 | 1 |
| EP-03 | Identitas, Peran, dan Otorisasi | 8 | 1 |
| EP-04 | Katalog Produk | 9 | 1 sampai 2 |
| EP-05 | Transaksi Penjualan | 12 | 1 sampai 2 |
| EP-06 | Pembayaran | 6 | 1 sampai 5 |
| EP-07 | Shift dan Kas | 5 | 1 |
| EP-08 | Persediaan | 8 | 1 sampai 3 |
| EP-09 | F&B | 6 | 2 sampai 4 |
| EP-10 | Jasa | 5 | 2 |
| EP-11 | Pelanggan, Promo, Loyalitas | 6 | 2 sampai 3 |
| EP-12 | Pembelian | 4 | 3 |
| EP-13 | Laporan dan Analitik | 6 | 1 sampai 3 |
| EP-14 | Offline dan Sinkronisasi | 5 | 4 |
| EP-15 | Platform dan Langganan | 6 | 5 |

---

## EP-01 Onboarding dan Multi-Tenancy

### US-001 Registrasi mandiri pemilik bisnis
> Sebagai **calon pengguna**, saya ingin mendaftar sendiri tanpa menghubungi sales, agar saya bisa langsung mencoba aplikasi.

**Kriteria Penerimaan**
- Diberikan saya di halaman registrasi, ketika saya mengisi nama, email, dan kata sandi yang valid, maka akun tenant dibuat dengan status `trial` dan saya langsung masuk ke wizard pembuatan bisnis.
- Ketika email sudah terdaftar, maka sistem menampilkan pesan yang mengarahkan ke halaman login, bukan pesan error teknis.
- Ketika kata sandi kurang dari 8 karakter, maka validasi menolak sebelum permintaan dikirim.
- Setelah registrasi, maka email verifikasi terkirim dan saya tetap dapat memakai aplikasi selama masa tenggang 7 hari tanpa verifikasi.
- Slug tenant dibuat otomatis dari nama dan dijamin unik.

FR-AUTH-01, FR-MT-01 | Must | 3 | Fase 1

### US-002 Membuat bisnis dengan preset tipe
> Sebagai **Pemilik Bisnis**, saya ingin memilih tipe usaha saat membuat bisnis, agar aplikasi langsung menampilkan fitur yang relevan tanpa saya konfigurasi satu per satu.

**Kriteria Penerimaan**
- Diberikan saya di wizard, ketika saya memilih tipe "F&B", maka modul meja, open bill, modifier, dan KDS otomatis aktif sesuai tabel preset di PRD.
- Ketika saya memilih tipe "Minimal", maka hanya modul inti yang aktif.
- Setelah preset diterapkan, maka saya melihat ringkasan modul aktif dengan opsi mengubahnya sebelum melanjutkan.
- Ketika saya melewati langkah tinjauan, maka preset tetap tersimpan.
- Outlet pertama dibuat otomatis dengan nama "Outlet Utama" dan dapat diganti kemudian.
- Terminal pertama dibuat otomatis dengan kode `K01` dan status aktif.

FR-MT-01, FR-MT-02, FR-FF-01 | Must | 5 | Fase 1

### US-003 Transaksi pertama tanpa setup panjang
> Sebagai **Pemilik Bisnis baru**, saya ingin bisa melakukan transaksi pertama tanpa harus memasukkan produk dulu, agar saya bisa menilai aplikasi dengan cepat.

**Kriteria Penerimaan**
- Diberikan saya baru selesai membuat bisnis dan belum punya produk, ketika saya membuka layar kasir, maka tombol "Item Cepat" tersedia untuk memasukkan nama dan harga bebas.
- Ketika saya menyelesaikan transaksi dengan item cepat, maka transaksi tersimpan normal dan muncul di laporan.
- Setiap langkah onboarding selain pembuatan bisnis dapat dilewati.
- Waktu dari selesai registrasi sampai transaksi pertama dapat ditempuh di bawah 15 menit oleh pengguna baru dalam uji usability.

FR-SALE-14, G-02 | Must | 3 | Fase 1

### US-004 Mengelola banyak bisnis dalam satu akun
> Sebagai **Pemilik Bisnis** yang punya bengkel dan laundry, saya ingin mengelola keduanya dari satu akun, agar saya tidak perlu login berkali kali.

**Kriteria Penerimaan**
- Diberikan saya punya dua bisnis, ketika saya login, maka pemilih bisnis muncul di header dan saya dapat berpindah tanpa login ulang.
- Ketika saya berpindah bisnis, maka seluruh data yang ditampilkan berganti sepenuhnya, termasuk navigasi yang menyesuaikan modul aktif bisnis tersebut.
- Data master satu bisnis tidak pernah muncul di bisnis lain, diverifikasi lewat uji otomatis pada setiap endpoint.
- Konteks bisnis yang dipilih tersimpan di sesi dan bertahan setelah refresh.

FR-MT-01, FR-MT-04, FR-AUTH-08 | Must | 5 | Fase 1

### US-005 Menambah outlet baru
> Sebagai **Pemilik Bisnis** yang membuka cabang, saya ingin menambah outlet, agar penjualan cabang tercatat terpisah namun tetap dalam satu bisnis.

**Kriteria Penerimaan**
- Ketika saya menambah outlet, maka outlet mewarisi pengaturan pajak dan pembulatan dari bisnis, dan dapat dioverride.
- Master produk otomatis tersedia di outlet baru dengan stok awal nol.
- Ketika paket langganan saya membatasi jumlah outlet dan batas tercapai, maka sistem menolak dengan ajakan upgrade yang menyebutkan paket yang dibutuhkan.
- Laporan dapat difilter per outlet dan dilihat gabungan.

FR-ORG-02, FR-BILL-07 | Must | 3 | Fase 1

### US-006 Mendaftarkan terminal kasir
> Sebagai **Manajer**, saya ingin mendaftarkan dan mengaktifkan perangkat kasir, agar tidak sembarang perangkat bisa bertransaksi atas nama outlet saya.

**Kriteria Penerimaan**
- Diberikan perangkat baru membuka aplikasi, ketika perangkat memasukkan kode pairing dari back office, maka perangkat terdaftar dengan status menunggu aktivasi.
- Perangkat tidak dapat bertransaksi sebelum saya mengaktifkannya.
- Setiap terminal memiliki kode prefix unik dalam outlet yang dipakai pada nomor struk.
- Saya dapat mencabut perangkat jarak jauh, dan setelah dicabut perangkat tersebut langsung kehilangan akses pada permintaan berikutnya.
- Daftar perangkat menampilkan terakhir aktif dan jumlah transaksi yang belum tersinkron.

FR-ORG-03, FR-ORG-04, FR-AUTH-10 | Must | 5 | Fase 1

### US-007 Mengatur pajak dan pembulatan
> Sebagai **Pemilik Bisnis**, saya ingin mengatur pajak dan pembulatan sesuai jenis usaha saya, agar total di struk sesuai praktik yang berlaku.

**Kriteria Penerimaan**
- Ketika saya mengaktifkan PPN 11 persen eksklusif, maka pajak ditambahkan di atas subtotal dan tampil terpisah di struk.
- Ketika saya memilih pajak inklusif, maka harga jual tidak berubah dan pajak diekstraksi dari harga, dengan rincian tetap tampil di struk.
- Ketika saya mengatur pembulatan ke 100 terdekat, maka grand total dibulatkan dan selisih pembulatan tampil sebagai baris tersendiri.
- Perubahan tarif pajak tidak mengubah transaksi yang sudah terjadi.
- Pengaturan dapat dioverride per outlet.

FR-ORG-05, FR-ORG-06, FR-ORG-08 | Must | 5 | Fase 1

---

## EP-02 Feature Flag dan Konfigurasi Modul

### US-008 Mengaktifkan dan mematikan modul
> Sebagai **Pemilik Bisnis**, saya ingin mematikan fitur yang tidak saya butuhkan, agar karyawan saya tidak bingung dengan menu yang tidak terpakai.

**Kriteria Penerimaan**
- Diberikan saya di halaman Pengaturan Modul, ketika saya mematikan modul "Manajemen Meja", maka menu Meja hilang dari navigasi seluruh pengguna bisnis tersebut dalam waktu di bawah 5 detik tanpa perlu login ulang.
- Ketika modul mati, maka endpoint API terkait mengembalikan `403 MODULE_DISABLED`, bukan hanya disembunyikan di UI.
- Ketika modul mati, maka field terkait hilang dari form (contoh: bagian Modifier hilang dari form produk).
- Ketika saya mengaktifkan kembali, maka seluruh data sebelumnya muncul utuh.
- Setiap perubahan tercatat di audit log dengan nama saya, waktu, dan nilai sebelum sesudah.

FR-FF-04, FR-FF-06, FR-FF-07 | Must | 8 | Fase 1

### US-009 Peringatan saat mematikan modul yang berisi data aktif
> Sebagai **Pemilik Bisnis**, saya ingin diperingatkan sebelum mematikan modul yang sedang dipakai, agar saya tidak mengacaukan operasional yang sedang berjalan.

**Kriteria Penerimaan**
- Diberikan ada 3 open bill aktif, ketika saya mematikan modul Open Bill, maka sistem menampilkan peringatan berisi jumlah data terdampak dan meminta konfirmasi eksplisit.
- Ketika saya membatalkan, maka modul tetap aktif dan tidak ada perubahan tersimpan.
- Ketika saya melanjutkan, maka data tersimpan dan hanya disembunyikan.
- Peringatan menjelaskan konsekuensi dalam bahasa awam, bukan istilah teknis.

FR-FF-05 | Must | 3 | Fase 1

### US-010 Dependensi antar modul
> Sebagai **Pemilik Bisnis**, saya ingin sistem mengurus keterkaitan antar fitur, agar saya tidak mengaktifkan fitur yang tidak akan berfungsi.

**Kriteria Penerimaan**
- Ketika saya mengaktifkan modul "Resep" sementara "Manajemen Stok" mati, maka sistem menawarkan mengaktifkan keduanya sekaligus dengan penjelasan alasannya.
- Ketika saya mematikan "Manajemen Stok" sementara "Resep" aktif, maka sistem menolak dan menyebutkan modul mana yang menghalangi.
- Dependensi diselesaikan rekursif, sehingga mengaktifkan modul dengan rantai dependensi tiga tingkat berhasil dalam satu operasi.
- Seluruh perubahan dependensi tersimpan dalam satu transaksi database.

FR-FF-02, FR-FF-03 | Must | 5 | Fase 1

### US-011 Melihat fitur yang terkunci paket
> Sebagai **Pemilik Bisnis**, saya ingin tahu fitur apa yang belum saya miliki, agar saya bisa menilai apakah perlu upgrade.

**Kriteria Penerimaan**
- Diberikan paket saya tidak mencakup modul KDS, ketika saya membuka halaman modul, maka KDS tampil dengan ikon gembok, deskripsi manfaat, dan nama paket minimum yang dibutuhkan.
- Ketika saya mencoba mengaktifkan modul terkunci, maka saya diarahkan ke halaman perbandingan paket, bukan pesan error.
- Modul yang tidak tersedia di paket mana pun tidak ditampilkan sama sekali.

FR-FF-08, FR-BILL-07 | Should | 3 | Fase 5

### US-012 Menyalin konfigurasi modul antar bisnis
> Sebagai **Pemilik Bisnis** dengan beberapa cabang usaha sejenis, saya ingin menyalin konfigurasi dari bisnis yang sudah jadi, agar saya tidak mengatur ulang dari awal.

**Kriteria Penerimaan**
- Ketika saya memilih bisnis sumber, maka sistem menampilkan pratinjau modul mana yang akan aktif dan mana yang akan mati.
- Modul yang tidak tersedia di paket saat ini dilewati dengan pemberitahuan.
- Penyalinan hanya menyentuh konfigurasi modul, tidak menyentuh data master atau transaksi.

FR-FF-09 | Could | 3 | Fase 5

---

## EP-03 Identitas, Peran, dan Otorisasi

### US-013 Mengundang karyawan
> Sebagai **Pemilik Bisnis**, saya ingin mengundang karyawan dengan peran tertentu, agar mereka bisa bekerja sesuai tanggung jawabnya.

**Kriteria Penerimaan**
- Ketika saya mengirim undangan, maka penerima menerima email berisi tautan yang kedaluwarsa dalam 7 hari.
- Ketika penerima sudah punya akun, maka menerima undangan cukup menambahkan peran baru tanpa membuat akun kedua.
- Saya dapat membatasi cakupan akses ke outlet tertentu saat mengundang.
- Undangan yang belum diterima dapat dibatalkan atau dikirim ulang.
- Batas jumlah pengguna sesuai paket ditegakkan saat undangan dikirim, bukan saat diterima.

FR-AUTH-06, FR-RBAC-04 | Must | 5 | Fase 1

### US-014 Login cepat kasir dengan PIN
> Sebagai **Kasir**, saya ingin masuk dengan PIN singkat, agar pergantian shift di terminal berjalan cepat.

**Kriteria Penerimaan**
- Diberikan terminal sudah terotorisasi, ketika saya memasukkan PIN yang benar, maka saya masuk ke layar kasir dalam waktu di bawah 2 detik.
- Ketika PIN salah 5 kali berturut turut, maka akun terkunci 15 menit dan pesan menjelaskan sisa waktu.
- PIN tersimpan dalam bentuk hash, tidak pernah plaintext, dan tidak pernah dikembalikan oleh API.
- PIN unik dalam satu outlet, sehingga tidak ada dua karyawan dengan PIN sama.
- PIN dapat diverifikasi saat offline menggunakan hash yang telah disinkronkan.

FR-AUTH-03, NFR-SEC-03 | Must | 5 | Fase 1

### US-015 Otorisasi supervisor tanpa ganti sesi
> Sebagai **Kasir**, saya ingin supervisor cukup memasukkan PIN di terminal saya untuk menyetujui aksi khusus, agar pelanggan tidak menunggu lama.

**Kriteria Penerimaan**
- Diberikan saya memicu aksi yang melampaui izin saya, ketika dialog otorisasi muncul dan supervisor memasukkan PIN yang valid, maka aksi dijalankan dan sesi saya tetap aktif.
- Ketika PIN milik pengguna yang juga tidak punya izin tersebut, maka sistem menolak dengan pesan yang jelas.
- Token otorisasi berumur maksimal 90 detik dan terikat pada aksi serta transaksi tertentu, sehingga tidak dapat dipakai ulang.
- Alasan wajib diisi sebelum aksi dijalankan.
- Catatan otorisasi menyimpan kasir, supervisor, aksi, konteks, dan alasan.

FR-RBAC-05, FR-RBAC-06, FR-AUD-01 | Must | 8 | Fase 1

### US-016 Membatasi diskon per peran
> Sebagai **Pemilik Bisnis**, saya ingin membatasi besaran diskon yang bisa diberikan kasir, agar margin saya terjaga.

**Kriteria Penerimaan**
- Ketika saya mengatur batas diskon kasir 5 persen, maka input diskon di atas nilai tersebut memicu dialog otorisasi supervisor.
- Batas berlaku untuk diskon per item maupun per transaksi.
- Batas dalam nominal dan persentase keduanya didukung.
- Laporan diskon menampilkan seluruh diskon manual beserta pemberi otorisasi dan alasannya.

FR-RBAC-07, FR-SALE-05, FR-RPT-07 | Should | 5 | Fase 1

### US-017 Peran kustom
> Sebagai **Pemilik Bisnis**, saya ingin membuat peran sendiri, agar cocok dengan struktur tim saya yang tidak standar.

**Kriteria Penerimaan**
- Ketika saya membuat peran baru, maka saya dapat memilih izin secara granular yang dikelompokkan per modul.
- Izin milik modul yang mati tidak ditampilkan.
- Peran bawaan tidak dapat dihapus, tetapi dapat disalin sebagai dasar peran baru.
- Peran yang masih dipakai pengguna tidak dapat dihapus sebelum pengguna dipindahkan.

FR-RBAC-02, FR-RBAC-03 | Should | 5 | Fase 2

### US-018 Menonaktifkan karyawan yang keluar
> Sebagai **Pemilik Bisnis**, saya ingin menonaktifkan akses karyawan yang berhenti, agar data saya aman tanpa kehilangan riwayat transaksinya.

**Kriteria Penerimaan**
- Ketika saya menonaktifkan karyawan, maka sesi aktifnya dicabut pada permintaan berikutnya.
- Riwayat transaksi atas namanya tetap utuh dan tetap muncul di laporan.
- Karyawan nonaktif tidak muncul di daftar pilihan pramuniaga pada transaksi baru.
- Ketika karyawan tersebut kembali bekerja, maka akun dapat diaktifkan lagi dengan riwayat yang tersambung.

FR-AUTH-07 | Must | 3 | Fase 1

### US-019 Melihat riwayat perangkat dan sesi
> Sebagai **Pemilik Bisnis**, saya ingin melihat perangkat apa saja yang mengakses akun bisnis saya, agar saya bisa mendeteksi akses yang tidak wajar.

**Kriteria Penerimaan**
- Daftar menampilkan nama perangkat, platform, versi aplikasi, terakhir aktif, dan lokasi kasar berdasarkan IP.
- Saya dapat mencabut sesi tertentu.
- Perangkat dengan transaksi belum tersinkron ditandai khusus dengan jumlahnya, dan sistem memperingatkan sebelum saya mencabutnya.

FR-AUTH-10 | Should | 3 | Fase 2

### US-020 Dua faktor untuk peran sensitif
> Sebagai **Pemilik Bisnis**, saya ingin mengaktifkan verifikasi dua langkah, agar akun saya lebih aman.

**Kriteria Penerimaan**
- Ketika saya mengaktifkan 2FA, maka sistem menampilkan QR untuk aplikasi authenticator dan kode pemulihan sekali pakai.
- 2FA hanya berlaku untuk login berbasis kata sandi, tidak untuk login PIN kasir di terminal terotorisasi.
- Saya dapat mewajibkan 2FA untuk peran Manajer ke atas di bisnis saya.

FR-AUTH-05 | Should | 5 | Fase 2

---

## EP-04 Katalog Produk

### US-021 Menambah produk sederhana
> Sebagai **Pemilik Bisnis**, saya ingin menambah produk dengan cepat, agar katalog saya siap dipakai berjualan.

**Kriteria Penerimaan**
- Form minimum hanya meminta nama dan harga. Field lain opsional.
- Field yang tampil menyesuaikan modul aktif: SKU dan barcode hanya muncul jika modul barcode aktif, modifier hanya jika modul modifier aktif.
- Ketika saya menyimpan, maka produk langsung tersedia di layar kasir tanpa perlu refresh manual pada terminal aktif.
- Varian default dibuat otomatis di belakang layar meski saya tidak mengisi varian apa pun.

FR-PRD-01, FR-PRD-02 | Must | 5 | Fase 1

### US-022 Import produk massal
> Sebagai **Pemilik Bisnis** yang pindah dari aplikasi lain, saya ingin mengunggah katalog dari file, agar saya tidak mengetik ratusan produk satu per satu.

**Kriteria Penerimaan**
- Sistem menyediakan template CSV dan XLSX yang dapat diunduh.
- Ketika file diunggah, maka sistem menampilkan pratinjau 10 baris pertama beserta pemetaan kolom sebelum saya konfirmasi.
- Ketika ada baris tidak valid, maka sistem memproses baris yang valid dan menghasilkan file laporan kesalahan berisi nomor baris dan alasan.
- Import 1.000 produk selesai di bawah 60 detik dan diproses lewat antrean dengan indikator kemajuan.
- Ketika SKU sudah ada, maka saya dapat memilih perilaku: lewati, perbarui, atau buat baru.

FR-PRD-17 | Must | 8 | Fase 1

### US-023 Produk dengan varian
> Sebagai **Pemilik Toko Baju**, saya ingin satu produk punya beberapa ukuran dan warna, agar stok tiap kombinasi terpantau terpisah.

**Kriteria Penerimaan**
- Ketika saya mendefinisikan atribut Ukuran (S, M, L) dan Warna (Merah, Biru), maka sistem menghasilkan 6 kombinasi varian.
- Setiap varian dapat memiliki harga, SKU, barcode, dan stok sendiri.
- Saya dapat menghapus kombinasi yang tidak dijual sebelum menyimpan.
- Di layar kasir, memilih produk bervarian menampilkan pemilih varian, dan varian yang stoknya habis ditandai.

FR-PRD-04 | Must | 8 | Fase 2

### US-024 Modifier untuk menu minuman
> Sebagai **Pemilik Kedai Kopi**, saya ingin pelanggan bisa memilih level gula dan topping, agar pesanan sesuai permintaan dan harga menyesuaikan.

**Kriteria Penerimaan**
- Ketika saya membuat grup modifier "Level Gula" dengan tipe pilihan tunggal dan wajib, maka kasir tidak dapat menambahkan item ke keranjang sebelum memilih.
- Ketika grup "Topping" bertipe pilihan ganda dengan maksimal 3, maka kasir tidak dapat memilih opsi keempat.
- Harga tambahan modifier ikut terhitung dalam subtotal item dan tampil di struk sebagai baris di bawah item.
- Modifier tampil di tiket dapur.
- Opsi modifier dapat ditandai habis, dan langsung nonaktif di semua terminal.

FR-PRD-05, FR-PRD-06, FR-FNB-11 | Must | 8 | Fase 2

### US-025 Harga berbeda per kanal
> Sebagai **Pemilik Restoran**, saya ingin harga ojek online berbeda dari harga makan di tempat, agar komisi platform tidak menggerus margin.

**Kriteria Penerimaan**
- Ketika saya membuat daftar harga untuk kanal "online", maka memilih kanal tersebut di kasir mengubah harga seluruh item yang terdaftar.
- Item yang tidak ada di daftar harga khusus memakai harga dasar.
- Struk dan laporan menampilkan kanal transaksi.
- Laporan dapat memisahkan omzet per kanal.

FR-PRD-12, FR-SALE-13 | Should | 5 | Fase 3

### US-026 Harga grosir bertingkat
> Sebagai **Pemilik Toko Grosir**, saya ingin harga turun otomatis saat pembelian banyak, agar kasir tidak perlu menghitung manual.

**Kriteria Penerimaan**
- Ketika saya mengatur harga 10.000 untuk 1 sampai 11 pcs dan 9.000 untuk 12 pcs ke atas, maka menambahkan 12 pcs otomatis memakai harga 9.000 untuk seluruh kuantitas.
- Perubahan kuantitas di keranjang memicu perhitungan ulang harga secara langsung.
- Harga yang dipakai dan sumbernya tersimpan pada baris transaksi untuk audit.

FR-PRD-10 | Should | 5 | Fase 4

### US-027 Multi satuan
> Sebagai **Pemilik Toko Grosir**, saya ingin menjual per pcs dan per dus dari stok yang sama, agar stok tidak terpecah.

**Kriteria Penerimaan**
- Ketika saya mendefinisikan 1 dus sama dengan 24 pcs, maka menjual 1 dus mengurangi stok 24 pcs.
- Barcode dus dapat berbeda dari barcode pcs dan langsung memilih satuan yang tepat saat dipindai.
- Laporan stok menampilkan dalam satuan dasar dengan opsi konversi tampilan.

FR-PRD-09, FR-PRD-07 | Should | 8 | Fase 4

### US-028 Resep bahan baku
> Sebagai **Pemilik Kedai Kopi**, saya ingin penjualan kopi otomatis mengurangi stok biji dan susu, agar saya tahu kapan harus belanja.

**Kriteria Penerimaan**
- Ketika saya mendefinisikan resep 1 gelas Latte memakai 18 gram biji dan 150 ml susu, maka menjual 2 gelas mengurangi 36 gram dan 300 ml.
- Ketika bahan baku habis, maka perilaku mengikuti pengaturan izinkan stok minus di tingkat bisnis.
- Mutasi stok tercatat dengan tipe `recipe_consumption` dan menunjuk ke transaksi penjualannya.
- Laporan pemakaian bahan baku tersedia per periode.
- HPP produk jadi dihitung dari total HPP bahan baku pada saat transaksi.

FR-PRD-13, FR-INV-09 | Should | 8 | Fase 3

### US-029 Mengarsipkan produk
> Sebagai **Pemilik Bisnis**, saya ingin menghentikan penjualan produk lama tanpa menghapusnya, agar laporan historis tetap utuh.

**Kriteria Penerimaan**
- Ketika saya mengarsipkan produk, maka produk hilang dari layar kasir tetapi tetap muncul di laporan periode lampau.
- Struk lama tetap menampilkan nama produk dengan benar karena memakai snapshot.
- Produk terarsip dapat dikembalikan kapan saja.
- Produk dengan stok tersisa memicu peringatan sebelum diarsipkan.

FR-PRD-19 | Must | 3 | Fase 1

---

## EP-05 Transaksi Penjualan

### US-030 Transaksi retail dengan barcode
> Sebagai **Kasir toko**, saya ingin memindai barcode dan langsung menyelesaikan pembayaran, agar antrean cepat.

**Kriteria Penerimaan**
- Ketika saya memindai barcode yang terdaftar, maka item masuk keranjang dalam waktu di bawah 100 ms tanpa klik tambahan.
- Ketika barcode yang sama dipindai lagi, maka kuantitas item bertambah, bukan membuat baris baru.
- Ketika barcode tidak ditemukan, maka muncul pesan singkat dan pilihan mencari manual, tanpa mengosongkan keranjang.
- Transaksi retail sederhana selesai dalam maksimal 5 interaksi layar.

FR-SALE-01, G-03 | Must | 5 | Fase 1

### US-031 Mencari produk tanpa barcode
> Sebagai **Kasir**, saya ingin mencari produk dengan mengetik sebagian nama, agar saya tetap cepat meski barang tidak berbarcode.

**Kriteria Penerimaan**
- Ketika saya mengetik minimal 2 karakter, maka hasil muncul dalam waktu di bawah 200 ms.
- Pencarian mencakup nama, SKU, dan barcode sekaligus.
- Pencarian toleran terhadap salah ketik ringan.
- Pencarian tetap berfungsi saat offline memakai data lokal.

FR-SALE-01, NFR-PERF-05 | Must | 5 | Fase 1

### US-032 Diskon per item dan per transaksi
> Sebagai **Kasir**, saya ingin memberi diskon dalam nominal atau persentase, agar saya bisa melayani permintaan pelanggan sesuai kebijakan.

**Kriteria Penerimaan**
- Diskon dapat diterapkan pada satu baris item atau pada seluruh transaksi.
- Diskon persentase dan nominal keduanya didukung, dan sistem menampilkan hasil perhitungannya secara langsung.
- Diskon transaksi dialokasikan proporsional ke item untuk keperluan laporan laba per produk.
- Diskon tidak boleh membuat total menjadi negatif.
- Diskon melampaui batas peran memicu otorisasi supervisor.

FR-SALE-04, FR-SALE-05 | Must | 5 | Fase 1

### US-033 Menahan transaksi
> Sebagai **Kasir**, saya ingin menyimpan keranjang sementara, agar saya bisa melayani pelanggan berikutnya saat pelanggan pertama masih memilih barang.

**Kriteria Penerimaan**
- Ketika saya menahan transaksi, maka saya dapat memberi label untuk mengenalinya.
- Daftar transaksi tertahan menampilkan label, jumlah item, perkiraan total, dan waktu ditahan.
- Ketika saya memanggil kembali, maka keranjang kembali persis seperti semula termasuk diskon dan modifier.
- Transaksi tertahan yang belum diselesaikan sampai tutup shift ditampilkan sebagai pengingat sebelum shift ditutup.

FR-SALE-07 | Must | 5 | Fase 1

### US-034 Membatalkan transaksi yang sudah dibayar
> Sebagai **Supervisor**, saya ingin membatalkan transaksi yang salah, agar laporan dan stok kembali benar.

**Kriteria Penerimaan**
- Void memerlukan otorisasi dan alasan wajib.
- Ketika transaksi dibatalkan, maka stok dikembalikan lewat mutasi pembalik, bukan dengan menghapus mutasi asli.
- Transaksi yang dibatalkan tetap tersimpan dengan status `voided` dan tidak dihitung dalam omzet.
- Kas shift menyesuaikan otomatis.
- Void muncul di laporan pengendalian beserta pelaku dan pemberi otorisasi.
- Transaksi dari shift yang sudah ditutup tidak dapat divoid, hanya dapat diretur.

FR-SALE-08, FR-RPT-07 | Must | 8 | Fase 2

### US-035 Retur barang
> Sebagai **Kasir**, saya ingin memproses pengembalian barang, agar pelanggan mendapat uang atau tukar barang sesuai kebijakan toko.

**Kriteria Penerimaan**
- Ketika saya mencari transaksi asal, maka saya dapat memilih item dan kuantitas yang diretur, tidak harus seluruhnya.
- Sistem mencegah retur melebihi kuantitas yang dibeli, termasuk memperhitungkan retur sebelumnya atas transaksi yang sama.
- Saya dapat memilih pengembalian tunai, kredit toko, atau tukar barang.
- Saya dapat memilih apakah barang kembali ke stok atau tidak, sesuai kondisi barang.
- Poin loyalitas yang diperoleh dari transaksi asal dikoreksi proporsional.

FR-SALE-09 | Must | 8 | Fase 2

### US-036 Nomor struk yang selalu unik
> Sebagai **Pemilik Bisnis**, saya ingin nomor struk tidak pernah bentrok, agar pembukuan saya rapi.

**Kriteria Penerimaan**
- Nomor mengikuti format prefix terminal, tanggal, dan urutan.
- Ketika terminal offline, maka nomor tetap dihasilkan dan tidak bentrok dengan terminal lain karena prefix berbeda.
- Ketika transaksi tersinkron, maka server memverifikasi keunikan kombinasi terminal dan nomor.
- Nomor tidak pernah dipakai ulang meski transaksi dibatalkan.

FR-SALE-10, FR-OFF-08 | Must | 5 | Fase 1

### US-037 Cetak struk
> Sebagai **Kasir**, saya ingin struk tercetak otomatis setelah pembayaran, agar pelanggan langsung menerima bukti.

**Kriteria Penerimaan**
- Ketika pembayaran selesai, maka struk tercetak tanpa dialog cetak sistem.
- Ketika printer gagal, maka struk masuk antrean cetak dan saya melihat pemberitahuan disertai opsi coba lagi atau tampilkan QR struk digital.
- Cetak ulang menghasilkan struk bertanda "COPY" dan tercatat di audit log.
- Struk memuat identitas outlet, nomor, waktu, kasir, rincian item, modifier, diskon, pajak, pembulatan, total, dan pembayaran.
- Laci kas terbuka otomatis untuk pembayaran tunai.

FR-SALE-16, FR-HW-01, FR-HW-04, FR-HW-07 | Must | 8 | Fase 1

### US-038 Struk digital
> Sebagai **Pelanggan**, saya ingin menerima struk lewat tautan, agar saya tidak perlu menyimpan kertas.

**Kriteria Penerimaan**
- Ketika kasir memilih struk digital, maka tautan dibuat dan dapat dibagikan lewat WhatsApp atau ditampilkan sebagai QR.
- Halaman struk dapat dibuka tanpa login dan memakai token acak yang tidak dapat ditebak.
- Halaman tidak menampilkan data internal seperti HPP atau margin.
- Tautan kedaluwarsa setelah periode yang dikonfigurasi.

FR-SALE-17, FR-HW-09 | Should | 5 | Fase 2

### US-039 Item terbuka
> Sebagai **Kasir**, saya ingin menjual barang yang belum terdaftar, agar penjualan tidak terhambat.

**Kriteria Penerimaan**
- Ketika saya menambah item terbuka, maka saya mengisi nama, harga, dan kuantitas.
- Item terbuka tidak mengurangi stok dan tidak memerlukan master produk.
- Laporan mengelompokkan item terbuka terpisah agar mudah ditinjau.
- Izin membuat item terbuka dapat dibatasi per peran.

FR-SALE-14 | Should | 3 | Fase 1

### US-040 Mencari riwayat transaksi
> Sebagai **Supervisor**, saya ingin mencari transaksi lama, agar saya bisa menangani komplain pelanggan.

**Kriteria Penerimaan**
- Pencarian tersedia berdasarkan nomor struk, tanggal, nama pelanggan, kasir, dan nominal.
- Hasil menampilkan detail lengkap termasuk item, pembayaran, dan siapa yang memproses.
- Dari hasil pencarian saya dapat langsung mencetak ulang atau memproses retur.
- Kasir hanya dapat melihat transaksi shiftnya sendiri kecuali diberi izin lebih luas.

FR-SALE-18, FR-RPT-13 | Must | 5 | Fase 1

### US-041 Menetapkan pramuniaga
> Sebagai **Pemilik Toko**, saya ingin mencatat siapa yang melayani penjualan, agar komisi dihitung tepat meski kasirnya orang lain.

**Kriteria Penerimaan**
- Kasir dapat memilih pramuniaga saat transaksi, dan pilihan ini terpisah dari identitas kasir.
- Pramuniaga dapat ditetapkan per baris item untuk kasus beberapa pelayan dalam satu nota.
- Laporan penjualan per pramuniaga tersedia terpisah dari laporan per kasir.

FR-SALE-12, FR-EMP-04 | Should | 5 | Fase 3

---

## EP-06 Pembayaran

### US-042 Pembayaran tunai dengan saran pecahan
> Sebagai **Kasir**, saya ingin sistem menyarankan nominal uang, agar saya tidak salah hitung kembalian.

**Kriteria Penerimaan**
- Sistem menampilkan tombol nominal umum (uang pas, 50.000, 100.000) berdasarkan total.
- Kembalian dihitung otomatis dan ditampilkan besar agar mudah dibaca.
- Ketika nominal kurang dari total, maka tombol selesai nonaktif dengan penjelasan.

FR-PAY-03 | Must | 3 | Fase 1

### US-043 Pembayaran terpisah
> Sebagai **Kasir**, saya ingin menerima pembayaran dengan beberapa metode sekaligus, agar pelanggan bisa membayar sebagian tunai dan sebagian kartu.

**Kriteria Penerimaan**
- Saya dapat menambahkan beberapa baris pembayaran dengan metode berbeda.
- Sisa yang harus dibayar diperbarui setiap kali baris ditambahkan.
- Transaksi tidak dapat diselesaikan sebelum total pembayaran mencukupi, kecuali sisanya dijadikan piutang.
- Struk menampilkan rincian tiap metode.

FR-PAY-02 | Must | 5 | Fase 2

### US-044 QRIS dinamis
> Sebagai **Kasir**, saya ingin QRIS muncul dengan nominal otomatis dan terkonfirmasi sendiri, agar saya tidak perlu mengecek mutasi manual.

**Kriteria Penerimaan**
- Ketika saya memilih QRIS, maka QR dengan nominal transaksi tampil dalam waktu di bawah 3 detik.
- Ketika pembayaran diterima gateway, maka status transaksi berubah otomatis tanpa saya menekan apa pun.
- Ketika QR kedaluwarsa, maka saya dapat membuat ulang tanpa mengulang transaksi.
- Ketika terminal offline, maka metode ini nonaktif dengan penjelasan dan alternatif QRIS statis tetap tersedia sebagai pembayaran manual.
- Biaya MDR tercatat untuk laporan.

FR-PAY-06, FR-PAY-05, FR-OFF-06 | Should | 8 | Fase 5

### US-045 Uang muka dan pelunasan
> Sebagai **Kasir bengkel**, saya ingin menerima DP dan pelunasan terpisah, agar arus kas tercatat benar.

**Kriteria Penerimaan**
- Ketika order dibuat, maka saya dapat menerima pembayaran sebagian dan sisanya tercatat sebagai kewajiban pelanggan.
- Riwayat pembayaran per order menampilkan tanggal, nominal, metode, dan penerima.
- Order tidak dapat diselesaikan sebelum lunas, kecuali pemilik mengizinkan piutang.
- Setiap pembayaran masuk ke shift saat pembayaran diterima, bukan saat order dibuat.

FR-PAY-07, FR-SRV-03 | Should | 5 | Fase 2

### US-046 Penjualan piutang dengan batas kredit
> Sebagai **Pemilik Toko Grosir**, saya ingin membatasi piutang per pelanggan, agar risiko gagal bayar terkendali.

**Kriteria Penerimaan**
- Ketika kasir memilih metode piutang, maka sistem memeriksa sisa batas kredit pelanggan.
- Ketika batas terlampaui, maka transaksi ditolak kecuali disetujui supervisor.
- Saldo piutang pelanggan diperbarui langsung setelah transaksi.
- Metode ini tidak tersedia saat offline karena membutuhkan saldo terkini.

FR-PAY-08, FR-CUST-06 | Should | 5 | Fase 3

### US-047 Mencatat biaya admin per metode
> Sebagai **Pemilik Bisnis**, saya ingin tahu berapa biaya yang saya bayar ke penyedia pembayaran, agar saya tahu margin bersih sebenarnya.

**Kriteria Penerimaan**
- Setiap metode pembayaran dapat memiliki biaya persentase atau tetap.
- Biaya tercatat pada tiap pembayaran dan tidak mengubah nominal yang dibayar pelanggan bila ditanggung merchant.
- Laporan metode pembayaran menampilkan kolom biaya dan nilai bersih.

FR-PAY-05 | Should | 3 | Fase 3

---

## EP-07 Shift dan Kas

### US-048 Membuka shift
> Sebagai **Kasir**, saya ingin mencatat modal awal saat mulai bekerja, agar perhitungan akhir shift jelas.

**Kriteria Penerimaan**
- Ketika saya membuka shift, maka saya mengisi modal awal laci kas.
- Saya tidak dapat bertransaksi sebelum shift dibuka, dan sistem mengarahkan saya ke pembukaan shift secara otomatis.
- Satu terminal hanya boleh punya satu shift terbuka, ditegakkan di tingkat basis data.
- Ketika ada shift milik kasir lain masih terbuka di terminal itu, maka saya melihat informasinya dan hanya manajer yang dapat menutup paksa.

FR-SHIFT-01, FR-SHIFT-02, FR-SHIFT-07 | Must | 5 | Fase 1

### US-049 Kas masuk dan keluar di tengah shift
> Sebagai **Kasir**, saya ingin mencatat uang yang keluar masuk laci di luar penjualan, agar hitungan akhir tetap cocok.

**Kriteria Penerimaan**
- Saya dapat mencatat kas masuk dan kas keluar dengan keterangan wajib.
- Kas keluar di atas nominal tertentu memerlukan otorisasi supervisor.
- Setiap pencatatan memengaruhi kas yang diharapkan saat tutup shift.
- Saya dapat melampirkan foto nota untuk pengeluaran.

FR-SHIFT-03, FR-EXP-04 | Must | 5 | Fase 1

### US-050 Menutup shift dengan hitungan fisik
> Sebagai **Kasir**, saya ingin proses tutup kas yang transparan, agar saya tidak disalahkan atas selisih yang bukan kesalahan saya.

**Kriteria Penerimaan**
- Ketika saya menutup shift, maka sistem meminta hitungan fisik per pecahan uang dan menjumlahkan otomatis.
- Setelah input selesai, maka sistem menampilkan kas seharusnya, kas aktual, dan selisih.
- Ketika selisih melebihi ambang yang dikonfigurasi, maka alasan wajib diisi dan otorisasi supervisor diperlukan.
- Setelah shift ditutup, maka laporan Z tercetak dan data shift tidak dapat diubah.
- Ringkasan menampilkan total per metode pembayaran, jumlah void, diskon, dan retur selama shift.

FR-SHIFT-04, FR-SHIFT-05, FR-SHIFT-06 | Must | 8 | Fase 1

### US-051 Melihat ringkasan tanpa menutup shift
> Sebagai **Kasir**, saya ingin melihat rekap sementara, agar saya bisa mengecek kas di tengah hari.

**Kriteria Penerimaan**
- Laporan X dapat dicetak kapan saja tanpa menutup shift.
- Laporan X tidak mengubah status shift dan dapat dicetak berkali kali.
- Laporan X menampilkan angka yang sama dengan laporan Z bila shift ditutup saat itu juga.

FR-SHIFT-06 | Should | 3 | Fase 1

### US-052 Menutup paksa shift yang terlupa
> Sebagai **Manajer**, saya ingin menutup shift yang lupa ditutup kasir, agar laporan harian tidak menggantung.

**Kriteria Penerimaan**
- Shift yang melewati batas durasi wajar ditandai di dasbor.
- Ketika saya menutup paksa, maka saya wajib mengisi catatan dan hitungan kas jika ada.
- Shift yang ditutup paksa ditandai berbeda di laporan dan menyimpan identitas penutupnya.
- Kasir yang bersangkutan menerima notifikasi.

FR-SHIFT-08 | Should | 3 | Fase 2

---

## EP-08 Persediaan

### US-053 Melihat stok terkini
> Sebagai **Staf Gudang**, saya ingin melihat stok per outlet, agar saya tahu apa yang perlu ditambah.

**Kriteria Penerimaan**
- Daftar stok dapat difilter per kategori, per outlet, dan status (menipis, habis, normal).
- Stok yang berada di bawah minimum ditandai jelas.
- Saya dapat melihat kartu stok satu produk berisi seluruh mutasi berurutan dengan saldo berjalan.
- Halaman ini tidak menampilkan informasi harga jual atau margin bila peran saya tidak memiliki izin tersebut.

FR-INV-01, FR-INV-02, FR-RPT-13 | Must | 5 | Fase 1

### US-054 Menyesuaikan stok dengan alasan
> Sebagai **Staf Gudang**, saya ingin mencatat barang rusak atau hilang, agar catatan stok sesuai kenyataan.

**Kriteria Penerimaan**
- Penyesuaian memerlukan pemilihan alasan dari daftar dan catatan opsional.
- Sistem menampilkan stok sistem dan meminta stok aktual, lalu menghitung selisih sendiri.
- Penyesuaian menghasilkan mutasi stok dengan tipe `adjustment` dan menyimpan dampak nilai HPP.
- Penyesuaian tercatat di audit log dan muncul di laporan penyesuaian.

FR-INV-03 | Must | 5 | Fase 1

### US-055 Stok opname
> Sebagai **Pemilik Bisnis**, saya ingin melakukan penghitungan fisik berkala, agar saya tahu selisih sebenarnya.

**Kriteria Penerimaan**
- Ketika opname dimulai, maka stok sistem dibekukan sebagai pembanding pada saat itu.
- Mode penghitungan buta menyembunyikan stok sistem dari penghitung.
- Beberapa orang dapat menghitung kategori berbeda dalam satu sesi opname.
- Sebelum diposting, saya melihat ringkasan selisih beserta nilai rupiahnya.
- Setelah diposting, maka penyesuaian stok dibuat otomatis dan opname tidak dapat diubah.
- Transaksi penjualan yang terjadi selama opname berlangsung tetap tercatat dan diperhitungkan.

FR-INV-04, FR-INV-05 | Must | 13 | Fase 3

### US-056 Transfer stok antar outlet
> Sebagai **Manajer**, saya ingin memindahkan barang antar cabang, agar stok merata.

**Kriteria Penerimaan**
- Outlet pengirim membuat dokumen transfer, dan stok berkurang saat dikirim.
- Outlet penerima mengonfirmasi penerimaan, dan stok bertambah saat diterima.
- Selisih antara dikirim dan diterima tampil sebagai barang dalam perjalanan.
- Penerimaan sebagian didukung dengan sisa tetap tercatat.
- Kedua outlet melihat riwayat transfer dan status terkini.

FR-INV-06 | Should | 8 | Fase 3

### US-057 Peringatan stok menipis
> Sebagai **Pemilik Bisnis**, saya ingin diberi tahu saat stok menipis, agar saya tidak kehabisan barang laris.

**Kriteria Penerimaan**
- Batas minimum dapat diatur per produk dan dioverride per outlet.
- Ketika stok menyentuh batas, maka notifikasi terkirim maksimal sekali per hari per produk agar tidak mengganggu.
- Dasbor menampilkan daftar produk yang perlu dibeli.
- Notifikasi tidak dikirim untuk produk yang sudah diarsipkan.

FR-INV-07 | Must | 5 | Fase 2

### US-058 Kontrol penjualan saat stok habis
> Sebagai **Pemilik Bisnis**, saya ingin menentukan apakah kasir boleh menjual barang yang stoknya nol, agar sesuai cara kerja usaha saya.

**Kriteria Penerimaan**
- Ketika stok minus dilarang, maka kasir tidak dapat menambahkan item yang stoknya nol dan melihat pesan yang jelas.
- Ketika stok minus diizinkan, maka penjualan tetap berjalan dan produk ditandai untuk ditinjau.
- Pengaturan berlaku per bisnis dan tidak memengaruhi transaksi yang sudah terjadi.
- Transaksi offline tidak pernah ditolak karena stok, apa pun pengaturannya, dan hanya ditandai saat sinkronisasi.

FR-INV-10, FR-OFF-05 | Must | 5 | Fase 2

### US-059 Melacak kedaluwarsa
> Sebagai **Pemilik Apotek**, saya ingin tahu barang mana yang mendekati kedaluwarsa, agar saya bisa menghabiskannya lebih dulu.

**Kriteria Penerimaan**
- Penerimaan barang mencatat nomor batch dan tanggal kedaluwarsa.
- Penjualan mengurangi batch dengan kedaluwarsa terdekat lebih dulu.
- Laporan menampilkan barang yang akan kedaluwarsa dalam 30, 60, dan 90 hari.
- Barang yang sudah kedaluwarsa ditandai dan dapat dikeluarkan lewat penyesuaian dengan alasan `expired`.

FR-PRD-14, FR-INV-12 | Should | 8 | Fase 4

### US-060 Menjaga konsistensi stok
> Sebagai **Pemilik Bisnis**, saya ingin yakin angka stok tidak pernah rusak diam diam, agar saya bisa memercayai laporan.

**Kriteria Penerimaan**
- Job harian membandingkan saldo stok tercatat dengan hasil penjumlahan seluruh mutasi.
- Ketika ditemukan selisih, maka sistem mencatat dan memberi tahu pemilik, bukan memperbaiki diam diam.
- Halaman diagnostik menampilkan produk yang bermasalah beserta mutasi terkait.

FR-INV-13, AP-03 | Must | 5 | Fase 4

---

## EP-09 Modul F&B

### US-061 Membuka bill per meja
> Sebagai **Pelayan**, saya ingin membuka tagihan per meja, agar pesanan bisa ditambah sepanjang tamu duduk.

**Kriteria Penerimaan**
- Ketika saya memilih meja kosong dan mengisi jumlah tamu, maka bill terbuka dan status meja berubah menjadi terisi.
- Beberapa terminal dapat menambah item ke bill yang sama tanpa saling menimpa.
- Ketika bill dibayar, maka status meja berubah menjadi perlu dibersihkan lalu kosong.
- Denah meja memperlihatkan status dan durasi tamu duduk secara realtime.

FR-FNB-01, FR-FNB-02 | Must | 8 | Fase 2

### US-062 Mengirim pesanan ke dapur
> Sebagai **Pelayan**, saya ingin mengirim pesanan ke dapur tanpa menutup bill, agar makanan mulai dimasak lebih awal.

**Kriteria Penerimaan**
- Ketika saya menekan kirim, maka hanya item yang belum dikirim yang diteruskan ke dapur.
- Tiket dapur tercetak di printer sesuai kategori item, sehingga minuman ke bar dan makanan ke dapur.
- Item yang sudah dikirim ditandai di keranjang dan tidak dapat dihapus tanpa otorisasi.
- Catatan item ikut tercetak di tiket.

FR-FNB-05, FR-FNB-07, FR-HW-03 | Must | 8 | Fase 2

### US-063 Layar dapur
> Sebagai **Staf Dapur**, saya ingin melihat pesanan di layar, agar saya tidak bergantung pada kertas.

**Kriteria Penerimaan**
- Pesanan baru muncul di layar dalam waktu di bawah 2 detik setelah dikirim.
- Pesanan diurutkan berdasarkan waktu masuk dan menampilkan durasi tunggu berjalan.
- Pesanan yang melewati ambang waktu berubah warna sebagai peringatan.
- Saya dapat mengubah status menjadi diproses lalu siap, dan pelayan melihat perubahan itu.
- Layar tetap menampilkan pesanan terakhir bila koneksi terputus, dan menyegerakan sinkronisasi saat pulih.

FR-FNB-06, FR-FNB-08 | Should | 8 | Fase 4

### US-064 Pindah dan gabung meja
> Sebagai **Pelayan**, saya ingin memindahkan atau menggabungkan tamu, agar melayani perubahan di lapangan.

**Kriteria Penerimaan**
- Pindah meja memindahkan seluruh bill beserta riwayat pesanan.
- Gabung meja menyatukan dua bill menjadi satu dengan jejak asal masing masing.
- Kedua aksi tercatat di audit log.
- Meja asal kembali berstatus kosong setelah dipindahkan.

FR-FNB-03 | Should | 5 | Fase 2

### US-065 Split bill
> Sebagai **Kasir**, saya ingin membagi tagihan, agar tamu bisa membayar sendiri sendiri.

**Kriteria Penerimaan**
- Saya dapat membagi berdasarkan item yang dipilih atau membagi rata sejumlah orang.
- Setiap bagian menghasilkan struk terpisah dengan nomor sendiri.
- Pajak dan service charge dialokasikan proporsional pada setiap bagian.
- Pembagian dapat dibatalkan selama belum ada bagian yang dibayar.

FR-FNB-04 | Should | 8 | Fase 3

### US-066 Menandai menu habis
> Sebagai **Staf Dapur**, saya ingin menandai menu yang habis, agar pelayan tidak menjualnya.

**Kriteria Penerimaan**
- Ketika saya menandai item habis, maka item nonaktif di seluruh terminal dalam waktu di bawah 5 detik.
- Item habis tetap terlihat dengan tanda jelas, tidak hilang begitu saja, agar pelayan dapat menjelaskan ke tamu.
- Penandaan otomatis hilang saat pergantian hari, dan dapat dikembalikan manual kapan saja.

FR-FNB-11 | Should | 3 | Fase 2

---

## EP-10 Modul Jasa

### US-067 Membuat order servis
> Sebagai **Kasir laundry**, saya ingin mencatat order dengan estimasi selesai, agar pelanggan tahu kapan bisa mengambil.

**Kriteria Penerimaan**
- Order mencatat pelanggan, layanan, kuantitas terukur seperti kilogram, dan catatan.
- Estimasi selesai dihitung otomatis dari durasi layanan dan dapat disesuaikan manual.
- Nota order dan label penanda barang dapat dicetak, keduanya memuat QR pelacakan.
- Order dapat menerima DP saat dibuat.

FR-SRV-01, FR-SRV-02, FR-SRV-04, FR-SRV-09 | Must | 8 | Fase 2

### US-068 Alur status yang dapat dikonfigurasi
> Sebagai **Pemilik Bisnis jasa**, saya ingin menentukan tahapan pengerjaan sendiri, agar sesuai proses usaha saya.

**Kriteria Penerimaan**
- Saya dapat mendefinisikan urutan status beserta label dan warnanya.
- Saya dapat menandai status mana yang memicu notifikasi ke pelanggan dan status mana yang bersifat final.
- Perubahan status tercatat lengkap dengan pelaku dan waktu.
- Status yang sudah dipakai order aktif tidak dapat dihapus.

FR-SRV-01 | Must | 5 | Fase 2

### US-069 Halaman lacak untuk pelanggan
> Sebagai **Pelanggan**, saya ingin mengecek status pesanan tanpa menelepon, agar saya datang saat sudah siap.

**Kriteria Penerimaan**
- Tautan pelacakan dapat dibuka tanpa login memakai token acak.
- Halaman menampilkan status terkini, riwayat tahapan, dan estimasi selesai.
- Halaman tidak menampilkan data internal seperti nama teknisi, HPP, atau margin.
- Halaman tetap dapat diakses selama 30 hari setelah order selesai.

FR-SRV-05 | Should | 5 | Fase 2

### US-070 Notifikasi saat pesanan siap
> Sebagai **Pemilik laundry**, saya ingin pelanggan diberi tahu otomatis, agar barang cepat diambil.

**Kriteria Penerimaan**
- Ketika status berubah ke status yang ditandai memicu notifikasi, maka pesan terkirim otomatis ke nomor pelanggan.
- Template pesan dapat disunting dan mendukung variabel nama, nomor order, dan tautan pelacakan.
- Kegagalan pengiriman tercatat dan dapat dikirim ulang manual.
- Pelanggan tanpa nomor telepon dilewati tanpa menghentikan perubahan status.

FR-SRV-06, FR-NOTIF-03 | Should | 5 | Fase 2

### US-071 Penugasan pekerja
> Sebagai **Manajer bengkel**, saya ingin menugaskan teknisi pada order, agar tanggung jawab dan komisi jelas.

**Kriteria Penerimaan**
- Order dapat ditugaskan ke satu teknisi, dan tiap item layanan dapat ditugaskan berbeda.
- Teknisi melihat daftar tugasnya sendiri.
- Komisi terhitung berdasarkan aturan yang berlaku saat order diselesaikan.
- Laporan performa per teknisi tersedia.

FR-SRV-07, FR-EMP-04 | Should | 5 | Fase 3

---

## EP-11 Pelanggan, Promo, dan Loyalitas

### US-072 Mendaftarkan pelanggan dari layar kasir
> Sebagai **Kasir**, saya ingin mendaftarkan pelanggan tanpa keluar dari transaksi, agar antrean tidak terganggu.

**Kriteria Penerimaan**
- Pencarian berdasarkan nomor telepon menampilkan hasil dalam waktu di bawah 200 ms.
- Ketika tidak ditemukan, maka tombol daftar baru muncul dengan nomor telepon yang sudah terisi.
- Form pendaftaran cepat hanya meminta nama dan telepon.
- Setelah tersimpan, pelanggan langsung terpasang pada transaksi berjalan.
- Pendaftaran tetap dapat dilakukan saat offline dan direkonsiliasi saat sinkron.

FR-CUST-02, FR-CUST-03 | Must | 5 | Fase 2

### US-073 Promo otomatis terjadwal
> Sebagai **Pemilik Bisnis**, saya ingin promo berjalan otomatis pada jam tertentu, agar saya tidak mengandalkan ingatan kasir.

**Kriteria Penerimaan**
- Promo dapat dibatasi berdasarkan tanggal, hari dalam seminggu, dan rentang jam.
- Promo yang memenuhi syarat diterapkan otomatis di keranjang dan diberi label sumbernya.
- Ketika beberapa promo cocok, maka aturan penumpukan dan prioritas menentukan hasilnya secara konsisten.
- Kasir dapat melihat promo apa yang diterapkan dan berapa nilainya.
- Promo tetap berjalan saat offline karena aturannya tersimpan lokal.

FR-PROMO-01, FR-PROMO-03, FR-PROMO-04 | Should | 13 | Fase 3

### US-074 Program poin
> Sebagai **Pemilik Bisnis**, saya ingin memberi poin pada pelanggan, agar mereka kembali berbelanja.

**Kriteria Penerimaan**
- Rasio perolehan dan penukaran poin dapat dikonfigurasi.
- Poin bertambah otomatis setelah transaksi selesai dan tampil di struk.
- Penukaran poin memerlukan pelanggan terpilih dan tidak boleh melebihi saldo.
- Poin dari transaksi yang diretur dikoreksi proporsional.
- Saat offline, poin tetap diperoleh dan dihitung saat sinkron, tetapi penukaran tidak tersedia.

FR-PROMO-07 | Should | 8 | Fase 3

### US-075 Kupon dengan kode
> Sebagai **Pemilik Bisnis**, saya ingin membagikan kode promo, agar saya bisa mengukur efektivitas kampanye.

**Kriteria Penerimaan**
- Kupon dapat dibuat massal dengan kode unik atau kode tunggal yang dipakai banyak orang.
- Batas pemakaian total dan per pelanggan ditegakkan.
- Kupon yang melewati kuota karena sinkronisasi offline tetap diterima namun ditandai untuk ditinjau.
- Laporan menampilkan jumlah pemakaian, nilai diskon, dan omzet yang dihasilkan.

FR-PROMO-05, FR-PROMO-09 | Should | 8 | Fase 3

### US-076 Riwayat dan segmentasi pelanggan
> Sebagai **Pemilik Bisnis**, saya ingin melihat kebiasaan belanja pelanggan, agar saya bisa menawarkan yang tepat.

**Kriteria Penerimaan**
- Profil pelanggan menampilkan total belanja, frekuensi, rata rata nilai transaksi, dan produk favorit.
- Daftar pelanggan dapat diurutkan berdasarkan nilai belanja dan difilter berdasarkan periode tidak aktif.
- Data dapat diekspor untuk keperluan kampanye.

FR-CUST-04 | Should | 5 | Fase 3

### US-077 Menagih piutang jatuh tempo
> Sebagai **Pemilik Toko Grosir**, saya ingin melihat piutang yang jatuh tempo, agar arus kas saya terjaga.

**Kriteria Penerimaan**
- Laporan umur piutang mengelompokkan tagihan pada rentang 0 sampai 30, 31 sampai 60, 61 sampai 90, dan di atas 90 hari.
- Saya dapat mencatat pembayaran sebagian dan sisanya tetap tercatat.
- Pembayaran piutang secara tunai masuk ke shift yang sedang berjalan.
- Pengingat otomatis dapat diaktifkan menjelang jatuh tempo.

FR-CUST-07, FR-CUST-08, FR-CUST-09 | Should | 8 | Fase 3

---

## EP-12 Pembelian

### US-078 Menerima barang dari supplier
> Sebagai **Staf Gudang**, saya ingin mencatat barang masuk, agar stok dan harga beli terbarui.

**Kriteria Penerimaan**
- Penerimaan dapat dilakukan dengan atau tanpa PO.
- Stok bertambah dan HPP rata rata diperbarui saat penerimaan diposting.
- Ketika harga beli berbeda dari sebelumnya, maka sistem menampilkan perbandingan sebagai konfirmasi.
- Penerimaan menghasilkan hutang usaha bila pembayaran belum dilakukan.
- Nomor batch dan kedaluwarsa dapat diisi bila modul terkait aktif.

FR-PUR-03, FR-INV-08, FR-PUR-07 | Should | 8 | Fase 3

---

## Ringkasan Estimasi per Fase

| Fase | Story | Total Point | Fokus |
|---|---|---|---|
| Fase 1 | 28 | 142 | MVP: berjualan sungguhan dengan satu tenant |
| Fase 2 | 20 | 118 | Ekspansi vertical: F&B dan jasa |
| Fase 3 | 16 | 108 | Operasional lanjut: pembelian, promo, piutang |
| Fase 4 | 8 | 52 | Ketahanan: offline, KDS, multi satuan |
| Fase 5 | 6 | 34 | Monetisasi dan integrasi |

Catatan: story EP-13 sampai EP-15 (laporan lanjutan, sinkronisasi offline detail, dan platform langganan) belum dirinci pada versi dokumen ini dan akan ditambahkan sebelum masuk Fase 4. Kerangka kriteria penerimaannya sudah tersedia pada bagian 12 PRD dan bagian 10 dokumen arsitektur.
