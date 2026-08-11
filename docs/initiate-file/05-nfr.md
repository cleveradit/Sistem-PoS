# Non-Functional Requirements (NFR)
## Aplikasi Point of Sale (POS) Multi-Tenant

| Item | Keterangan |
|---|---|
| Dokumen Induk | PRD POS Multi-Tenant v1.0, Architecture v1.0 |
| Versi | 1.0 |
| Tanggal | 10 Agustus 2026 |

---

## 1. Cara Membaca Dokumen Ini

Setiap NFR memiliki lima komponen:

| Komponen | Arti |
|---|---|
| **Kode** | Pengenal unik untuk dirujuk di story dan uji |
| **Kebutuhan** | Pernyataan yang dapat diverifikasi, bukan aspirasi |
| **Metrik** | Angka yang menjadi ambang lulus atau gagal |
| **Cara Verifikasi** | Bagaimana diuji dan pada tahap apa |
| **Prioritas** | Must, Should, Could |

Kebutuhan tanpa angka bukan NFR, melainkan harapan. Dokumen ini menghindari kata "cepat", "aman", dan "mudah" tanpa disertai ukuran.

### 1.1 Perangkat Referensi

Seluruh target performa diukur pada tiga kelas perangkat berikut, bukan pada mesin pengembang:

| Kelas | Spesifikasi Acuan | Bobot Pengujian |
|---|---|---|
| **Rendah** | Android 9, RAM 3 GB, Snapdragon 450 setara, jaringan 3G | 40 persen pengguna |
| **Menengah** | Android 12, RAM 6 GB, laptop i3 gen 10, jaringan 4G | 45 persen pengguna |
| **Tinggi** | Tablet modern atau desktop, jaringan fiber | 15 persen pengguna |

Target default merujuk pada kelas **Menengah**. Kelas Rendah memiliki toleransi 2 kali lipat kecuali disebut lain.

---

## 2. Performa

### 2.1 Responsivitas Layar Kasir

| Kode | Kebutuhan | Metrik | Verifikasi | Prioritas |
|---|---|---|---|---|
| NFR-PERF-01 | Menambah item ke keranjang terasa seketika | p95 di bawah 100 ms dari pemindaian sampai item tampil | Uji otomatis Playwright dengan katalog 10.000 item | Must |
| NFR-PERF-02 | Pencarian produk responsif saat mengetik | p95 di bawah 200 ms untuk kueri 2 karakter, katalog 50.000 item | Benchmark lokal IndexedDB dan uji query server | Must |
| NFR-PERF-03 | Layar kasir siap dipakai dengan cepat | Time to Interactive di bawah 3 detik pada 3G, di bawah 1,5 detik saat aset ter-cache | Lighthouse CI pada pipeline | Must |
| NFR-PERF-04 | Penyimpanan transaksi tidak menahan kasir | Di bawah 500 ms saat online (p95), di bawah 50 ms saat offline | Uji beban k6 dan uji lokal | Must |
| NFR-PERF-05 | Perhitungan ulang total setelah perubahan keranjang tidak terasa jeda | Di bawah 50 ms untuk keranjang 50 baris dengan promo aktif | Unit benchmark PricingEngine | Must |
| NFR-PERF-06 | Perpindahan antar layar utama kasir mulus | Di bawah 300 ms | Uji manual terstruktur dan Playwright | Should |
| NFR-PERF-07 | Cetak struk dimulai segera setelah pembayaran | Perintah terkirim ke Print Agent di bawah 200 ms | Uji integrasi Print Agent | Must |

**Catatan desain:** Target NFR-PERF-01 hanya dapat dicapai bila pencarian barcode dilakukan pada indeks lokal, bukan lewat jaringan. Ini menjadi alasan teknis wajibnya penyimpanan katalog lokal, terlepas dari kebutuhan offline.

### 2.2 Backend dan Laporan

