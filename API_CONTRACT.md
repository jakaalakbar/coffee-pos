# API Contract - Coffee POS System

Dokumen ini menjelaskan struktur kontrak API untuk Coffee POS System sesuai dengan PRD dan Schema yang disepakati.

## 1. Base URL & Versioning

Semua endpoint dalam spesifikasi ini diawali dengan prefix:
`YOUR_DOMAIN/api/v1`

## 2. Authentication

Aplikasi POS ini menggunakan JSON Web Token (JWT) untuk otentikasi. Setiap request ke protected (private) endpoint wajib menyertakan token di HTTP Header:

```http
Authorization: Bearer <your_jwt_token_here>
```

## 3. Standard Response Format

Agar parsing di client konsisten, berikut adalah standar bentuk respon API.

### 3.1 Sukses

Response sukses secara umum (baik mengembalikan object tunggal, boolean, dsb):

```json
{
  "success": true,
  "message": "Operasi berhasil",
  "data": {
    // Berisi Object, Array, atau Null
  }
}
```

### 3.2 Sukses dengan Pagination (List)

Diperuntukkan saat mengambil list data (seperti Get Products) yang menerapkan paginasi halaman:

```json
{
  "success": true,
  "message": "Data berhasil diambil",
  "data": [
    // Array of objects
  ],
  "meta": {
    "current_page": 1,
    "total_pages": 5,
    "total_items": 50,
    "per_page": 10
  }
}
```

### 3.3 Error Validasi

Bentuk response khusus format input atau argumen yang salah tipe/gagal validasi:

```json
{
  "success": false,
  "message": "Validasi gagal",
  "errors": {
    "quantity": "Kuantitas tidak boleh kurang dari 1"
  }
}
```

### 3.4 Error Umum

Bentuk format error general yang tidak memerlukan rincian di setiap field/kolom:

```json
{
  "success": false,
  "message": "Gagal terhubung ke database atau rincian error dari server"
}
```

### 3.5 HTTP Status Code Standards

Aplikasi mutlak mengikuti pembagian status kode berikut:

- **200 OK**: Sukses untuk request `GET` dan `PUT`
- **201 Created**: Sukses untuk request `POST` (create data baru)
- **400 Bad Request**: Request tidak valid secara bisnis logic
- **401 Unauthorized**: Tidak ada token atau token expired
- **403 Forbidden**: Token valid tapi role pengguna tidak punya hak akses
- **404 Not Found**: Data pada parameter/ID tidak ditemukan
- **422 Unprocessable Entity**: Format input salah atau validasi gagal
- **429 Too Many Requests**: Overload permintaan (Rate limit)
- **500 Internal Server Error**: Kegagalan sistem atau Exception server

---

## 4. Daftar Endpoint

### A. Auth

#### 1. Login User

- **Method & Path:** `POST /api/v1/auth/login`
- **Akses:** Public
- **Request Body:**

```json
{
  "email": "owner@coffee.com",
  "password": "secretpassword"
}
```

- **Response Success (200):**

```json
{
  "user": {
    "id": "uuid-1234",
    "name": "Owner Budi",
    "role": "owner"
  },
  "token": "eyJhbGciOiJIUzI1..."
}
```

- **Response Error:** `401 Unauthorized` (Kredensial salah) / `400 Bad Request` (Validasi)

#### 2. Get Profile

- **Method & Path:** `GET /api/v1/auth/me`
- **Akses:** Owner, Cashier
- **Response Success (200):** Data profil user yang sedang login beserta status akunnya.

#### 3. Refresh Token

- **Method & Path:** `POST /api/v1/auth/refresh`
- **Akses:** Owner, Cashier
- **Request Body:**

```json
{
  "refresh_token": "eyJhbGci..."
}
```

- **Response Success (201):**
  Mengembalikan token akses (`token`) serta pembaharuan `refresh_token` yang baru.

```json
{
  "token": "eyJhbGci...",
  "refresh_token": "eyJhbGci..."
}
```

