# Konfigurasi Frontend (Frontend Configuration)

> Mengaktifkan distributed lock pada halaman CRUD hasil generate melalui property payload UDF.

Kembali ke [ikhtisar Distributed Lock](README.md).

---

## Behavior

Pada aplikasi frontend yang di-generate, distributed lock diaktifkan melalui blok `features` di payload UDF. Saat user mengklik Edit atau Delete, sistem melakukan acquire lock terlebih dahulu, dan melepaskannya saat panel form atau dialog konfirmasi ditutup.

Apabila lock gagal di-acquire (record sedang dikunci user lain), panel form atau dialog tidak dibuka dan pesan konflik ditampilkan beserta nama user yang sedang memproses record.

Konfigurasi frontend ini memerlukan backend yang sudah mengaktifkan distributed lock. Lihat [Konfigurasi Backend](configuration.md).

---

## Property Payload

```json
"features": {
  "enableDistributedLock": true,
  "distributedLock": {
    "ttl": 300,
    "infoMessage": "Distributed lock is active on this form. Do not leave the editor or delete confirmation open without action for more than {ttl}.",
    "detailLock": {
      "resource": "item_products",
      "keyField": "item_product_id"
    }
  }
}
```

| Property | Tipe | Default | Keterangan |
|----------|------|---------|------------|
| `enableDistributedLock` | boolean | `false` | Mengaktifkan lock pada operasi Edit dan Delete. Memerlukan Redis aktif dan `LOCK_DISTRIBUTED_ENABLED=true` di backend |
| `distributedLock.ttl` | number | `300` | Durasi maksimal lock dalam detik. Tidak boleh melebihi `LOCK_RESOURCE_MAX_TTL` di backend |
| `distributedLock.infoMessage` | string | `""` | Pesan informasi saat halaman dibuka. Placeholder `{ttl}` diganti otomatis dengan durasi terformat. Kosong berarti pesan tidak ditampilkan |
| `distributedLock.detailLock` | object | — | Lapisan lock kedua untuk halaman composite (master-detail). Opsional |
| `distributedLock.detailLock.resource` | string | — | Nama resource untuk lock detail items (contoh: `"item_products"`) |
| `distributedLock.detailLock.keyField` | string | — | Nama field di detail items yang menjadi lock key (contoh: `"item_product_id"`) |

---

## Dua Lapisan Lock pada Halaman Composite

Halaman non-composite (master sederhana) cukup menggunakan header lock tanpa `detailLock`. Halaman composite (master-detail) mendukung lapisan lock kedua melalui `detailLock`.

| Lapisan | Target | Timing Acquire | Timing Release |
|---------|--------|----------------|----------------|
| Header | Record utama (mis. `stock_inbound_id`) | Klik Edit | Panel ditutup |
| Detail Items | Product yang diinput (mis. `item_product_id`) | Klik Save | Setelah AJAX selesai |

Header lock mencegah concurrent edit pada record yang sama. Detail item lock mencegah concurrent stock allocation pada product yang sama di transaksi berbeda.

---

## Catatan TTL

Nilai `distributedLock.ttl` di payload tidak boleh melebihi `LOCK_RESOURCE_MAX_TTL` di backend. Apabila melebihi, backend menolak request acquire lock. TTL 300 detik (5 menit) umum dipakai untuk halaman master, sedangkan halaman composite cenderung memerlukan TTL lebih besar karena user mengedit header dan detail items sekaligus.

---

Pemecahan masalah lock pada frontend (lock conflict tidak muncul, dll) ada di [Troubleshooting](troubleshooting.md).