| Kode | Kebutuhan | Metrik | Verifikasi | Prioritas |
|---|---|---|---|---|
| NFR-PERF-08 | Endpoint API transaksional responsif | p95 di bawah 300 ms, p99 di bawah 800 ms | Uji beban k6 pada staging | Must |
| NFR-PERF-09 | Dasbor ringkas termuat cepat | Di bawah 2 detik untuk data hari berjalan | Uji dengan dataset 1 juta transaksi | Must |
| NFR-PERF-10 | Laporan periode satu bulan termuat wajar | Di bawah 5 detik | Uji dengan dataset representatif | Must |
| NFR-PERF-11 | Laporan periode di atas 3 bulan tidak memblokir | Diproses asinkron, hasil dikirim lewat notifikasi dalam 3 menit | Uji job Horizon | Should |
| NFR-PERF-12 | Ekspor besar tidak membebani permintaan web | Di atas 5.000 baris diproses lewat antrean | Uji fungsional | Should |
| NFR-PERF-13 | Sinkronisasi batch efisien | 100 transaksi tersinkron di bawah 10 detik pada 4G | Uji integrasi sinkronisasi | Must |
| NFR-PERF-14 | Import produk massal selesai wajar | 1.000 produk di bawah 60 detik | Uji job import | Must |

### 2.3 Kapasitas

| Kode | Kebutuhan | Metrik | Verifikasi | Prioritas |
|---|---|---|---|---|
| NFR-PERF-15 | Katalog besar tetap dapat dioperasikan | 50.000 produk per bisnis tanpa degradasi pencarian melewati NFR-PERF-02 | Uji dataset sintetis | Must |
| NFR-PERF-16 | Jam sibuk outlet tertangani | 50 transaksi per menit per outlet | Uji beban skenario jam makan siang | Must |
| NFR-PERF-17 | Beban platform tertangani | 500 transaksi per detik agregat lintas tenant pada puncak | Uji beban bertahap | Should |
| NFR-PERF-18 | Ukuran unduhan awal PWA terkendali | Bundel awal di bawah 500 KB terkompresi | Analisis bundel di CI, gagal bila melewati ambang | Should |
| NFR-PERF-19 | Penyimpanan lokal terminal tidak membengkak | Di bawah 100 MB untuk katalog 10.000 item dan riwayat 7 hari | Uji ukuran IndexedDB | Should |

---

## 3. Skalabilitas

| Kode | Kebutuhan | Metrik | Verifikasi | Prioritas |
|---|---|---|---|---|
| NFR-SCALE-01 | Aplikasi dapat diskalakan horizontal | Penambahan instance tidak memerlukan perubahan kode. Tidak ada state di memori instance | Review arsitektur, uji dengan 2 instance di belakang proxy | Must |
| NFR-SCALE-02 | Pekerjaan berat terpisah dari permintaan web | Laporan, import, ekspor, agregasi berjalan di worker terpisah | Review kode dan konfigurasi Horizon | Must |
| NFR-SCALE-03 | Antrean tidak saling memblokir | Antrean terpisah: `critical` (sinkronisasi, cetak), `default`, `reports`, `notifications` | Konfigurasi Horizon dan uji beban | Must |
| NFR-SCALE-04 | Basis data siap tumbuh | Partisi terpasang pada `audit_logs` sejak awal, siap diaktifkan pada `sales`, `sale_items`, `stock_movements` | Uji migrasi partisi pada dataset besar | Should |
| NFR-SCALE-05 | Isolasi tenant besar dimungkinkan tanpa refactor | `TenantConnectionResolver` sudah menjadi jalur tunggal akses koneksi sejak v1 | Review arsitektur | Should |
| NFR-SCALE-06 | Pertumbuhan tenant tidak menurunkan performa tenant lain | Degradasi p95 di bawah 20 persen saat jumlah tenant naik 10 kali | Uji beban multi tenant | Should |
| NFR-SCALE-07 | Aset statis dilayani terpisah dari aplikasi | Gambar produk dan aset lewat object storage atau CDN | Review deployment | Should |

---

## 4. Ketersediaan dan Pemulihan