- **Response Error:** `401 Unauthorized` (Token tidak valid / expired) atau `422 Unprocessable Entity` (Jika input body tidak lengkap).

#### 4. Logout User

- **Method & Path:** `POST /api/v1/auth/logout`
- **Akses:** Owner, Cashier
- **Response Success (200):**

```json
{
  "success": true,
  "message": "Logout berhasil"
}
```

- **Response Error:** `401 Unauthorized` (Jika token tidak ada/tidak valid).

---

### B. Categories (Owner)

#### 1. List Kategori

- **Method & Path:** `GET /api/v1/owner/categories`
- **Akses:** Owner
- **Response Success (200):**

```json
{
  "success": true,
  "message": "Data kategori berhasil diambil",
  "data": [
    {
      "id": "uuid-cat-1",
      "name": "Espresso Based",
      "created_at": "2026-04-14T08:00:00Z",
      "updated_at": "2026-04-14T08:00:00Z"
    }
  ]
}
```

#### 2. Create Kategori

- **Method & Path:** `POST /api/v1/owner/categories`
- **Akses:** Owner
- **Request Body:**

```json
{
  "name": "Espresso Based"
}
```

- **Response Success (201):**

```json
{
  "success": true,
  "message": "Kategori berhasil dibuat",
  "data": {
    "id": "uuid-cat-new",
    "name": "Espresso Based",
    "created_at": "2026-04-14T08:15:00Z",
    "updated_at": "2026-04-14T08:15:00Z"
  }
}
```

- **Response Error:** `422 Unprocessable Entity` (Contoh: nama kategori kosong atau terlalu pendek).

#### 3. Update Kategori

- **Method & Path:** `PUT /api/v1/owner/categories/:id`
- **Akses:** Owner
- **Request Body:**

```json
{
  "name": "Non Coffee"
}
```

- **Response Success (200):**

```json
{
  "success": true,
  "message": "Kategori berhasil diupdate",
  "data": {
    "id": "uuid-cat-1",
    "name": "Non Coffee",
    "created_at": "2026-04-14T08:00:00Z",
    "updated_at": "2026-04-14T08:20:00Z"
  }
}
```

- **Response Error:** `404 Not Found` (Jika kategori tidak ditemukan) atau `422 Unprocessable Entity`.

#### 4. Delete Kategori

- **Method & Path:** `DELETE /api/v1/owner/categories/:id`
- **Akses:** Owner
- **Response Success (200):**

```json
{
  "success": true,
  "message": "Kategori berhasil dihapus"
}
```

- **Response Error:** `404 Not Found` (Kategori tidak ditemukan) atau `400 Bad Request` (Jika data masih terelasi dengan master data di tabel `products`).

---

### C. Products (Owner)

#### 1. List Produk Katalog

- **Method & Path:** `GET /api/v1/owner/products`
- **Akses:** Owner
- **Query Params:** `?category_id=uuid&search=latte&page=1&limit=10`
- **Response Success (200):** Menggunakan format paginasi.

```json
{
  "success": true,
  "message": "Data produk berhasil diambil",
  "data": [
    {
      "id": "uuid-prod-123",
      "category_id": "uuid-cat-123",
      "name": "Caramel Macchiato",
      "price": 2500000,
      "is_available": true,
      "created_at": "2026-04-14T08:00:00Z",
      "updated_at": "2026-04-14T08:00:00Z"
    }
  ],
  "meta": {
    "current_page": 1,
    "total_pages": 5,
    "total_items": 50,
    "per_page": 10
  }
}
```

#### 2. Detail Produk

- **Method & Path:** `GET /api/v1/owner/products/:id`
- **Akses:** Owner
- **Response Success (200):**

