# Database Schema Design - Coffee POS System

Dokumen ini merinci desain basis data MySQL untuk sistem Coffee POS berdasarkan kebutuhan fungsional di `PRD.md`.
_Catatan: Desain ini sudah mencakup penyesuaian terbaru (seperti is_active di users, is_available di products, struktur pergerakan stok, dll)._

## 1. Daftar Tabel

| Nama Tabel            | Deskripsi                                                                |
| :-------------------- | :----------------------------------------------------------------------- |
| **users**             | Menyimpan data otentikasi pengguna dan peran (Owner, Cashier).           |
| **categories**        | Mengelompokkan produk ke dalam kategori tertentu.                        |
| **products**          | Katalog menu dan barang fisik yang dijual, beserta harga dan statusnya.  |
| **stocks**            | Riwayat dan data mutasi stok barang (masuk, keluar, retur, penyesuaian). |
| **tables**            | Informasi tata letak atau nomor meja untuk pemesanan dine-in.            |
| **shifts**            | Mencatat sesi waktu kerja kasir beserta modal kas.                       |
| **promos**            | Menyimpan aturan besaran diskon/promo yang berlaku.                      |
| **transactions**      | Induk/header setiap nota transaksi.                                      |
| **transaction_items** | Detail produk-produk apa saja yang dipesan dalam satu transaksi.         |

---

## 2. Struktur Tabel

### Tipe Data Umum

- **ID (Primary Key)**: Menggunakan `VARCHAR(36)` format UUID.
- **Harga / Nominal**: Menggunakan `BIGINT` untuk menyimpan nominal uang dalam satuan sen (rupiah terkecil) agar menghindari masalah _precision/floating point_.
- **Waktu**: Semua tabel dilengkapi kolom `created_at` dan `updated_at` bertipe `TIMESTAMP`.

### 2.1 users

| Kolom      | Tipe Data                | Constraint       | Keterangan              |
| :--------- | :----------------------- | :--------------- | :---------------------- |
| id         | VARCHAR(36)              | PRIMARY KEY      | UUID                    |
| name       | VARCHAR(100)             | NOT NULL         | Nama pengguna           |
| email      | VARCHAR(100)             | UNIQUE, NOT NULL | Digunakan untuk login   |
| password   | VARCHAR(255)             | NOT NULL         | Hash Bcrypt password    |
| role       | ENUM('owner', 'cashier') | NOT NULL         | Hak akses pengguna      |
| is_active  | BOOLEAN                  | DEFAULT TRUE     | Status aktif akun user  |
| created_at | TIMESTAMP                | NOT NULL         | Waktu record dibuat     |
| updated_at | TIMESTAMP                | NOT NULL         | Waktu record diperbarui |

### 2.2 categories

| Kolom      | Tipe Data    | Constraint  | Keterangan                    |
| :--------- | :----------- | :---------- | :---------------------------- |
| id         | VARCHAR(36)  | PRIMARY KEY | UUID                          |
| name       | VARCHAR(100) | NOT NULL    | Nama kategori (cth: Espresso) |
| created_at | TIMESTAMP    | NOT NULL    | -                             |
| updated_at | TIMESTAMP    | NOT NULL    | -                             |

### 2.3 products

_(Fitur CRUD memiliki "Soft Delete")_
| Kolom | Tipe Data | Constraint | Keterangan |
| :--- | :--- | :--- | :--- |
| id | VARCHAR(36) | PRIMARY KEY | UUID |
| category_id | VARCHAR(36) | FOREIGN KEY | Relasi ke `categories.id` |
| name | VARCHAR(150) | NOT NULL | Nama/judul produk |
| description | TEXT | NULL | Deskripsi tambahan |
| price | BIGINT | NOT NULL | Harga dasar dalam satuan sen |
| image_url | VARCHAR(255) | NULL | Foto produk |
| is_available| BOOLEAN | DEFAULT TRUE | Tanda ketersediaan (aktif/non-aktif) |
| created_at | TIMESTAMP | NOT NULL | - |
| updated_at | TIMESTAMP | NOT NULL | - |
| deleted_at | TIMESTAMP | NULL | Diisi jika produk di-soft-delete |

### 2.4 stocks