| Kode | Kebutuhan | Metrik | Verifikasi | Prioritas |
|---|---|---|---|---|
| NFR-AVAIL-01 | Layanan inti tersedia | Uptime 99,5 persen per bulan, setara maksimal 3,6 jam gangguan | Monitor eksternal, laporan bulanan | Must |
| NFR-AVAIL-02 | Penjualan tidak berhenti saat server tidak dapat dijangkau | Terminal tetap dapat bertransaksi penuh secara offline | Uji chaos: matikan jaringan saat transaksi berjalan | Must |
| NFR-AVAIL-03 | Deployment tidak menghentikan layanan | Zero downtime deploy, migrasi memakai pola expand-contract | Uji deploy di staging sambil menjalankan beban | Must |
| NFR-AVAIL-04 | Backup terjadwal dan teruji | Backup harian penuh, WAL archiving berkelanjutan, retensi 30 hari | Cron terpantau, uji restore bulanan | Must |
| NFR-AVAIL-05 | Kehilangan data maksimum terbatas | RPO 15 menit untuk data server, 0 untuk transaksi yang sudah tersimpan lokal | Uji pemulihan | Must |
| NFR-AVAIL-06 | Waktu pemulihan terbatas | RTO 4 jam | Simulasi pemulihan bencana dua kali setahun | Must |
| NFR-AVAIL-07 | Kegagalan layanan eksternal tidak menjatuhkan aplikasi | Payment gateway, WhatsApp, dan storage diakses dengan timeout dan circuit breaker | Uji dengan layanan disimulasikan gagal | Must |
| NFR-AVAIL-08 | Kegagalan cetak tidak menggagalkan transaksi | Transaksi tetap tersimpan, cetak masuk antrean dengan retry | Uji integrasi | Must |
| NFR-AVAIL-09 | Halaman status tersedia saat gangguan | Halaman status terpisah dari infrastruktur utama | Review deployment | Could |

---

## 5. Keamanan

### 5.1 Autentikasi dan Sesi

| Kode | Kebutuhan | Metrik | Verifikasi | Prioritas |
|---|---|---|---|---|
| NFR-SEC-01 | Seluruh komunikasi terenkripsi | TLS 1.2 ke atas, HSTS aktif, tanpa endpoint HTTP | Uji SSL Labs minimal peringkat A | Must |
| NFR-SEC-02 | Kata sandi tersimpan aman | Argon2id, minimal 8 karakter, dicek terhadap daftar kata sandi bocor | Review kode, uji otomatis | Must |
| NFR-SEC-03 | PIN kasir tersimpan aman | Hash bcrypt dengan pepper, tidak pernah dikembalikan API, maksimal 5 percobaan lalu terkunci 15 menit | Uji otomatis, review respons API | Must |
| NFR-SEC-04 | Percobaan masuk dibatasi | 10 percobaan per menit per IP, 5 per akun | Uji rate limit | Must |
| NFR-SEC-05 | Sesi kasir terkunci saat ditinggalkan | Kunci otomatis setelah idle, default 10 menit, dapat dikonfigurasi | Uji fungsional | Should |
| NFR-SEC-06 | Token perangkat dapat dicabut | Pencabutan berlaku pada permintaan berikutnya, maksimal 60 detik | Uji fungsional | Must |
| NFR-SEC-07 | Token otorisasi supervisor tidak dapat dipakai ulang | TTL 90 detik, terikat aksi dan konteks, sekali pakai | Uji keamanan khusus | Must |

### 5.2 Otorisasi dan Isolasi

| Kode | Kebutuhan | Metrik | Verifikasi | Prioritas |
|---|---|---|---|---|
| NFR-SEC-08 | Tidak ada kebocoran data antar tenant | Nol temuan pada uji isolasi yang mencakup 100 persen endpoint | Uji otomatis wajib di CI, build gagal bila ada temuan | Must |
| NFR-SEC-09 | Tidak ada akses objek langsung tanpa otorisasi | Setiap endpoint yang menerima ID memiliki uji IDOR | Uji otomatis | Must |
| NFR-SEC-10 | Feature flag ditegakkan di server | Endpoint modul nonaktif mengembalikan 403, diuji per modul | Uji otomatis per modul | Must |
| NFR-SEC-11 | Izin ditegakkan di server | Setiap izin sensitif memiliki uji akses ditolak | Uji otomatis | Must |
| NFR-SEC-12 | Row Level Security aktif pada tabel kritis | Policy terpasang pada `sales`, `sale_items`, `stock_movements`, `customers`, `audit_logs` | Uji query langsung tanpa scope aplikasi | Should |