```json
{
  "success": true,
  "message": "Detail produk",
  "data": {
    "id": "uuid-prod-123",
    "category_id": "uuid-cat-123",
    "name": "Caramel Macchiato",
    "description": "Kopi caramel dengan sentuhan macchiato",
    "price": 2500000,
    "image_url": "https://storage.xyz/image.jpg",
    "is_available": true,
    "created_at": "2026-04-14T08:00:00Z",
    "updated_at": "2026-04-14T08:00:00Z"
  }
}
```

- **Response Error:** `404 Not Found` (Produk tidak ditemukan).

#### 3. Create Produk

- **Method & Path:** `POST /api/v1/owner/products`
- **Akses:** Owner
- **Request Body:** _(Gunakan `multipart/form-data` jika ada field upload `image`, atau raw JSON jika text saja)._

```json
{
  "category_id": "uuid-cat-123",
  "name": "Caramel Macchiato",
  "description": "Kopi caramel dengan sentuhan macchiato",
  "price": 2500000,
  "is_available": true
}
```

_(Catatan: Harga 2500000 = Rp 25.000 karena satuan sen)._

- **Response Success (201):** Mengembalikan full object data yang baru dibuat.
- **Response Error:** `422 Unprocessable Entity` (Kesalahan validasi form / ukuran foto).

#### 4. Update Produk

- **Method & Path:** `PUT /api/v1/owner/products/:id`
- **Akses:** Owner
- **Request Body:** Form parameters serupa dengan request _Create_.
- **Response Success (200):** Object produk setelah record di `update`.
- **Response Error:** `404 Not Found` atau `422 Unprocessable Entity`.

#### 5. Delete (Soft Delete) Produk

- **Method & Path:** `DELETE /api/v1/owner/products/:id`
- **Akses:** Owner
- **Response Success (200):**

```json
{
  "success": true,
  "message": "Produk berhasil dihapus"
}
```

- **Response Error:** `404 Not Found` (Produk tidak ditemukan).

---

### D. Stock (Owner)

#### 1. Cek Sisa Stok Produk

- **Method & Path:** `GET /api/v1/owner/products/:id/stock`
- **Akses:** Owner
- **Response Success (200):**

```json
{
  "success": true,
  "message": "Sisa stok berhasil diambil",
  "data": {
    "product_id": "uuid-prod-123",
    "total_quantity": 150
  }
}
```

- **Response Error:** `404 Not Found` (Produk tidak ditemukan).

#### 2. Penyesuaian/Mutasi Manual (Adjustment)

- **Method & Path:** `POST /api/v1/owner/products/:id/stock/adjustment`
- **Akses:** Owner
- **Request Body:**

```json
{
  "quantity": 50,
  "type": "adjustment",
  "note": "Restock beans dari supplier"
}
```

_(Catatan: `type` bisa diisi `adjustment` atau `return`. `quantity` dapat bernilai negatif jika barang tumpah/rusak)._

- **Response Success (201):** Mengembalikan data histori pergerakan stok yang baru dicatat.
- **Response Error:** `422 Unprocessable Entity` (Jika payload tidak lengkap).

#### 3. Log Pergerakan Stok (Movements)

- **Method & Path:** `GET /api/v1/owner/products/:id/stock/movements`
- **Akses:** Owner
- **Query Params:** `?page=1&limit=10`
- **Response Success (200):** Array pergerakan stok diiringi _pagination_.

```json
{
  "success": true,
  "message": "Histori pergerakan stok berhasil diambil",
  "data": [
    {
      "id": "uuid-stock-log-1",
      "quantity": 50,
      "type": "adjustment",
      "note": "Restock beans dari supplier",
      "created_at": "2026-04-14T08:00:00Z",
      "updated_at": "2026-04-14T08:00:00Z"
    },
    {
      "id": "uuid-stock-log-2",
      "quantity": -2,
      "type": "sale",
      "note": "Dipesan pada nota transaksi TX-XYZ123",
      "created_at": "2026-04-14T15:30:00Z",
      "updated_at": "2026-04-14T15:30:00Z"
    }
  ],
  "meta": {
    "current_page": 1,
    "total_pages": 4,
    "total_items": 38,
    "per_page": 10
  }
}
```

