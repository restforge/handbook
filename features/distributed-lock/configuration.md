# Konfigurasi Backend (Backend Configuration)

> Prasyarat, variable `.env`, dan strategi acquisition untuk per-record lock.

Kembali ke [ikhtisar Distributed Lock](README.md).

---

## Prasyarat (Prerequisites)

| Komponen | Keterangan |
|----------|------------|
| Redis Server | Redis harus berjalan dan dapat diakses |
| Cluster Mode | Fitur ini dirancang untuk multi-worker. Pada single worker, lock tetap berfungsi namun manfaatnya minimal |

### Independensi dari Cache Layer

Distributed lock dan cache layer adalah dua fitur yang berdiri sendiri. Keduanya menggunakan Redis sebagai backend, tetapi tidak saling bergantung.

| Konfigurasi | Perilaku |
|-------------|----------|
| `CACHE_ENABLED=true`, `LOCK_DISTRIBUTED_ENABLED=true` | Cache mempercepat response, lock melindungi concurrent write per-record |
| `CACHE_ENABLED=false`, `LOCK_DISTRIBUTED_ENABLED=true` | Hanya serialisasi operasi update/delete per-record |
| `CACHE_ENABLED=true`, `LOCK_DISTRIBUTED_ENABLED=false` | Hanya cache, tanpa proteksi concurrent write |
| `CACHE_ENABLED=false`, `LOCK_DISTRIBUTED_ENABLED=false` | Perilaku standar tanpa cache dan tanpa lock |

Cache invalidation berjalan independen setelah setiap operasi WRITE, tidak terkait dengan mekanisme lock.

### Validasi Saat Startup (Startup Validation)

Apabila `CACHE_ENABLED=true` atau `LOCK_DISTRIBUTED_ENABLED=true`, server memvalidasi koneksi Redis saat startup.

| Kondisi | Perilaku |
|---------|----------|
| Redis tidak dapat diakses | Server menolak untuk jalan dengan pesan error koneksi Redis beserta host/port yang digunakan |
| Redis terhubung | Server berjalan normal dengan fitur Redis aktif |

---

## Operasi yang Dilindungi (Protected Operations)

Lock bersifat per-record, sehingga operasi pada record yang berbeda tetap berjalan paralel.

| Operasi | Lock | Alasan |
|---------|------|--------|
| `add` (INSERT) | Tanpa lock | INSERT membuat row baru, sehingga tidak ada konflik concurrent pada record yang sama |
| `update` | WRITE lock per-record | Melindungi dari concurrent update pada record yang sama |
| `delete` | WRITE lock per-record | Melindungi dari concurrent update+delete pada record yang sama |
| `datatables` (READ) | Tanpa lock | Operasi READ, database MVCC menjamin read consistency |

---

## Variable `.env`

Tambahkan variable berikut di file `.env` (root) atau `config/*.env`:

```env
# ===========================================
# DISTRIBUTED LOCK CONFIGURATION (Redis-based)
# ===========================================

# Mengaktifkan distributed lock (default: false)
LOCK_DISTRIBUTED_ENABLED=true

# Strategi acquire lock (default: retry)
#   retry  = Exponential backoff sampai lock didapat atau timeout
#   reject = Satu percobaan, langsung error jika record sedang di-lock
LOCK_DISTRIBUTED_STRATEGY=retry

# TTL lock per-record dalam detik (default: 10)
LOCK_DISTRIBUTED_TTL=10

# Jumlah retry saat acquire gagal (default: 3, hanya untuk strategy=retry)
LOCK_DISTRIBUTED_RETRY=3

# Base delay antar retry dalam milliseconds (default: 100, hanya untuk strategy=retry)
# Actual delay menggunakan exponential backoff: DELAY * 2^attempt (100ms, 200ms, 400ms)
LOCK_DISTRIBUTED_RETRY_DELAY=100

# Batas maksimum TTL resource lock dalam detik (default: 300)
LOCK_RESOURCE_MAX_TTL=300
```

### Penjelasan Parameter