_(Mencatat mutasi / riwayat pergerakan stok)_
| Kolom | Tipe Data | Constraint | Keterangan |
| :--- | :--- | :--- | :--- |
| id | VARCHAR(36) | PRIMARY KEY | UUID |
| product_id | VARCHAR(36) | FOREIGN KEY | Relasi ke `products.id` |
| quantity | INT | NOT NULL | Jumlah perubahan/stok (Positif/Negatif)|
| type | ENUM('purchase', 'sale', 'adjustment', 'return') | NOT NULL | Tipe pergerakan mutasi stok |
| note | VARCHAR(255) | NULL | Catatan alasan perubahan |
| created_at | TIMESTAMP | NOT NULL | Waktu pencatatan mutasi |
| updated_at | TIMESTAMP | NOT NULL | - |

### 2.5 tables

| Kolom      | Tipe Data                     | Constraint          | Keterangan                |
| :--------- | :---------------------------- | :------------------ | :------------------------ |
| id         | VARCHAR(36)                   | PRIMARY KEY         | UUID                      |
| name       | VARCHAR(50)                   | NOT NULL            | Nomor meja (cth: Meja 12) |
| status     | ENUM('available', 'occupied') | DEFAULT 'available' | Status penggunaan dimeja  |
| created_at | TIMESTAMP                     | NOT NULL            | -                         |
| updated_at | TIMESTAMP                     | NOT NULL            | -                         |

### 2.6 shifts

| Kolom        | Tipe Data              | Constraint     | Keterangan                          |
| :----------- | :--------------------- | :------------- | :---------------------------------- |
| id           | VARCHAR(36)            | PRIMARY KEY    | UUID                                |
| cashier_id   | VARCHAR(36)            | FOREIGN KEY    | Relasi ke `users.id`                |
| opening_cash | BIGINT                 | NOT NULL       | Modal awal saat buka shift (sen)    |
| closing_cash | BIGINT                 | NULL           | Rekap uang akhir saat tutup (sen)   |
| total_sales  | BIGINT                 | DEFAULT 0      | Total pendapatan selama shift (sen) |
| status       | ENUM('open', 'closed') | DEFAULT 'open' | Menandakan session beroperasi       |
| start_time   | TIMESTAMP              | NOT NULL       | Jam mula shift dibuka               |
| end_time     | TIMESTAMP              | NULL           | Waktu menutup / berakhir shift      |
| created_at   | TIMESTAMP              | NOT NULL       | -                                   |
| updated_at   | TIMESTAMP              | NOT NULL       | -                                   |

### 2.7 promos

| Kolom      | Tipe Data                     | Constraint   | Keterangan                         |
| :--------- | :---------------------------- | :----------- | :--------------------------------- |
| id         | VARCHAR(36)                   | PRIMARY KEY  | UUID                               |
| name       | VARCHAR(100)                  | NOT NULL     | Judul promo (cth: "Diskon 20%")    |
| promo_type | ENUM('percentage', 'nominal') | NOT NULL     | Jenis dan bentuk potongan          |
| amount     | BIGINT                        | NOT NULL     | Angka persentase atau nominal(sen) |
| is_active  | BOOLEAN                       | DEFAULT TRUE | Dapat digunakan atau tidak         |
| created_at | TIMESTAMP                     | NOT NULL     | -                                  |
| updated_at | TIMESTAMP                     | NOT NULL     | -                                  |

### 2.8 transactions

| Kolom             | Tipe Data                                        | Constraint        | Keterangan                         |
| :---------------- | :----------------------------------------------- | :---------------- | :--------------------------------- |
| id                | VARCHAR(36)                                      | PRIMARY KEY       | UUID                               |
| shift_id          | VARCHAR(36)                                      | FOREIGN KEY       | Relasi ke `shifts.id`              |
| table_id          | VARCHAR(36)                                      | FOREIGN KEY       | Relasi ke `tables.id`              |
| cashier_id        | VARCHAR(36)                                      | FOREIGN KEY       | Relasi ke `users.id`               |
| promo_id          | VARCHAR(36)                                      | FOREIGN KEY       | Relasi ke `promos.id` (Nullable)   |
| subtotal_amount   | BIGINT                                           | NOT NULL          | Total harga sblm diskon (sen)      |
| discount_amount   | BIGINT                                           | DEFAULT 0         | Nominal dipotong karena promo      |
| final_amount      | BIGINT                                           | NOT NULL          | Total harga akhir (sen)            |
| payment_status    | ENUM('pending', 'success', 'failed', 'refunded') | DEFAULT 'pending' | Status proses final Midtrans       |
| midtrans_token    | VARCHAR(255)                                     | NULL              | URL / Token Midtrans Snap          |
| midtrans_order_id | VARCHAR(100)                                     | UNIQUE, NULL      | Reference order untuk Midtrans     |
| created_at        | TIMESTAMP                                        | NOT NULL          | Transaksi dibuat                   |
| updated_at        | TIMESTAMP                                        | NOT NULL          | Update terakhir (seperti callback) |