- **Response Error:** `404 Not Found` (Produk tidak ditemukan).

---

### E. Tables (Owner)

#### 1. List Meja

- **Method & Path:** `GET /api/v1/owner/tables`
- **Akses:** Owner
- **Response Success (200):**

```json
{
  "success": true,
  "message": "Data meja berhasil diambil",
  "data": [
    {
      "id": "uuid-table-1",
      "name": "Meja 01",
      "status": "available",
      "created_at": "2026-04-14T08:00:00Z",
      "updated_at": "2026-04-14T08:00:00Z"
    }
  ]
}
```

#### 2. Create Meja

- **Method & Path:** `POST /api/v1/owner/tables`
- **Akses:** Owner
- **Request Body:**

```json
{
  "name": "Meja 02",
  "status": "available"
}
```

- **Response Success (201):**

```json
{
  "success": true,
  "message": "Meja berhasil ditambahkan",
  "data": {
    "id": "uuid-table-2",
    "name": "Meja 02",
    "status": "available",
    "created_at": "2026-04-14T08:15:00Z",
    "updated_at": "2026-04-14T08:15:00Z"
  }
}
```

- **Response Error:** `422 Unprocessable Entity` (Jika nama meja kosong).

#### 3. Update Meja

- **Method & Path:** `PUT /api/v1/owner/tables/:id`
- **Akses:** Owner
- **Request Body:** Sama seperti _Create_, mendukung pembaharuan _partial_.
- **Response Success (200):**

```json
{
  "success": true,
  "message": "Meja berhasil diupdate",
  "data": {
    "id": "uuid-table-2",
    "name": "Meja VIP",
    "status": "available",
    "created_at": "2026-04-14T08:15:00Z",
    "updated_at": "2026-04-14T08:20:00Z"
  }
}
```

- **Response Error:** `404 Not Found` (Meja tidak ditemukan) atau `422 Unprocessable Entity` (Validasi error).

#### 4. Delete Meja

- **Method & Path:** `DELETE /api/v1/owner/tables/:id`
- **Akses:** Owner
- **Response Success (200):**

```json
{
  "success": true,
  "message": "Meja berhasil dihapus"
}
```

- **Response Error:** `404 Not Found` (Jika meja tidak ada) atau `400 Bad Request` (Bila meja masih terkait dengan data transaksi).

---

### F. Cashier Management (Owner)

#### 1. List User Kasir

- **Method & Path:** `GET /api/v1/owner/cashiers`
- **Akses:** Owner
- **Query Params:** `?search=siti&page=1&limit=10`
- **Response Success (200):** Format pagination list kasir tunggal.

```json
{
  "success": true,
  "message": "Data kasir berhasil diambil",
  "data": [
    {
      "id": "uuid-user-1",
      "name": "Siti Kasir",
      "email": "siti@coffee.com",
      "role": "cashier",
      "is_active": true,
      "created_at": "2026-04-14T08:00:00Z"
    }
  ],
  "meta": {
    "current_page": 1,
    "total_pages": 1,
    "total_items": 1,
    "per_page": 10
  }
}
```

#### 2. Buat Akun Kasir Baru

- **Method & Path:** `POST /api/v1/owner/cashiers`
- **Akses:** Owner
- **Request Body:**

```json
{
  "name": "Siti Kasir",
  "email": "siti@coffee.com",
  "password": "initpasswordbaru"
}
```

_(Catatan: Field `role` otomatis ter-assign sebagai "cashier", dan `is_active` default true)._

- **Response Success (201):** Object user baru (tanpa mencantumkan password).
- **Response Error:** `422 Unprocessable Entity` (Contoh: jika email duplikat atau password kurang panjang).

#### 3. Update Data Profil Kasir

- **Method & Path:** `PUT /api/v1/owner/cashiers/:id`
- **Akses:** Owner
- **Request Body:**