### 5.3 Data dan Aplikasi

| Kode | Kebutuhan | Metrik | Verifikasi | Prioritas |
|---|---|---|---|---|
| NFR-SEC-13 | Tidak ada kerentanan OWASP Top 10 | Nol temuan kritis dan tinggi | Pemindaian otomatis di CI, uji penetrasi tahunan | Must |
| NFR-SEC-14 | Data kartu tidak disimpan | Tidak pernah menyimpan PAN penuh atau CVV. Hanya 4 digit terakhir dan nomor referensi | Review skema dan kode | Must |
| NFR-SEC-15 | Data sensitif di terminal terlindungi | Field sensitif di IndexedDB dienkripsi dengan kunci turunan device token | Review kode | Should |
| NFR-SEC-16 | Unggahan berkas tervalidasi | Validasi MIME dan ekstensi, batas 10 MB, disimpan tanpa hak eksekusi | Uji otomatis dengan berkas berbahaya | Must |
| NFR-SEC-17 | Rahasia tidak masuk repositori | Nol temuan pemindai rahasia | Pemindai rahasia di pre-commit dan CI | Must |
| NFR-SEC-18 | Dependensi bebas kerentanan diketahui | Nol kerentanan kritis dan tinggi tanpa mitigasi | Dependabot dan audit mingguan | Must |
| NFR-SEC-19 | Impersonasi terkendali dan transparan | Wajib alasan, maksimal 60 menit, tercatat, dan terlihat oleh pemilik tenant | Uji fungsional dan review audit log | Must |
| NFR-SEC-20 | API publik dibatasi laju | 60 permintaan per menit per API key, dapat dinaikkan per paket | Uji rate limit | Should |

---

## 6. Keandalan dan Integritas Data

Bagian ini paling kritis untuk aplikasi POS. Kesalahan performa membuat pengguna kesal, kesalahan integritas data membuat pengguna kehilangan uang.

| Kode | Kebutuhan | Metrik | Verifikasi | Prioritas |
|---|---|---|---|---|
| NFR-REL-01 | Transaksi tidak pernah hilang | Nol kehilangan pada uji chaos: mati listrik, browser ditutup paksa, jaringan putus saat kirim | Uji chaos terstruktur sebelum setiap rilis mayor | Must |
| NFR-REL-02 | Transaksi tidak pernah duplikat | Nol duplikat pada 10.000 kali pengiriman ulang acak | Uji idempotensi otomatis | Must |
| NFR-REL-03 | Nilai uang tidak pernah kehilangan presisi | Nol selisih pada uji perhitungan 100.000 kombinasi diskon, pajak, dan pembulatan | Uji properti (property based testing) | Must |
| NFR-REL-04 | Perhitungan klien dan server identik | Nol selisih pada golden dataset 500 skenario yang dijalankan di kedua sisi | Uji bersama di CI, gagal bila berbeda | Must |
| NFR-REL-05 | Saldo stok konsisten dengan ledger | Nol selisih pada rekonsiliasi harian, selisih apa pun dilaporkan bukan diperbaiki diam diam | Job rekonsiliasi harian dengan pemantauan | Must |
| NFR-REL-06 | Transaksi selesai tidak dapat diubah | Percobaan mengubah nilai transaksi selesai ditolak di tingkat basis data | Uji trigger | Must |
| NFR-REL-07 | Laporan historis stabil | Perubahan harga, nama produk, atau tarif pajak tidak mengubah laporan periode lampau | Uji regresi dengan dataset beku | Must |
| NFR-REL-08 | Agregat laporan akurat | Selisih agregat versus perhitungan langsung nol setelah rekonsiliasi H-1 | Job verifikasi harian | Must |
| NFR-REL-09 | Satu shift terbuka per terminal | Ditegakkan constraint basis data, bukan hanya logika aplikasi | Uji konkurensi | Must |
| NFR-REL-10 | Job antrean membawa konteks tenant | Nol job tanpa `tenant_id` eksplisit | Uji otomatis dan analisis statis pada base class | Must |
| NFR-REL-11 | Penyimpangan jam klien terdeteksi | Selisih di atas 5 menit ditandai pada sinkronisasi dan dilaporkan | Uji sinkronisasi dengan jam dimanipulasi | Should |
| NFR-REL-12 | Kegagalan sinkronisasi terlihat | Terminal dengan antrean di atas 20 transaksi atau usia di atas 2 jam memicu peringatan di UI dan notifikasi pemilik | Uji fungsional dan pemantauan | Must |