| Parameter | Default | Satuan | Keterangan |
|-----------|---------|--------|------------|
| `LOCK_DISTRIBUTED_ENABLED` | `false` | — | Mengaktifkan fitur distributed lock. Wajib `true` agar endpoint resource lock tersedia |
| `LOCK_DISTRIBUTED_STRATEGY` | `retry` | — | Strategi acquire: `retry` atau `reject` |
| `LOCK_DISTRIBUTED_TTL` | `10` | detik | Durasi maksimal lock per-record aktif sebelum expire otomatis |
| `LOCK_DISTRIBUTED_RETRY` | `3` | kali | Jumlah percobaan acquire sebelum timeout. Hanya berlaku untuk `strategy=retry` |
| `LOCK_DISTRIBUTED_RETRY_DELAY` | `100` | ms | Base delay antar retry (exponential backoff). Hanya berlaku untuk `strategy=retry` |
| `LOCK_RESOURCE_MAX_TTL` | `300` | detik | Batas atas TTL yang dapat diminta saat acquire resource lock. Request yang melebihi batas ini ditolak |

---

## Strategi Acquisition (Acquisition Strategy)

Parameter `LOCK_DISTRIBUTED_STRATEGY` menentukan perilaku saat record sedang di-lock oleh worker lain. Pemilihan nilai ini berdampak langsung pada pengalaman client, sehingga perlu disesuaikan dengan pola pemakaian API.

### Strategy: `retry` (Default)

Request menunggu dengan exponential backoff sampai lock tersedia atau timeout tercapai.

```
Concurrent UPDATE pada record 'abc' (strategy=retry):
  Worker A ──→ Lock(abc) acquired ──→ UPDATE ──→ Release
  Worker B ──→ wait 100ms ──→ wait 200ms ──→ Lock(abc) acquired ──→ UPDATE ──→ Release
  Worker C ──→ wait 100ms ──→ wait 200ms ──→ wait 400ms ──→ TIMEOUT ERROR
```

- Request menunggu giliran, sehingga client tidak perlu menangani retry sendiri
- Jumlah percobaan ditentukan `LOCK_DISTRIBUTED_RETRY`, delay antar percobaan `LOCK_DISTRIBUTED_RETRY_DELAY * 2^attempt`
- Apabila semua percobaan gagal, operasi ditolak dengan error lock timeout

### Strategy: `reject`

Request langsung ditolak apabila lock tidak tersedia, yaitu satu percobaan tanpa menunggu.

```
Concurrent UPDATE pada record 'abc' (strategy=reject):
  Worker A ──→ Lock(abc) SET NX → OK ──→ UPDATE ──→ Release
  Worker B ──→ Lock(abc) SET NX → FAIL ──→ IMMEDIATE ERROR
```

- Hanya satu request yang dieksekusi per-record pada satu waktu, request lain langsung mendapat error
- Parameter `LOCK_DISTRIBUTED_RETRY` dan `LOCK_DISTRIBUTED_RETRY_DELAY` diabaikan

### Perbandingan (Comparison)

| Aspek | `retry` | `reject` |
|-------|---------|----------|
| Perilaku saat lock tidak tersedia | Menunggu dan coba lagi | Langsung error |
| Jumlah request yang berhasil | Semua (jika tidak timeout) | Hanya 1 per-record per-waktu |
| Latensi request kedua | Bertambah (waktu tunggu) | Tidak ada (langsung error) |
| `RETRY` dan `RETRY_DELAY` | Digunakan | Diabaikan |
| Client handling | Tidak perlu retry logic | Perlu retry logic di client |

### Pemilihan Strategi (Strategy Selection)

| Kondisi | Rekomendasi |
|---------|-------------|
| API digunakan oleh frontend/UI | `retry`, karena user tidak perlu menekan tombol ulang |
| API digunakan oleh sistem otomatis yang punya retry logic | `reject`, karena sistem menangani retry sendiri |
| Traffic tinggi dengan banyak concurrent write pada record yang sama | `retry`, untuk mengurangi error rate |
| Memerlukan response time yang konsisten | `reject`, karena tidak ada variasi latensi dari waiting |
| Default atau tidak yakin | `retry`, karena backward compatible dan lebih toleran |

---

Lihat juga: [Resource Lock API](resource-lock.md) untuk penguncian manual, dan [Troubleshooting](troubleshooting.md) untuk format lock key dan pemecahan masalah.