```json
{
  "name": "Siti Rahmawati",
  "email": "siti.rahma@coffee.com"
}
```

_(Catatan: Digunakan ekslusif untuk update data teks sederhana diri tanpa merubah password dan status)._

- **Response Success (200):** Object detail user yang diperbarui.
- **Response Error:** `404 Not Found` (Kasir tidak ditemukan) atau `422 Unprocessable Entity` (Email bentrok).

#### 4. Aktif/Non-aktifkan Kasir (Toggle Status)

- **Method & Path:** `PATCH /api/v1/owner/cashiers/:id/toggle-status`
- **Akses:** Owner
- **Response Success (200):** API ini mengubah kondisi flag `is_active` (`true` -> `false` dan sebaliknya).

```json
{
  "success": true,
  "message": "Status kasir berhasil diubah",
  "data": {
    "id": "uuid-user-1",
    "is_active": false
  }
}
```

- **Response Error:** `404 Not Found` (Kasir invalid).

---

### G. Promos

#### 1. CRUD Promo

- **GET** `/api/v1/promos` (Akses: Owner, Cashier - Cashier hanya lihat yang aktif)
- **POST** `/api/v1/promos` (Akses: Owner)

```json
{
  "name": "Diskon Akhir Tahun 15%",
  "promo_type": "percentage",
  "amount": 15,
  "is_active": true
}
```

- **PUT** `/api/v1/promos/:id` (Akses: Owner)
- **DELETE** `/api/v1/promos/:id` (Akses: Owner)

---

### H. Reports (Owner)

#### 1. Total Pendapatan (Revenue)

- **Method & Path:** `GET /api/v1/owner/reports/revenue`
- **Akses:** Owner
- **Query Params:** `?start_date=2026-04-01&end_date=2026-04-30`
- **Response Success (200):**

```json
{
  "success": true,
  "message": "Data revenue berhasil diambil",
  "data": {
    "total_revenue": 15000000,
    "total_transactions": 350,
    "start_date": "2026-04-01",
    "end_date": "2026-04-30"
  }
}
```

#### 2. Produk Terlaris (Top Products)

- **Method & Path:** `GET /api/v1/owner/reports/top-products`
- **Akses:** Owner
- **Query Params:** `?limit=5&start_date=2026-04-01&end_date=2026-04-30`
- **Response Success (200):**

```json
{
  "success": true,
  "message": "Data produk terlaris berhasil diambil",
  "data": [
    {
      "product_id": "uuid-prod-1",
      "product_name": "Caramel Macchiato",
      "total_qty_sold": 120,
      "total_revenue": 3000000
    }
  ]
}
```

#### 3. Rekap Kasir (Cashier Summary)

- **Method & Path:** `GET /api/v1/owner/reports/cashier-summary`
- **Akses:** Owner
- **Query Params:** `?start_date=2026-04-01&end_date=2026-04-30`
- **Response Success (200):**

```json
{
  "success": true,
  "message": "Data performa kasir berhasil diambil",
  "data": [
    {
      "cashier_id": "uuid-user-1",
      "cashier_name": "Siti Kasir",
      "total_shifts_handled": 14,
      "total_revenue_generated": 7500000
    }
  ]
}
```

#### 4. Export Laporan Transaksi (CSV)

- **Method & Path:** `GET /api/v1/owner/reports/export`
- **Akses:** Owner
- **Query Params:** `?format=csv&start_date=2026-04-01&end_date=2026-04-30`
- **Response Success (200):**
  Stream file dataset raw CSV (`Content-Type: text/csv` dengan Header `Content-Disposition: attachment; filename="report-20260401-20260430.csv"`).
- **Response Error:** `400 Bad Request` (Jika struktur parameter filter tanggal _missing_ atau kurang valid).

---