---

## 7. Usability

| Kode | Kebutuhan | Metrik | Verifikasi | Prioritas |
|---|---|---|---|---|
| NFR-UX-01 | Kasir baru dapat bekerja tanpa pelatihan formal | 8 dari 10 peserta uji menyelesaikan transaksi pertama dalam 3 menit tanpa bantuan | Uji usability moderasi, 10 peserta | Must |
| NFR-UX-02 | Transaksi retail efisien | Maksimal 5 interaksi layar dari mulai sampai selesai | Analisis alur dan pengukuran | Must |
| NFR-UX-03 | Layar kasir nyaman disentuh | Target sentuh minimal 44 x 44 piksel pada elemen transaksi, jarak antar target minimal 8 piksel | Audit desain otomatis | Must |
| NFR-UX-04 | Aksi destruktif tidak terjadi tanpa sengaja | Konfirmasi eksplisit pada void, hapus, dan tutup shift. Tidak ada konfirmasi berbasis hover | Review UI | Must |
| NFR-UX-05 | Pesan kesalahan dapat ditindaklanjuti | Setiap pesan menyebutkan penyebab dan langkah berikutnya dalam bahasa awam. Nol kode teknis mentah di UI | Audit teks UI | Must |
| NFR-UX-06 | Status koneksi selalu terlihat | Indikator online, offline, dan jumlah antrean sinkron tampil permanen di layar kasir | Review UI | Must |
| NFR-UX-07 | Onboarding tidak memaksa | Setiap langkah setelah pembuatan bisnis dapat dilewati. Time to first transaction di bawah 15 menit | Uji usability | Must |
| NFR-UX-08 | Kompleksitas tersembunyi secara default | Pengguna preset Minimal melihat maksimal 6 item menu utama | Review konfigurasi preset | Must |
| NFR-UX-09 | Operasi utama dapat dilakukan lewat papan ketik | Shortcut untuk cari, bayar, tahan, dan batal pada terminal desktop | Uji fungsional | Should |
| NFR-UX-10 | Angka besar mudah dibaca | Total dan kembalian memakai ukuran minimal 24 piksel dengan pemisah ribuan | Review UI | Must |

---

## 8. Aksesibilitas

| Kode | Kebutuhan | Metrik | Verifikasi | Prioritas |
|---|---|---|---|---|
| NFR-A11Y-01 | Kontras memadai | Rasio kontras minimal 4,5:1 untuk teks normal, 3:1 untuk teks besar | axe-core di CI | Must |
| NFR-A11Y-02 | Informasi tidak hanya lewat warna | Status meja, stok, dan transaksi memakai ikon atau label selain warna | Review UI | Must |
| NFR-A11Y-03 | Navigasi papan ketik lengkap pada back office | Seluruh fungsi dapat dicapai tanpa tetikus, fokus terlihat jelas | Uji manual dan axe-core | Should |
| NFR-A11Y-04 | Struktur semantik benar | Nol pelanggaran WCAG 2.1 AA level kritis | axe-core di CI | Should |
| NFR-A11Y-05 | Teks dapat diperbesar | Aplikasi tetap berfungsi pada zoom 200 persen | Uji manual | Should |

---

## 9. Kompatibilitas

