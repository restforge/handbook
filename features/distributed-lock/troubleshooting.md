# Troubleshooting dan Catatan Lanjutan

> Pemecahan masalah, format lock key, dampak performa, dan limitasi yang disadari.

Kembali ke [ikhtisar Distributed Lock](README.md).

---

## Format Lock Key

Bagian ini relevan terutama untuk debugging atau inspeksi manual melalui Redis CLI. Pemakaian normal tidak memerlukan pengetahuan format ini.

Per-record lock (otomatis) menggunakan format:

```
rf:lock:{project}:{endpoint}:{recordId}:{lockType}

Contoh:
  rf:lock:inventory:supplier:550e8400-e29b-41d4-a716-446655440000:write
```

Resource lock (manual) menggunakan format:

```
rf:rlock:{resource}:{key}:{value}

Contoh:
  rf:rlock:item_products:item_product_id:101
```

Inspeksi manual via Redis CLI:

```bash
# Lihat semua lock yang aktif
redis-cli KEYS "rf:lock:*"

# Hapus lock tertentu (hanya dalam keadaan darurat)
redis-cli DEL "rf:lock:inventory:supplier:550e8400-...:write"
```

Penghapusan lock secara manual tidak disarankan kecuali dalam keadaan darurat, karena lock memiliki TTL yang akan expire otomatis.

---

## Pemecahan Masalah (Troubleshooting)

### Server menolak jalan: "REDIS CONNECTION FAILED"

Server exit saat startup karena `LOCK_DISTRIBUTED_ENABLED=true` (atau `CACHE_ENABLED=true`) tetapi Redis tidak dapat diakses.

| Penyebab | Langkah |
|----------|---------|
| Redis belum running | Jalankan Redis server |
| Host/port salah | Periksa `REDIS_HOST` dan `REDIS_PORT` di file `.env` |
| Password salah | Periksa `REDIS_PASSWORD` di file `.env` |

### Operasi WRITE gagal dengan error lock

Response error `"Failed to acquire lock for update/delete operation..."`. Penyebab dapat dibedakan dari log event (`write_lock_timeout`, `write_lock_rejected`, `write_lock_error`).

| Penyebab | Solusi |
|----------|--------|
| Redis down / tidak terhubung | Periksa status Redis |
| Concurrent write pada record yang sama | Operasi lain sedang memproses record yang sama, coba lagi |
| Lock TTL terlalu rendah | Naikkan `LOCK_DISTRIBUTED_TTL` |
| Database query lambat | Optimasi query atau naikkan TTL |
| Retry terlalu sedikit (`retry`) | Naikkan `LOCK_DISTRIBUTED_RETRY` |
| Error rate tinggi (`reject`) | Pertimbangkan beralih ke `strategy=retry` |

### Endpoint `/lock/acquire` return HTTP 404

Resource Lock API tidak terdaftar. Pastikan `LOCK_DISTRIBUTED_ENABLED=true`, Redis aktif, dan periksa log startup `Resource Lock API registered at /api/{project}/lock`.

### Acquire gagal: "ttl exceeds maximum" (HTTP 400)

Nilai `ttl` dalam request melebihi `LOCK_RESOURCE_MAX_TTL`. Turunkan TTL request, atau naikkan batas server (dalam satuan detik).

### Release return `released_keys: 0`

| Penyebab | Solusi |
|----------|--------|
| `lock_token` salah | Gunakan `lock_token` yang sama dari response acquire |
| Lock sudah expire | Tidak memerlukan tindakan, lock sudah otomatis dilepaskan |
| Lock sudah di-release sebelumnya | Tidak memerlukan tindakan, kondisi ini aman |

### Lock conflict frontend tidak muncul (dua user bisa edit bersamaan)

| Langkah | Pemeriksaan |
|---------|-------------|
| 1 | Pastikan `enableDistributedLock: true` ada di blok `features` payload |
| 2 | Pastikan frontend sudah di-generate ulang setelah menambahkan konfigurasi lock |
| 3 | Pastikan kedua browser login dengan user yang berbeda |

### Redis down saat server berjalan

| Operasi | Perilaku |
|---------|----------|
| READ (datatables, get, lookup) | Tetap berfungsi (tidak menggunakan lock) |
| INSERT (add) | Tetap berfungsi (tidak menggunakan lock) |
| UPDATE, DELETE | Ditolak (error) untuk mencegah race condition |

Redis client melakukan auto-reconnect. Begitu Redis kembali online, semua operasi pulih tanpa restart server.

---

## Dampak Terhadap Performa (Performance Impact)

| Aspek | Dampak |
|-------|--------|
| Overhead per request (update/delete) | Sekitar 1-2ms (roundtrip ke Redis) |
| Operasi INSERT dan READ | Tidak ada overhead (tanpa lock) |
| Per-record serialization | Menambah latensi hanya pada concurrent write ke record yang sama (by design) |
| Single worker mode | Overhead minimal, manfaat minimal |

### Rekomendasi

- Aktifkan distributed lock hanya apabila menggunakan cluster mode (multi-worker)
- Sesuaikan `LOCK_DISTRIBUTED_TTL` dengan durasi query database terlama. Apabila query memerlukan 5 detik, set TTL minimal 10 detik
- Monitor log `write_lock_timeout` dan `write_lock_waiting` untuk mendeteksi kontention pada record tertentu

---

## Limitasi yang Disadari (Known Limitations)

| Limitasi | Keterangan |
|----------|------------|
| `createComposite` tanpa lock otomatis | INSERT membuat record baru tanpa PK untuk di-lock. Gunakan [Resource Lock API](resource-lock.md) untuk skenario shared resource |
| Cross-endpoint | Per-record lock pada satu endpoint tidak melindungi skenario cross-endpoint (mis. stock-outbound vs stock-inbound mengakses stok yang sama). Gunakan database-level `SELECT ... FOR UPDATE` atau isolation level SERIALIZABLE |