### I. Shifts (Cashier)
#### 1. Buka Shift (Open)
- **Method & Path:** `POST /api/v1/cashier/shifts/open`
- **Akses:** Cashier
- **Request Body:**
```json
{
  "opening_cash": 15000000
}
```
*(Catatan: Nominal merepresentasikan pecahan sen/Rupiah utuh, yaitu Rp 150.000).*
- **Response Success (201):**
```json
{
  "success": true,
  "message": "Shift berhasil dibuka",
  "data": {
    "id": "uuid-shift-1",
    "cashier_id": "uuid-user-1",
    "opening_cash": 15000000,
    "status": "open",
    "start_time": "2026-04-14T08:00:00Z"
  }
}
```
- **Response Error:** `400 Bad Request` (Jika kasir saat ini sudah memiliki shift aktif / belum ditutup).

#### 2. Tutup Shift (Close)
- **Method & Path:** `POST /api/v1/cashier/shifts/close`
- **Akses:** Cashier
- **Request Body:**
```json
{
  "closing_cash": 125000000
}
```
*(Catatan: Input berupa kas aktual yang ada di laci saat shift akan ditutup).*
- **Response Success (200):** Sistem akan mengakumulasi `total_sales` transaksi sepanjang shift lalu menyimpannya.
```json
{
  "success": true,
  "message": "Shift berhasil ditutup",
  "data": {
    "id": "uuid-shift-1",
    "opening_cash": 15000000,
    "closing_cash": 125000000,
    "total_sales": 110000000,
    "status": "closed",
    "start_time": "2026-04-14T08:00:00Z",
    "end_time": "2026-04-14T17:00:00Z"
  }
}
```
- **Response Error:** `400 Bad Request` (Jika kasir ini tidak punya shift `open` yang bisa ditutup).

#### 3. Shift Aktif (Saat Ini)
- **Method & Path:** `GET /api/v1/cashier/shifts/current`
- **Akses:** Cashier
- **Response Success (200):** Menghasilkan data berformat sama persis seperti sesi *Open Shift* (Status: `open`).
- **Response Error:** `404 Not Found` (Jika kasir saat ini tidak memiliki shift operasional yang aktif).

---

### J. Orders (Cashier)

#### 1. Buat Draft Transaksi Baru
- **Method & Path:** `POST /api/v1/cashier/orders`
- **Akses:** Cashier (Shift harus berstatus `open`)
- **Request Body:**
```json
{
  "customer_name": "Anto",
  "table_id": "uuid-table-1"
}
```
*(Catatan: `table_id` bersifat opsional/nullable jika pelanggan *takeaway*).*
- **Response Success (201):**
```json
{
  "success": true,
  "message": "Draft transaksi berhasil dibuat",
  "data": {
    "id": "uuid-order-1",
    "customer_name": "Anto",
    "table_id": "uuid-table-1",
    "status": "pending",
    "total_amount": 0
  }
}
```

#### 2. Detail Order / Keranjang
- **Method & Path:** `GET /api/v1/cashier/orders/:id`
- **Akses:** Cashier
- **Response Success (200):** Menghasilkan rincian *list items*, subtotal, info promo yang diaplikasikan, dan `total_amount` real-time.

#### 3. Tambah Item ke Order
- **Method & Path:** `POST /api/v1/cashier/orders/:id/items`
- **Akses:** Cashier
- **Request Body:**
```json
{
  "product_id": "uuid-prod-123",
  "quantity": 2
}
```
- **Response Success (201):** Object rincian item yang baru saja dimasukkan. Server secara back-end mengkalkulasi list item untuk membentuk *total amount* order yang baru.
- **Response Error:** `422 Unprocessable Entity` (Jika produk tidak ditemukan / habis stok / kuantitas minus).

#### 4. Update Qty Item
- **Method & Path:** `PUT /api/v1/cashier/orders/:id/items/:item_id`
- **Akses:** Cashier
- **Request Body:**
```json
{
  "quantity": 3
}
```
- **Response Success (200):** Menghasilkan *summary* item order setelah kuantitas di-adjust secara langsung.
- **Response Error:** `400 Bad Request` (Jika order telah di checkout/dibayar sebelumnya).