| Kode | Kebutuhan | Metrik | Verifikasi | Prioritas |
|---|---|---|---|---|
| NFR-COMP-01 | Dukungan browser | Chrome, Edge, Safari, Firefox dua versi terakhir | Uji lintas browser di CI | Must |
| NFR-COMP-02 | Dukungan perangkat kasir | Android 9 ke atas, iPadOS 15 ke atas, Windows 10 ke atas | Uji pada perangkat referensi | Must |
| NFR-COMP-03 | PWA dapat dipasang | Lolos kriteria installability, berfungsi offline setelah kunjungan pertama | Lighthouse PWA audit | Must |
| NFR-COMP-04 | Kompatibel printer thermal umum | Minimal 5 model printer teruji, mencakup Epson, Xprinter, dan Iware | Uji perangkat keras terdokumentasi | Must |
| NFR-COMP-05 | Barcode scanner tanpa driver | Berfungsi dengan scanner mode keyboard wedge tanpa konfigurasi | Uji perangkat keras | Must |
| NFR-COMP-06 | Berfungsi pada layar kecil | Layar kasir dapat dioperasikan pada lebar minimal 768 piksel, back office pada 360 piksel | Uji responsif | Must |
| NFR-COMP-07 | Toleran terhadap jaringan buruk | Berfungsi pada latensi 500 ms dan packet loss 5 persen | Uji dengan throttling jaringan | Should |

---

## 10. Maintainability

| Kode | Kebutuhan | Metrik | Verifikasi | Prioritas |
|---|---|---|---|---|
| NFR-MNT-01 | Cakupan uji memadai | Minimal 70 persen keseluruhan, minimal 90 persen pada modul Sales, Payment, Inventory, dan PricingEngine | Laporan coverage di CI, gagal bila turun | Must |
| NFR-MNT-02 | Batas modul tidak dilanggar | Nol pelanggaran aturan ketergantungan | Deptrac di CI, build gagal bila melanggar | Must |
| NFR-MNT-03 | Analisis statis lulus | PHPStan level 6 tanpa error, TypeScript strict mode tanpa error | CI | Must |
| NFR-MNT-04 | Gaya kode konsisten | Pint dan ESLint tanpa pelanggaran | CI | Must |
| NFR-MNT-05 | Migrasi dapat dibalik | Setiap migrasi memiliki `down` yang teruji, kecuali migrasi data satu arah yang didokumentasikan | Uji migrasi maju mundur di CI | Must |
| NFR-MNT-06 | Kontrak API terdokumentasi | OpenAPI spec dihasilkan otomatis dan tervalidasi terhadap implementasi | CI | Should |
| NFR-MNT-07 | Waktu pipeline terkendali | Pipeline penuh di bawah 10 menit | Pemantauan CI | Should |
| NFR-MNT-08 | Lingkungan pengembangan cepat disiapkan | Dari clone sampai aplikasi berjalan di bawah 15 menit dengan satu perintah | Uji pada mesin bersih | Should |
| NFR-MNT-09 | Feature flag baru mudah ditambahkan | Menambah modul baru tidak memerlukan perubahan pada modul lain | Review arsitektur | Must |

---

## 11. Observability

| Kode | Kebutuhan | Metrik | Verifikasi | Prioritas |
|---|---|---|---|---|
| NFR-OBS-01 | Error terpantau dengan konteks | Setiap exception membawa tag `tenant_id`, `business_id`, `outlet_id`, dan `user_id` | Review konfigurasi Sentry | Must |
| NFR-OBS-02 | Log terstruktur | Format JSON dengan `request_id` yang dapat ditelusuri lintas layanan | Review konfigurasi logging | Must |
| NFR-OBS-03 | Metrik bisnis terpantau | Transaksi per menit, kegagalan sinkronisasi, kegagalan cetak, dan panjang outbox terpantau realtime | Dasbor internal | Must |
| NFR-OBS-04 | Peringatan proaktif | Peringatan terkirim bila error rate melewati 1 persen, antrean melewati 1.000 job, atau uptime turun | Konfigurasi alerting | Must |
| NFR-OBS-05 | Query lambat terdeteksi | Query di atas 500 ms tercatat dan ditinjau mingguan | Laravel Pulse | Should |
| NFR-OBS-06 | Jejak audit lengkap | 100 persen aksi sensitif tercatat, diverifikasi lewat daftar periksa aksi | Uji otomatis per aksi sensitif | Must |
| NFR-OBS-07 | Data pribadi tidak masuk log | Nol nomor telepon, email, atau PIN dalam log aplikasi | Pemindaian log otomatis | Must |

