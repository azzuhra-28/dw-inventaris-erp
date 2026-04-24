# 🏭 Data Warehouse Inventaris ERP

> Proyek ETL (Extract, Transform, Load) dari MySQL ke PostgreSQL menggunakan Apache Pentaho (PDI/Spoon) dalam rangka membangun Data Warehouse sistem inventaris ERP dengan skema Star Schema.

---

## 📌 Deskripsi Proyek

Proyek ini merupakan tugas praktikum mata kuliah **Data Warehouse** di Program Studi Sains Data Terapan.

Tujuan proyek ini adalah memindahkan dan mentransformasi data dari database operasional (MySQL) ke dalam Data Warehouse (PostgreSQL) menggunakan pipeline ETL berbasis Pentaho Data Integration (PDI).

---

## 🛠️ Teknologi yang Digunakan

| Tool | Fungsi |
|------|--------|
| **MySQL** | Database sumber (OLTP) |
| **PostgreSQL** | Database tujuan (Data Warehouse) |
| **Pentaho PDI (Spoon)** | Tool ETL — membangun pipeline transformasi |
| **SQL** | Query untuk ekstraksi & pembuatan tabel |

---

## 🗂️ Arsitektur Star Schema

```
                    ┌─────────────────┐
                    │   dim_waktu     │
                    └────────┬────────┘
                             │
┌──────────────┐    ┌────────┴────────┐    ┌──────────────────────┐
│  dim_produk  │────│                 │────│    dim_pelanggan      │
└──────────────┘    │  fakta_         │    └──────────────────────┘
                    │  transaksi_     │
┌──────────────┐    │  inventaris     │    ┌──────────────────────┐
│  dim_gudang  │────│                 │────│    dim_pemasok        │
└──────────────┘    └────────┬────────┘    └──────────────────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
   ┌──────────────────┐       ┌─────────────────────┐
   │ dim_jenis_       │       │   dim_sumber_erp     │
   │ transaksi        │       └─────────────────────┘
   └──────────────────┘
```

---

## 📁 Struktur Repository

```
data-warehouse-etl-pentaho/
│
├── README.md
│
├── transformations/              ← File .ktr Pentaho
│   ├── dim_gudang.ktr
│   ├── dim_jenis_transaksi.ktr
│   ├── dim_pelanggan.ktr
│   ├── dim_pemasok.ktr
│   ├── dim_produk.ktr
│   ├── dim_sumber_erp.ktr
│   ├── dim_waktu.ktr
│   └── fakta_transaksi_inventaris.ktr
│
├── sql/
│   ├── mysql_schema.sql          ← DDL tabel staging & dimensi di MySQL
│   └── postgres_schema.sql       ← DDL tabel fakta di PostgreSQL
│
└── docs/
    └── screenshots/              ← Hasil tangkapan layar ETL
```

---

## 🗃️ Struktur Database

### MySQL (Sumber)

**Staging Table**
```sql
CREATE TABLE stg_transaksi (
    kode_transaksi      VARCHAR(50),
    tanggal             DATE,
    kode_produk         VARCHAR(50),
    kode_gudang         VARCHAR(50),
    kode_pemasok        VARCHAR(50),
    kode_pelanggan      VARCHAR(50),
    kode_jenis_transaksi VARCHAR(50),
    kode_sumber         VARCHAR(50),
    jumlah_masuk        INT,
    jumlah_keluar       INT,
    harga               DECIMAL(18,2)
);
```

**Tabel Dimensi (MySQL)**
- `dim_produk` — data produk (kode, nama, kategori, sub kategori, satuan)
- `dim_pelanggan` — data pelanggan (kode, nama, wilayah, jenis)
- `dim_pemasok` — data pemasok (kode, nama, wilayah)
- `dim_gudang` — data gudang (kode, nama, kota, wilayah, kapasitas)
- `dim_jenis_transaksi` — jenis transaksi (masuk/keluar/penyesuaian)
- `dim_sumber_erp` — modul ERP sumber data
- `dim_waktu` — dimensi waktu (tanggal, bulan, tahun, kuartal, hari)

### PostgreSQL (Tujuan)

**Tabel Fakta**
```sql
CREATE TABLE fakta_transaksi (
    id_fakta            SERIAL PRIMARY KEY,
    sk_waktu            INT,
    sk_produk           INT,
    sk_gudang           INT,
    sk_pemasok          INT,
    sk_pelanggan        INT,
    sk_jenis_transaksi  INT,
    sk_sumber           INT,
    jumlah_masuk        INT,
    jumlah_keluar       INT,
    nilai_masuk         NUMERIC(18,2),
    nilai_keluar        NUMERIC(18,2),
    biaya_total         NUMERIC(18,2),
    jumlah_penyesuaian  INT
);
```

---

## ⚙️ Pipeline ETL

Setiap file `.ktr` merepresentasikan satu alur transformasi dimensi:

| File | Deskripsi |
|------|-----------|
| `dim_produk.ktr` | Ekstrak kode produk unik → generate atribut → load ke dim_produk |
| `dim_pelanggan.ktr` | Ekstrak data pelanggan unik dari staging |
| `dim_pemasok.ktr` | Ekstrak data pemasok unik dari staging |
| `dim_gudang.ktr` | Ekstrak data gudang unik dari staging |
| `dim_jenis_transaksi.ktr` | Ekstrak jenis transaksi dari staging |
| `dim_sumber_erp.ktr` | Ekstrak modul ERP sumber dari staging |
| `dim_waktu.ktr` | Generate dimensi waktu dari kolom tanggal |
| `fakta_transaksi_inventaris.ktr` | Join semua surrogate key → load ke tabel fakta PostgreSQL |

### Alur ETL secara Umum:
```
MySQL (stg_transaksi)
        ↓  Table Input
   Filter / Distinct
        ↓  Modified JavaScript Value
   Generate Atribut
        ↓  Unique Rows
   Hapus Duplikat
        ↓  Table Output
PostgreSQL (dim_* / fakta_*)
```

---

## 🚀 Cara Menjalankan

1. **Pastikan MySQL dan PostgreSQL sudah berjalan**

2. **Import schema database:**
   ```bash
   # MySQL
   mysql -u root -p < sql/mysql_schema.sql

   # PostgreSQL
   psql -U postgres -f sql/postgres_schema.sql
   ```

3. **Buka Pentaho Spoon**, lalu buka file `.ktr` dari folder `transformations/`

4. **Jalankan transformasi dimensi terlebih dahulu** (urutan bebas):
   - `dim_produk.ktr`
   - `dim_pelanggan.ktr`
   - `dim_pemasok.ktr`
   - `dim_gudang.ktr`
   - `dim_jenis_transaksi.ktr`
   - `dim_sumber_erp.ktr`
   - `dim_waktu.ktr`

5. **Terakhir jalankan tabel fakta:**
   - `fakta_transaksi_inventaris.ktr`

> ⚠️ **Catatan:** Sesuaikan konfigurasi koneksi database (host, port, username, password) di setiap file `.ktr` sebelum dijalankan.

---