#### 5. Hapus Item dari Order
- **Method & Path:** `DELETE /api/v1/cashier/orders/:id/items/:item_id`
- **Akses:** Cashier
- **Response Success (200):** Terhapus dari list dan rekapitulasi poin nilai total kembali disesuaikan.

#### 6. Terapkan/Input Promo Code
- **Method & Path:** `POST /api/v1/cashier/orders/:id/promo`
- **Akses:** Cashier
- **Request Body:**
```json
{
  "promo_id": "uuid-promo-xyz"
}
```
- **Response Success (200):** Mendapatkan perhitungan `total_amount` final terkini setelah diskon diberikan.
- **Response Error:** `400 Bad Request` (Sesuai business logic, jika sebuah transaksi kedapatan sudah memakai metode promo sebelumnya).

#### 7. Gagal / Tarik Kembali Promo
- **Method & Path:** `DELETE /api/v1/cashier/orders/:id/promo`
- **Akses:** Cashier
- **Response Success (200):** Mengembalikan rekalkulasi `total_amount` pada draft order menjadi harga kotor (tanpa potongan sama sekali).

#### 8. Riwayat Transaksi Shift Berjalan
- **Method & Path:** `GET /api/v1/cashier/orders/history`
- **Akses:** Cashier
- **Query Params:** `?page=1&limit=10`
- **Response Success (200):** Menampilkan semua status *orders* yang terekam pada shift terbuka kasir tersebut, menggunakan format standar paginasi `meta`.

---

### K. Checkout & Payments
#### 1. Checkout Order (Generate SNAP)
- **Method & Path:** `POST /api/v1/cashier/orders/:id/checkout`
- **Akses:** Cashier
- **Request Body:**
```json
{
  "amount_tendered": 5000000 
}
```
*(Catatan: Optional untuk mendata kas yang diberikan oleh pelanggan saat pembayaran tunai seandainya diperlukan pencatatan jumlah uang kembalian. Kosongkan nilai jika pembayaran via transfer/Qris).*
- **Response Success (200):** Keranjang transaksi otomatis diset berstatus `pending_payment` dan keranjang dikunci. Sistem menerbitkan resi/link *SNAP Midtrans*.
```json
{
  "success": true,
  "message": "Checkout sukses, silakan melanjutkan proses pembayaran",
  "data": {
    "order_id": "uuid-order-1",
    "final_amount": 4250000,
    "payment_status": "pending_payment",
    "midtrans_token": "snap-token-xyz123",
    "midtrans_redirect_url": "https://app.midtrans.com/snap/v2/vtweb/snap-token-xyz123"
  }
}
```

#### 2. Cek Status Pembayaran Order
- **Method & Path:** `GET /api/v1/cashier/orders/:id/payment-status`
- **Akses:** Cashier
- **Response Success (200):** Rutinitas endpoint kecil ini dikhususkan agar Front-end POS bisa melakukan interval UI *polling* menunggu pembayaran pelanggan selesai.
```json
{
  "success": true,
  "data": {
    "order_id": "uuid-order-1",
    "payment_status": "paid" 
  }
}
```

#### 3. Midtrans Webhook (Server-to-Server)
- **Method & Path:** `POST /api/v1/webhooks/midtrans`
- **Akses:** Public / Midtrans IP Addresses
- **Request Body:** Sesuai dokumentasi baku Core/Snap HTTP *Payload Notification* Midtrans (berupa JSON detail seperti `order_id`, `transaction_status`, `signature_key`, dan lainnya).
- **Response Success (200):** Server mencetak log respons. Apabila field `transaction_status` adalah `settlement`, maka server secara asinkron akan otomatis men-*trigger* perintah pemotongan kuantitas ke semua produk bersangkutan di tabel `stocks` berjeniskan *sale*.
- **Response Error:** `400/401` Jika kalkulasi *Signature Key* verifikasi hash ditolak (mengindikasi indikasi pemalsuan).