### 2.9 transaction_items

| Kolom          | Tipe Data   | Constraint  | Keterangan                                      |
| :------------- | :---------- | :---------- | :---------------------------------------------- |
| id             | VARCHAR(36) | PRIMARY KEY | UUID                                            |
| transaction_id | VARCHAR(36) | FOREIGN KEY | Induk transaksi `transactions.id`               |
| product_id     | VARCHAR(36) | FOREIGN KEY | Relasi ke `products.id`                         |
| quantity       | INT         | NOT NULL    | Jumlah porsi dipesan                            |
| price_at_buy   | BIGINT      | NOT NULL    | Harga produk yang direkam saat itu              |
| subtotal       | BIGINT      | NOT NULL    | Total harga per item (quantity \* price_at_buy) |
| created_at     | TIMESTAMP   | NOT NULL    | -                                               |
| updated_at     | TIMESTAMP   | NOT NULL    | -                                               |

---

## 3. Relasi Antar Tabel (Foreign Keys)

- `products.category_id` → `categories.id` | Mengklasifikasikan setiap produk ke kategori tertentu
- `stocks.product_id` → `products.id` | Mencatat mutasi stok untuk produk yang bersangkutan
- `shifts.cashier_id` → `users.id` | Mengidentifikasi kasir yang membuka dan memegang shift tersebut
- `transactions.shift_id` → `shifts.id` | Mengelompokkan pendapatan transaksi dalam satu periode shift layar kasir
- `transactions.table_id` → `tables.id` | Menandakan transaksi ditujukan untuk pesanan meja mana
- `transactions.cashier_id` → `users.id` | Melacak kasir spesifik yang melayani nota transaksi ini
- `transactions.promo_id` → `promos.id` | Menerapkan aturan diskon dari promo yang valid
- `transaction_items.transaction_id` → `transactions.id` | Menggabungkan rincian pesanan dengan nota transaksi induk (header)
- `transaction_items.product_id` → `products.id` | Menghubungkan item pesanan dengan master profil produk di katalog

---

## 4. Keputusan Desain Penting (Design Decisions)

1. **Penerapan UUID vs Auto Increment**:
   - `UUID` digunakan pada seluruh primary key untuk mencegah kelemahan keamanan _Enumeration Attack_ (menebak order ID atau user ID via URL) dan memberikan probabilitas duplikasi yang sangat-sangat minim walau tersebar ke berbagai mesin atau microservice di masa mendatang.
2. **Uang/Harga sebagai `BIGINT` satuan Sen**:
   - Nilai finansial di sistem komputasi rentan terhadap _floating point inaccuracy_ (contoh: 0.1 + 0.2 = 0.30000000000000004). Menggunakan `BIGINT` untuk mencatat bilangan bulat desimal (dalam sen / `* 100`) menjamin laporan akuntansi bernilai genap dan persisi secara absolut hingga skala sistem yang lebih kompleks.
3. **Kolom `price_at_buy` di `transaction_items`**:
   - Produk sewaktu-waktu bisa diubah harganya oleh Owner. Tanpa mencatat nilai `price_at_buy` secara independen, penggantian harga pada produk induk akan merusak laporan transaksi di waktu lampau karena relasi.
4. **Soft Delete (`deleted_at`) di produk**:
   - Jika suatu produk dihapus permanen (`DROP`/`DELETE`), maka akan menyebabkan referensi `stock logs` dan `transactions` ikut rusak. `deleted_at (TIMESTAMP)` memastikan riwayat relasi aman tapi barang tesebut tidak tampil di aplikasi.
5. **Pergerakan Stok sebagai Log (Tabel `stocks`)**:
   - Desain tidak menggunakan simple "quantity per produk". Karena kebutuhan Owner memantau "Riwayat Pergerakan Stok", tabel disesuaikan dengan skema ledger (`purchase`, `sale`, `adjustment`, `return`). Saldo stok akhir suatu barang baru akan dihitung dinamis dengan mensumigasikan (SUM) row mutasi per barang, menjadikan transparansi lossing & audit kas lebih aman.
6. **Mekanisme Shift Binding Transaksi**:
   - Transaksi dihubungkan ke `shift_id` beriringan dengan `cashier_id`. Hal ini berguna secara logic agar di masa tutup shift (End Shift), rekapan laporan dapat presisi dari hasil penjumlahan `final_amount` atas `shift_id` tersebut.