---

## 12. Data, Retensi, dan Kepatuhan

| Kode | Kebutuhan | Metrik | Verifikasi | Prioritas |
|---|---|---|---|---|
| NFR-DATA-01 | Retensi transaksi memadai | Minimal 5 tahun untuk paket berbayar, 12 bulan untuk paket gratis | Review kebijakan dan job arsip | Must |
| NFR-DATA-02 | Audit log tersimpan cukup lama | 12 bulan aktif, diarsipkan setelahnya | Job arsip terjadwal | Must |
| NFR-DATA-03 | Portabilitas data terjamin | Pemilik dapat mengekspor seluruh data bisnisnya dalam format CSV atau JSON, selesai di bawah 30 menit | Uji fungsional ekspor | Must |
| NFR-DATA-04 | Penghapusan akun terkendali | Soft delete dengan tenggang 30 hari, pemberitahuan sebelum purge, purge permanen setelahnya | Uji fungsional | Must |
| NFR-DATA-05 | Kepatuhan UU PDP | Kebijakan privasi tersedia, persetujuan tercatat, mekanisme permintaan penghapusan data pribadi tersedia | Review kepatuhan | Must |
| NFR-DATA-06 | Data disimpan di wilayah yang sesuai | Basis data dan backup berada di wilayah Indonesia atau Singapura sesuai perjanjian layanan | Review infrastruktur | Should |
| NFR-DATA-07 | Backup terenkripsi | Enkripsi saat transit dan saat disimpan | Review konfigurasi backup | Must |
| NFR-DATA-08 | Data tenant tidak dipakai tanpa izin | Analitik agregat lintas tenant hanya dalam bentuk anonim, diatur dalam perjanjian layanan | Review kebijakan | Must |

---

## 13. Lokalisasi

| Kode | Kebutuhan | Metrik | Verifikasi | Prioritas |
|---|---|---|---|---|
| NFR-LOC-01 | Bahasa antarmuka | Bahasa Indonesia sebagai default, Inggris tersedia | Uji kelengkapan berkas terjemahan | Should |
| NFR-LOC-02 | Format mata uang benar | Rupiah tanpa desimal secara default, dapat dikonfigurasi | Uji unit format | Must |
| NFR-LOC-03 | Zona waktu ditangani benar | Laporan harian memakai zona waktu outlet, bukan zona waktu server | Uji dengan outlet WIB, WITA, dan WIT | Must |
| NFR-LOC-04 | Jam tutup buku dihormati | Transaksi pukul 01.00 masuk hari sebelumnya bila jam tutup buku diatur pukul 03.00 | Uji unit `business_date` | Must |
| NFR-LOC-05 | Tidak ada teks tertanam di kode | Seluruh teks antarmuka melalui berkas terjemahan | Pemindaian otomatis | Should |

---

## 14. Testability

| Kode | Kebutuhan | Metrik | Verifikasi | Prioritas |
|---|---|---|---|---|
| NFR-TEST-01 | Seed data representatif tersedia | Seeder menghasilkan tenant lengkap untuk empat tipe bisnis dengan data satu bulan | Uji seeder | Must |
| NFR-TEST-02 | Uji tidak bergantung layanan eksternal | Payment gateway, WhatsApp, dan storage memiliki implementasi tiruan | Review kode uji | Must |
| NFR-TEST-03 | Uji offline dapat dijalankan otomatis | Skenario putus jaringan diuji lewat Playwright dengan kontrol jaringan | Uji E2E | Must |
| NFR-TEST-04 | Uji kombinasi feature flag terkendali | Uji dijalankan pada 5 preset tipe bisnis, bukan seluruh kombinasi | Konfigurasi matriks uji | Must |
| NFR-TEST-05 | Uji beban dapat direproduksi | Skrip k6 tersimpan di repositori dengan dataset acuan | Review repositori | Should |
| NFR-TEST-06 | Data uji tidak mengandung data nyata | Staging memakai data anonim | Review proses penyegaran staging | Must |

