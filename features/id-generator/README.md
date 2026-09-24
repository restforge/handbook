# ID Generator

ID Generator berfungsi untuk membuat identifier otomatis seperti nomor sales order, OTP, license key, dan voucher melalui satu layanan terpusat di atas Redis. Uniqueness tetap terjamin meski banyak request datang bersamaan, dan retry karena gangguan jaringan tidak menghasilkan nomor ganda.

Halaman ini berisi panduan tugas untuk konfigurasi backend maupun frontend. Spec atribut UDF `defaultValue.source: "idgen"` secara lengkap tersedia di [`catalogs/udf/id-generation.md`](../../catalogs/udf/id-generation.md).

> **Prasyarat:** Redis Server harus berjalan dan dapat diakses, karena seluruh counter dan pool nilai disimpan di Redis.

---

## Mengaktifkan IDGen di Backend (Backend Configuration)

Tambahkan master toggle dan koneksi Redis di file `.env`, lalu restart RESTForge runtime.

```env
IDGEN_ENABLED=true
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DB=0
```

Verifikasi lewat endpoint health sebelum dipakai di alur bisnis.

```
GET /api/inventory/idgen/health
```

```json
{ "success": true, "data": { "enabled": true, "redisConnected": true } }
```

> **Catatan:** Bila response berstatus 503, periksa nilai `IDGEN_ENABLED`, koneksi Redis, dan password Redis.

---

## Mengisi Field Otomatis di Frontend (Frontend Configuration)

Field pada halaman CRUD hasil generate dapat terisi otomatis dari IDGen melalui deklarasi `defaultValue.source: "idgen"` di payload UDF, tanpa menulis JavaScript.

```json
{
  "name": "invoice_no",
  "type": "text",
  "label": "Nomor Invoice",
  "readonly": true,
  "defaultValue": {
    "source": "idgen",
    "mode": "number",
    "resource": "invoice",
    "format": "yyyymm",
    "numDigits": 4,
    "separator": "/"
  }
}
```

Saat halaman di-generate ulang, form memanggil endpoint IDGen secara otomatis ketika dibuka, lalu mengisi field dengan nilai hasil generate.

> **Catatan:** Deklarasi frontend ini mengonsumsi service backend yang sama, sehingga backend wajib sudah aktif (`IDGEN_ENABLED=true` dan Redis terhubung). Bila tidak, field gagal terisi dan form menampilkan notifikasi error.

Pola idgen hanya berlaku pada field bertipe `number`, `text`, atau `textarea`. Untuk pola "preview lalu konfirmasi" memakai `reserve: true`, lihat Contoh End-to-End di akhir halaman. Spec atribut UDF lengkap ada di [`catalogs/udf/id-generation.md`](../../catalogs/udf/id-generation.md).

---

## Membuat Nomor Berurutan (Sequential Numbers)

Mode `number` menghasilkan nomor dokumen berurutan dengan format tanggal.

```
POST /api/inventory/idgen/number
```

```json
{
  "resource": "sales_order",
  "format": "yyyymm",
  "prefix": "SO",
  "separator": ".",
  "numDigits": 4
}
```

```json
{ "success": true, "data": { "value": "SO.202604.0001", "counter": 1, "isReused": false } }
```

Request berikutnya menghasilkan `SO.202604.0002`, lalu `SO.202604.0003`, dan seterusnya. Parameter `resource` memisahkan counter antar entity, sehingga nomor sales order tidak pernah tercampur dengan nomor invoice.

### Mengatur Kapan Counter Di-reset (Reset Period)

Parameter `format` menentukan kapan penomoran kembali ke angka satu.

| Format | Reset terjadi | Contoh hasil |
|--------|---------------|--------------|
| `text` | Tidak pernah | `INV-0001` |
| `yyyy` | Setiap tahun | `RPT/2026/001` |
| `yyyymm` | Setiap bulan | `SO.202604.0001` |
| `yyyymmdd` | Setiap hari | `LOG_20260408_00001` |

> **Catatan:** Reset terjadi otomatis tanpa cron job. Counter periode lama dibiarkan kedaluwarsa sendiri lewat TTL Redis.

---

## Memakai Kembali Nomor yang Dihapus (Reuse Pool)

Saat sebuah dokumen dihapus, kembalikan nomornya ke reuse pool agar urutan tetap rapat tanpa lubang.

```
POST /api/inventory/idgen/number/release
```

```json
{
  "resource": "sales_order",
  "formatKey": "SO_yyyymm_202604",
  "counterInt": 3
}
```

Sertakan `reuseDelete: true` pada request `number` berikutnya agar nomor terkecil di pool diambil lebih dulu sebelum counter dinaikkan.

```
POST /api/inventory/idgen/number
```

```json
{ "resource": "sales_order", "format": "yyyymm", "prefix": "SO", "numDigits": 4, "reuseDelete": true }
```

> **Catatan:** Tanpa `reuseDelete: true`, counter selalu naik dan pool diabaikan.

