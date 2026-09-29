Berdasarkan karakteristik **Offline-First POS**, berikut adalah rancangan arsitektur schema database yang ideal untuk **`tokobazar-db`** di Cloudflare D1.

Rancangan ini menerapkan prinsip **Hybrid Denormalization**: menggunakan *Foreign Keys* (`INTEGER`) untuk relasi data & analitik, serta menyimpan *Snapshot Strings* (`TEXT`) untuk menjaga independensi saat perangkat sedang *offline*.

---

### Schema SQL Ideal (`schema.sql`)

```sql
-- 1. TABEL USERS (Kasir & Admin)
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT UNIQUE NOT NULL,
    password TEXT NOT NULL,                     -- Disimpan sebagai hash (bcrypt/argon2)
    name TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'KASIR',          -- 'ADMIN', 'KASIR', 'MANAGER'
    is_active INTEGER NOT NULL DEFAULT 1,       -- Soft delete (1 = Aktif, 0 = Nonaktif)
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 2. TABEL PRODUCTS (Katalog Barang)
CREATE TABLE products (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    barcode TEXT UNIQUE,                        -- Boleh NULL jika produk tidak punya barcode
    name TEXT NOT NULL,
    price REAL NOT NULL CHECK (price >= 0),
    stock INTEGER NOT NULL DEFAULT 0,            -- Manajemen stok lokal/central
    is_active INTEGER NOT NULL DEFAULT 1,       -- Soft delete untuk barang discontinue
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 3. TABEL TRANSACTIONS (Header Transaksi)
CREATE TABLE transactions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    invoice_no TEXT UNIQUE NOT NULL,            -- Format: INV/YYYYMMDD/XXXX
    user_id INTEGER,                            -- Relasi utama ke kasir
    cashier_name TEXT NOT NULL,                  -- Snapshot nama kasir untuk struk offline
    subtotal_amount REAL NOT NULL,
    discount_amount REAL DEFAULT 0,
    tax_type TEXT DEFAULT 'rp',                 -- 'rp' (Rupiah) atau 'percent' (%)
    tax_value REAL DEFAULT 0,
    tax_amount REAL DEFAULT 0,
    admin_fee_amount REAL DEFAULT 0,
    total_amount REAL NOT NULL,
    paid_amount REAL NOT NULL,
    change_amount REAL NOT NULL,
    payment_method TEXT NOT NULL DEFAULT 'TUNAI',-- 'TUNAI', 'QRIS', 'DEBIT', dll.
    notes TEXT DEFAULT '',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET NULL
);

-- 4. TABEL TRANSACTION_ITEMS (Detail Item Transaksi)
CREATE TABLE transaction_items (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    invoice_no TEXT NOT NULL,                   -- Relasi ke header transaksi
    product_id INTEGER,                         -- Relasi ke katalog
    product_name TEXT NOT NULL,                 -- Snapshot nama barang saat dibeli
    price REAL NOT NULL,                        -- Snapshot harga barang saat dibeli
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    subtotal REAL NOT NULL,                     -- (price * quantity)
    FOREIGN KEY (invoice_no) REFERENCES transactions(invoice_no) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE SET NULL
);

-- 5. TABEL EXPENSES (Pengeluaran Operasional Store)
CREATE TABLE expenses (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER,                            -- Relasi ke kasir/admin yang mencatat
    cashier_name TEXT NOT NULL,                  -- Snapshot nama pencatat
    category TEXT NOT NULL,                     -- 'OPERASIONAL', 'STOK', 'LAINNYA'
    amount REAL NOT NULL CHECK (amount > 0),
    notes TEXT DEFAULT '',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET NULL
);

```

---

### Indexing untuk Performa Cloudflare D1

Karena Cloudflare D1 akan sering melayani query pencarian dan pelaporan, buat *Index* pada kolom-kolom yang sering di-filter:

```sql
-- Mempercepat pencarian histori transaksi berdasarkan tanggal & nomor invoice
CREATE INDEX idx_transactions_created_at ON transactions(created_at);
CREATE INDEX idx_transactions_user_id ON transactions(user_id);

-- Mempercepat lookup item pada nota
CREATE INDEX idx_transaction_items_invoice ON transaction_items(invoice_no);

-- Mempercepat pencarian produk berdasarkan Barcode saat scan barang
CREATE INDEX idx_products_barcode ON products(barcode);

```

---

### Keunggulan Arsitektur Ini untuk Offline-First POS:

1. **Aturan Soft Delete (`is_active`):**
Produk atau User tidak pernah dihapus secara permanen (`DELETE FROM`). Cukup ubah `is_active = 0`. Ini mencegah pecahnya referensi relasi *Foreign Key* saat sinkronisasi data.
2. **Fleksibilitas Item Custom:**
`product_id` di `transaction_items` boleh `NULL`. Jika ada produk rakitan/non-katalog yang dijual manual oleh kasir, sistem tidak akan *error*.
3. **Pencatatan Pajak & Biaya Tambahan yang Jelas:**
Tipe data `tax_value` menggunakan `REAL` (bukan `TEXT` seperti pada tabel lama Anda) agar perhitungan matematika langsung akurat tanpa konversi tipe data.
4. **Keamanan Riwayat (Audit Trail):**
Perubahan harga barang di tabel `products` tidak akan merusak hitungan omzet historis di `transaction_items` karena harga saat transaksi sudah terkunci (*snapshot*).