---

## 15. Matriks Prioritas dan Fase

| Kelompok NFR | Fase 1 | Fase 2 | Fase 3 | Fase 4 | Fase 5 |
|---|:--:|:--:|:--:|:--:|:--:|
| Performa layar kasir | Penuh | Penuh | Penuh | Penuh | Penuh |
| Performa laporan | Dasar | Dasar | Penuh | Penuh | Penuh |
| Skalabilitas | Fondasi | Fondasi | Menengah | Penuh | Penuh |
| Ketersediaan | 99 persen | 99 persen | 99,5 persen | 99,5 persen | 99,5 persen |
| Offline | Tidak | Tidak | Tidak | Penuh | Penuh |
| Keamanan autentikasi | Penuh | Penuh | Penuh | Penuh | Penuh |
| Isolasi tenant | Penuh | Penuh | Penuh | Penuh | Penuh |
| Integritas data | Penuh | Penuh | Penuh | Penuh | Penuh |
| Usability | Penuh | Penuh | Penuh | Penuh | Penuh |
| Aksesibilitas | Dasar | Dasar | Menengah | Menengah | Penuh |
| Observability | Dasar | Menengah | Penuh | Penuh | Penuh |
| Kepatuhan data | Dasar | Dasar | Menengah | Penuh | Penuh |

**Yang tidak boleh dikompromikan sejak Fase 1:** isolasi tenant, integritas data, keamanan autentikasi, dan performa layar kasir. Empat hal ini sulit diperbaiki belakangan karena menyangkut struktur data dan kepercayaan pengguna.

**Yang boleh bertahap:** aksesibilitas lanjutan, observability, skalabilitas, dan kepatuhan formal.

---

## 16. Anggaran Performa (Performance Budget)

Ambang yang menggagalkan pipeline bila dilampaui:

| Metrik | Anggaran | Konsekuensi Bila Dilampaui |
|---|---|---|
| Bundel JavaScript awal | 500 KB terkompresi | Build gagal |
| Bundel CSS | 100 KB terkompresi | Peringatan |
| Largest Contentful Paint layar kasir | 2,5 detik pada 3G | Build gagal |
| Total Blocking Time | 300 ms | Peringatan |
| Jumlah query per permintaan layar kasir | 15 | Build gagal, indikasi N+1 |
| Jumlah query per permintaan dasbor | 30 | Peringatan |
| Waktu respons p95 endpoint transaksi | 300 ms | Peringatan, tinjauan wajib |
| Ukuran respons API katalog | 2 MB per halaman | Build gagal |

---

## 17. Risiko NFR

| Risiko | Dampak | Mitigasi |
|---|---|---|
| Target performa kasir tidak tercapai pada perangkat kelas rendah | Pengguna segmen bawah tidak terlayani | Uji pada perangkat kelas rendah sejak sprint pertama, bukan menjelang rilis |
| Uji isolasi tenant menjadi beban dan mulai dilewati | Risiko kebocoran meningkat diam diam | Uji dijadikan otomatis dan wajib, tidak bergantung disiplin manual |
| Duplikasi PricingEngine menyimpang seiring waktu | Total berbeda antara offline dan online | Golden dataset bersama dijalankan di kedua pipeline, perbedaan menggagalkan build |
| Anggaran performa dilonggarkan berulang kali | Degradasi bertahap yang sulit dibalik | Perubahan anggaran memerlukan persetujuan eksplisit dan alasan tertulis |
| Cakupan uji 90 persen pada modul kritis memperlambat pengembangan | Kecepatan rilis turun | Fokus pada uji perilaku, bukan uji implementasi. Terima cakupan lebih rendah pada kode boilerplate |
| Uptime 99,5 persen sulit dicapai pada VPS tunggal | Janji layanan tidak terpenuhi | Mulai dengan komitmen 99 persen pada Fase 1 dan 2, naikkan setelah infrastruktur redundan tersedia |