---

## Menampilkan Nomor Preview di Form (Reservation)

Reservation mendukung pola "preview lalu konfirmasi" pada form yang mungkin dibatalkan, misalnya form sales order. Nomor ditahan sementara saat form dibuka, dan baru menjadi final saat form disubmit.

Saat form dibuka, lakukan reserve untuk mendapat nomor preview beserta token.

```
POST /api/inventory/idgen/reserve
```

```json
{ "resource": "sales_order", "format": "yyyymm", "prefix": "SO", "numDigits": 4, "ttl": 1800 }
```

```json
{ "success": true, "data": { "token": "550e8400-...", "value": "SO.202604.0012", "counter": 12 } }
```

Saat form disubmit, kunci nomor sebagai final dengan token tersebut.

```
POST /api/inventory/idgen/reserve/confirm
```

```json
{ "token": "550e8400-..." }
```

Operasi lain yang tersedia dengan token yang sama:

| Operasi | Endpoint | Kapan dipakai |
|---------|----------|---------------|
| Cancel | `POST /idgen/reserve/cancel` | User menekan tombol "Batal", nomor langsung dilepas ke reuse pool |
| Extend | `POST /idgen/reserve/extend` | Form panjang, batas waktu perlu diperpanjang |
| Lookup | `GET /idgen/reserve/:token` | Memeriksa status reservation tanpa mengubahnya |

> **Catatan:** Bila form ditinggalkan tanpa konfirmasi, nomor dilepas otomatis setelah batas waktu (`ttl`) habis. Memanggil cancel hanya mempercepat pelepasan tersebut.

---

## Membuat OTP atau PIN (OTP and PIN)

Mode `pin` menghasilkan kode numerik acak seperti OTP. Parameter `ttl` menetapkan masa berlaku dalam detik, dan `ensureUnique` mencegah nilai sama dikeluarkan dua kali selama masih berlaku.

```
POST /api/inventory/idgen/pin
```

```json
{ "resource": "user_otp", "digits": 6, "ensureUnique": true, "ttl": 300 }
```

```json
{ "success": true, "data": { "value": "938749", "expiresAt": "2026-04-08T12:39:56.789Z" } }
```

> **Catatan:** Keacakan diambil dari `crypto.randomInt()` yang crypto-secure, bukan `Math.random()`, sehingga layak untuk kebutuhan keamanan.

---

## Membuat License Key (Serial Key)

Mode `serial` menghasilkan license key atau activation code. Pattern menentukan bentuk hasilnya, dengan `X` mewakili huruf besar dan angka acak.

```
POST /api/inventory/idgen/serial
```

```json
{ "resource": "license", "pattern": "XXXX-XXXX-XXXX-XXXX" }
```

```json
{ "success": true, "data": { "value": "A383-KL7E-XO09-N73N" } }
```

Placeholder pattern yang umum dipakai:

| Placeholder | Digantikan oleh |
|-------------|-----------------|
| `9` atau `N` | Angka `0-9` |
| `A` | Huruf besar `A-Z` |
| `X` | Huruf besar dan angka |
| `H` | Hex huruf besar `0-9 A-F` |

> **Catatan:** Pada mode `serial`, `ensureUnique` aktif secara default karena license key wajib unik.

---

## Membuat Voucher Code (Voucher Code)

Mode `code` menghasilkan voucher atau kupon promo, biasanya dengan pattern numerik pendek dan `ttl` sesuai durasi promo.

```
POST /api/inventory/idgen/code
```

```json
{ "resource": "voucher", "pattern": "9999-9999", "ttl": 86400 }
```

```json
{ "success": true, "data": { "value": "3874-9098", "expiresAt": "2026-04-09T12:34:56.789Z" } }
```

---

## Mencegah Nomor Ganda saat Retry (Idempotency)

Sertakan header `Idempotency-Key` agar retry dengan key yang sama mengembalikan hasil identik tanpa membuat nomor baru. Fitur ini berguna ketika response pertama hilang karena gangguan jaringan dan request dikirim ulang.

```
POST /api/inventory/idgen/number
Idempotency-Key: user-123:create-so:order-2026-04-08-001
```

```json
{ "resource": "sales_order", "format": "yyyymm", "prefix": "SO", "numDigits": 4 }
```

> **Catatan:** Idempotency bersifat opt-in. Tanpa header tersebut, setiap request menghasilkan nilai baru. Pola key yang disarankan menggabungkan identitas client, nama operasi, dan kunci bisnis.

---

## Penggunaan IDGen dalam Processor (Service Injection)

Di dalam processor, ID Generator tersedia sebagai `services.idgen` tanpa perlu memanggil HTTP. Contoh mengisi nomor sales order sebelum data masuk ke database:

