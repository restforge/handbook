# Distributed Lock

> Koordinasi operasi WRITE antar worker pada cluster mode menggunakan Redis sebagai backend lock.

Distributed Lock mencegah race condition ketika beberapa worker memproses operasi WRITE pada record yang sama secara bersamaan. Fitur ini terdiri dari tiga lapisan yang berdiri sendiri:

- **Per-record lock** — otomatis pada operasi `update` dan `delete`, men-serialize WRITE pada record yang sama.
- **Resource lock** — manual melalui REST API untuk mengunci satu atau beberapa record sekaligus sebelum serangkaian operasi.
- **Frontend lock** — membungkus operasi Edit dan Delete pada halaman CRUD hasil generate dengan acquire/release otomatis.

---

## Referensi Cepat (Quick Reference)

| Properti | Nilai |
|----------|-------|
| Database | PostgreSQL, MySQL, Oracle, SQLite |
| Dependency | Redis |
| Lock otomatis | `update`, `delete` (per-record WRITE lock) |
| Lock manual | `POST /lock/acquire`, `POST /lock/release`, `POST /lock/status` |
| Strategi | `retry` (default) atau `reject` |
| Konfigurasi backend | `LOCK_DISTRIBUTED_ENABLED`, `LOCK_DISTRIBUTED_STRATEGY`, `LOCK_DISTRIBUTED_TTL`, `LOCK_RESOURCE_MAX_TTL` |
| Konfigurasi frontend | `features.enableDistributedLock`, `features.distributedLock` |

---

## Setup Singkat (Quick Setup)

Tiga langkah untuk mengaktifkan distributed lock dasar (per-record lock di backend dan lock pada halaman frontend).

### 1. Pastikan Redis aktif

Distributed lock memerlukan Redis. Server akan menolak jalan apabila lock diaktifkan tetapi Redis tidak dapat diakses.

### 2. Aktifkan di backend

Tambahkan variable berikut di file `.env` (root) atau `config/*.env`, lalu restart server:

```env
LOCK_DISTRIBUTED_ENABLED=true
LOCK_DISTRIBUTED_TTL=10
LOCK_RESOURCE_MAX_TTL=300
```

Setelah ini, operasi `update` dan `delete` otomatis terlindungi per-record. Detail seluruh parameter dan pemilihan strategi ada di [Konfigurasi Backend](configuration.md).

### 3. Aktifkan di frontend

Tambahkan property berikut di blok `features` pada payload halaman, lalu generate ulang halaman tersebut:

```json
"features": {
  "enableDistributedLock": true,
  "distributedLock": { "ttl": 300 }
}
```

Setelah generate ulang, halaman CRUD akan mengunci record saat Edit atau Delete. Detail property dan konfigurasi halaman composite ada di [Konfigurasi Frontend](frontend.md).

---

## Dokumentasi Lengkap (Full Documentation)

| Dokumen | Isi |
|---------|-----|
| [Konfigurasi Backend](configuration.md) | Prasyarat, seluruh variable `.env`, strategi `retry` vs `reject`, operasi yang dilindungi |
| [Konfigurasi Frontend](frontend.md) | Property payload UDF, dua lapisan lock pada halaman composite |
| [Resource Lock API](resource-lock.md) | Penguncian manual banyak record, referensi endpoint, pola penggunaan dengan composite |
| [Troubleshooting](troubleshooting.md) | Pemecahan masalah, format lock key, dampak performa, limitasi |
