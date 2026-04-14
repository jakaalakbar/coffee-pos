# Product Requirements Document (PRD) - Coffee POS System

## 1. Deskripsi Produk

**Coffee POS** adalah sistem manajemen penjualan (Point of Sales) yang dirancang khusus untuk operasional kedai kopi skala menengah. Sistem ini dibangun menggunakan bahasa pemrograman **Go** untuk performa yang optimal dan **MySQL** sebagai penyimpanan data yang handal. Fokus utama aplikasi ini adalah untuk menyederhanakan proses transaksi di kasir serta memberikan kontrol penuh kepada pemilik (owner) dalam memantau bisnis secara real-time.

## 2. User Roles

Terdapat dua peran pengguna utama dalam sistem:
| Role | Deskripsi Akses |
| :--- | :--- |
| **Owner** | Akses penuh ke seluruh modul sistem (Master Data, Manajemen User, Stok, dan Laporan). |
| **Cashier** | Akses terbatas pada modul operasional harian (Shift, Penjualan, dan Checkout). |

---

## 3. Fitur Owner

### 3.1 Manajemen Produk

Fitur untuk mengelola katalog produk yang dijual.

- **Fitur:** CRUD produk, soft delete, upload foto, status aktif/nonaktif.
- **Acceptance Criteria:**
  - Owner dapat menambah, mengubah, dan menonaktifkan produk.
  - Data yang dihapus tidak benar-benar hilang dari database (soft delete).
  - Produk dapat memiliki foto dan status ketersediaan.

### 3.2 Manajemen Kategori Produk

Pengelompokan produk untuk mempermudah pencarian (misal: Espresso Based, Non-Coffee, Food).

- **Fitur:** CRUD kategori.
- **Acceptance Criteria:**
  - Owner dapat menambahkan kategori baru dan menghubungkannya dengan produk.

### 3.3 Manajemen Stok & Riwayat Pergerakan

Sistem pemantauan jumlah stok produk secara real-time.

- **Fitur:** Update stok manual dan log riwayat pergerakan stok (masuk/keluar).
- **Acceptance Criteria:**
  - Setiap perubahan stok (penjualan atau update manual) tercatat di tabel riwayat.
  - Owner dapat melihat sisa stok terakhir.

### 3.4 Manajemen Meja

Pengelolaan layout atau nomor meja di coffee shop.

- **Fitur:** CRUD data meja.
- **Acceptance Criteria:**
  - Owner dapat mendaftarkan nomor atau nama meja yang tersedia.

### 3.5 Manajemen User Cashier

Pengelolaan akun staf yang bertugas di kasir.

- **Fitur:** CRUD user cashier.
- **Acceptance Criteria:**
  - Owner dapat membuat akun login untuk kasir dengan password yang aman.

### 3.6 Dashboard Laporan

Visualisasi data performa bisnis.

- **Fitur:** Revenue harian/mingguan/bulanan, produk terlaris, performa transaksi per kasir.
- **Acceptance Criteria:**
  - Dashboard menampilkan grafik atau angka ringkasan pendapatan dengan filter waktu.
  - Daftar produk paling banyak terjual ditampilkan dengan jelas.

### 3.7 Manajemen Promo & Diskon

Pengaturan strategi harga untuk menarik pelanggan.

- **Fitur:** CRUD Promo (persentase atau nominal).
- **Acceptance Criteria:**
  - Promo memiliki masa berlaku (opsional) dan dapat dipilih saat transaksi.
  - Diskon dapat berupa potongan harga langsung (Rp) atau persentase (%).

### 3.8 Export Laporan ke CSV

Kemudahan pengolahan data di luar aplikasi.

- **Fitur:** Tombol export laporan transaksi ke format CSV.
- **Acceptance Criteria:**
  - File CSV yang dihasilkan berisi data transaksi detail sesuai filter yang dipilih.

---

## 4. Fitur Cashier

### 4.1 Manajemen Shift

Prosedur pembukaan dan penutupan operasional kasir.

- **Fitur:** Buka shift (masukkan modal kas), tutup shift (input rekap kas akhir).
- **Acceptance Criteria:**
  - Kasir wajib membuka shift sebelum bisa melakukan transaksi.
  - Saat tutup shift, sistem menghitung selisih antara kas di sistem dan kas fisik.

### 4.2 Buat Transaksi Baru

Inti dari operasional harian.

- **Fitur:** Pilih meja, pilih produk, atur quantity.
- **Acceptance Criteria:**
  - Kasir dapat memilih produk dari katalog dan memasukkannya ke keranjang belanja.
  - Setiap transaksi harus terasosiasi dengan nomor meja.

### 4.3 Apply Promo

Penerapan diskon pada transaksi yang berjalan.

- **Fitur:** Dropdown atau input kode promo yang sedang aktif.
- **Acceptance Criteria:**
  - Total belanja berkurang secara otomatis setelah promo diterapkan.

### 4.4 Checkout dengan Midtrans Snap

Integrasi pembayaran cashless.

- **Fitur:** Integrasi workflow pembayaran melalui Midtrans.
- **Acceptance Criteria:**
  - Pelanggan dapat membayar via E-Wallet (Gopay/OVO), Virtual Account, atau QRIS.
  - Status transaksi di POS otomatis berubah menjadi "Success" setelah notifikasi dari Midtrans diterima.

### 4.5 Lihat Riwayat Transaksi Shift

Pemantauan transaksi harian oleh kasir.

- **Fitur:** List transaksi yang berhasil di shift yang sedang aktif.
- **Acceptance Criteria:**
  - Kasir hanya dapat melihat transaksi yang dilakukan di bawah akun dan shift-nya sendiri.

---

## 5. Business Rules

1. **Penerangan Stok:** Stok produk hanya akan berkurang secara otomatis setelah sistem menerima webhook **"Status: Confirmed/Settlement"** dari Midtrans.
2. **Validasi Shift:** Kasir wajib melakukan "Buka Shift" sebelum sistem mengizinkan pembuatan transaksi baru.
3. **Batasan Promo:** Satu transaksi hanya diperbolehkan menggunakan maksimal **satu promo** (tidak dapat digabungkan).
4. **Integritas Order:** Order yang sudah melalui proses checkout (sudah terbit Snap Token/URL) tidak dapat diubah itemnya (quantity atau produk). Jika ada perubahan, transaksi lama harus dibatalkan dan dibuat transaksi baru.
5. **Validasi Stok Saat Pesan:** Produk tidak dapat dipesan jika stok di sistem menunjukkan angka 0 (kecuali produk kategori non-stok).
6. **Integritas Data:** Data transaksi yang sudah berstatus "Success" tidak dapat diubah atau dihapus (harus melalui mekanisme void/refund di luar scope ini).
7. **Keamanan Login:** Password user harus di-hash menggunakan algoritma aman (Bcrypt).
8. **Session Management:** Login kasir akan expire jika tidak ada aktivitas dalam periode waktu tertentu (misal: 12 jam).

---

## 6. Out of Scope

1. Manajemen Purchasing/Pengadaan barang ke supplier.
2. Sistem Membership/Loyalty Point untuk pelanggan.
3. Pencetakan struk fisik via thermal printer (fokus pada digital receipt).
4. Integrasi dengan GrabFood/GoFood API.
5. Manajemen penggajian (Payroll) karyawan.

---

**Penyusun:** Antigravity AI
**Tanggal:** 14 April 2026