```js
async function onBeforeInsert(data, services) {
  const { idgen, createError } = services;
  if (!idgen || !idgen.isEnabled()) {
    return createError(503, 'ID Generator service unavailable');
  }

  const result = await idgen.number({
    resource: 'sales_order',
    format: 'yyyymm',
    prefix: 'SO',
    separator: '.',
    numDigits: 4,
    reuseDelete: true,
    idempotencyKey: data._requestId
  });

  data.so_number = result.value;
  return { success: true, data };
}
```

> **Catatan:** Saat dokumen dihapus, lepas nomornya dengan `idgen.releaseNumber()` di handler `onAfterDelete`. Kegagalan pelepasan sebaiknya tidak menggagalkan proses delete.

---

## Menangani Error Umum (Handling Common Errors)

Validasi parameter dilakukan lebih awal, sehingga nilai yang keliru langsung ditolak. Error berikut yang paling sering ditemui:

| Kode error | Status | Penyebab dan penanganan |
|------------|--------|-------------------------|
| `IDGEN_COUNTER_OVERFLOW` | 422 | Counter melebihi batas digit. Tetapkan `numDigits` dengan cadangan sejak awal |
| `IDGEN_UNIQUE_RETRY_EXHAUSTED` | 409 | Pattern terlalu pendek untuk volume nilai. Perpanjang pattern |
| `IDGEN_RESERVATION_NOT_FOUND` | 404 | Token reservation tidak ada, sudah dikonfirmasi, atau kedaluwarsa |
| `IDGEN_DISABLED` | 503 | Fitur belum aktif atau Redis tidak terhubung |

> **Catatan:** Counter overflow tidak dapat dipulihkan otomatis, sehingga pemilihan `numDigits` perlu mempertimbangkan estimasi volume per periode ditambah satu atau dua digit cadangan.

---

## Contoh End-to-End: Nomor Invoice dengan Reservasi (End-to-End Example)

Bagian ini menyatukan konfigurasi backend dan frontend untuk satu kasus nyata, yaitu form invoice yang menampilkan nomor preview saat dibuka dan memfinalkannya saat disimpan.

### Langkah 1 — Aktifkan IDGen di Backend

Tambahkan toggle dan koneksi Redis di file `.env`, lalu restart runtime.

```env
IDGEN_ENABLED=true
REDIS_HOST=localhost
REDIS_PORT=6379
```

### Langkah 2 — Deklarasikan Field di Payload UDF

Tambahkan field `invoice_no` dengan `reserve: true` pada payload halaman, lalu generate ulang halaman tersebut.

```json
{
  "name": "invoice_no",
  "type": "text",
  "label": "Nomor Invoice",
  "readonly": true,
  "defaultValue": {
    "source": "idgen",
    "mode": "number",
    "resource": "invoice",
    "format": "yyyymm",
    "numDigits": 4,
    "separator": "/",
    "reserve": true,
    "ttl": 120
  }
}
```

> **Catatan:** Hanya satu field per halaman yang boleh memakai `reserve: true`. Nilai `ttl` (detik) tidak boleh melebihi batas TTL reservasi di backend.

### Langkah 3 — Alur yang Terjadi di Runtime

Form hasil generate menjalankan tiga panggilan ke endpoint backend mengikuti siklus hidup form.

| Aksi user | Panggilan frontend | Endpoint backend | Hasil |
|-----------|--------------------|------------------|-------|
| Membuka form | reserve | `POST /idgen/reserve` | Field terisi preview `202606/0001` beserta token |
| Submit form | confirm | `POST /idgen/reserve/confirm` | Nomor dikunci sebagai final |
| Menutup atau batal | cancel | `POST /idgen/reserve/cancel` | Nomor dilepas ke reuse pool |

Token reservation dikelola otomatis oleh form. Preview yang ditinggalkan tanpa submit akan dilepas, baik melalui cancel maupun setelah `ttl` habis.

> **Catatan:** Ketiga endpoint tersebut dilayani oleh service backend yang sama dengan HTTP API di bagian sebelumnya. Deklarasi UDF hanya memindahkan pemicunya dari kode manual ke form hasil generate.

---

## Referensi Cepat (Quick Reference)

| Properti | Nilai |
|----------|-------|
| Aktivasi backend | `IDGEN_ENABLED=true` di file `.env` |
| Aktivasi frontend | UDF `defaultValue.source: "idgen"` pada field |
| Dependency | Redis Server |
| Mode tersedia | `number`, `pin`, `serial`, `code`, `random` |
| Sumber keacakan | `crypto.randomInt()` (crypto-secure) |
| Idempotency | Opt-in lewat header `Idempotency-Key` |
| Reuse pool dan reservation | Hanya untuk mode `number` |
| Pemakaian di processor | `services.idgen` |
| Base URL HTTP API | `/api/{project}/idgen/...` |
| Database | PostgreSQL, MySQL, Oracle, SQLite |

**Lihat juga**: [Referensi UDF id-generation](../../catalogs/udf/id-generation.md) · [`catalogs/`](../../catalogs/) · [`features/`](../)
