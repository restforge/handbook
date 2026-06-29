# Keamanan Processor

> Pola wajib, anti-pattern, dan checklist pre-merge untuk processor.

Kembali ke [ikhtisar Processor](README.md).

---

## Pembagian Tanggung Jawab

Router (auto-generated) dan processor (custom) memiliki tanggung jawab yang berbeda dan saling melengkapi.

| Lapisan | Tanggung Jawab | Mekanisme |
|---------|----------------|-----------|
| Router | Validasi mekanis: tipe, required, format, enum, length; masking sensitive field di debug log | Auto-injected dari payload schema; di-overwrite setiap generate |
| Processor | Validasi semantik: business rule, authorization, status transition, konsistensi antar entitas | Ditulis manual |

Validasi router tidak menggantikan validasi business rule di processor. Router hanya memastikan bahwa input yang sampai ke processor sudah memenuhi kontrak payload secara mekanis.

---

## Auto-Hardening di Router

Generator meng-inject dua lapisan keamanan otomatis ke router setiap kali dijalankan. Lapisan ini tidak dapat dihapus secara tidak sengaja karena router selalu ditimpa.

### 1. Router-Level Validation

Request yang tidak memenuhi schema di-reject dengan HTTP 400 sebelum processor dipanggil.

| Aspek | Perilaku |
|-------|----------|
| `required: true` | Field absent, `null`, atau empty string di-reject |
| `type` | Tipe data di-enforce sesuai nilai yang dideklarasikan di payload |
| `format` | Regex check untuk `email`, `url`, `phone-id`, `uuid` |
| `minLength` / `maxLength` | Length check untuk field bertipe `string` |
| `enum` | Whitelist nilai yang diizinkan |
| `headers[].mapTo` | Type check eksplisit sebelum di-copy ke `input` |

### 2. Sensitive Field Masking

Field yang namanya cocok pattern default di-mask menjadi `***MASKED***` pada debug log di router. Pattern default mencakup: `password`, `passwd`, `pwd`, `secret`, `token`, `api_key`, `authorization`, `credit_card`, `cvv`, `ssn`, `pin`, `private_key`, `refresh_token`, `access_token`, `otp`, `mfa`.

Masking juga dapat diperluas ke field custom via `sensitive: true` di payload — misalnya `ktp_number`, `npwp`. Nilai actual tetap diteruskan ke processor.

---

## Pola Wajib di Processor

### 1. Validasi Business Rule setelah Router

Router sudah menangani validasi mekanis. Processor fokus pada validasi yang membutuhkan query database atau logika bisnis.

```js
async process(input, services) {
  const { db } = services;

  // so_id: uuid, required sudah divalidasi router.
  // Processor fokus ke business rule.

  const result = await db.executeQuery(
    'SELECT so_id, status, created_by FROM sales.order WHERE so_id = $1',
    [input.so_id]
  );
  const order = result.rows?.[0];

  if (!order) return { success: false, statusCode: 404, message: 'Order tidak ditemukan.' };

  // Business rule: hanya draft yang boleh di-submit
  if (order.status !== 'draft') {
    return { success: false, statusCode: 422, message: `Status saat ini: ${order.status}.` };
  }

  // Business rule: hanya pembuat order yang boleh submit
  if (order.created_by !== input.submitted_by) {
    return { success: false, statusCode: 403, message: 'Akses ditolak.' };
  }
}
```

### 2. Parameterized Query untuk SQL Value

Semua nilai dari user wajib masuk ke SQL via placeholder. Tidak boleh ada concat user input ke SQL string.

```js
// ✅ BENAR
const sql = 'UPDATE sales.order SET status = $1, updated_by = $2 WHERE so_id = $3';
await db.executeQuery(sql, ['approved', input.approver_id, input.so_id]);

// ❌ SALAH — SQL injection
const sql = `UPDATE sales.order SET status = '${input.status}' WHERE so_id = '${input.so_id}'`;
```

Placeholder per database: `$1, $2` (PostgreSQL), `?` (MySQL), `:1, :2` (Oracle).

### 3. Whitelist untuk Identifier Dinamis

Nama kolom dan nama tabel tidak bisa di-parameterize. Jika processor perlu menerima kolom atau operator dinamis dari user (dynamic search, sort), lakukan whitelist sebelum ke query.

```js
const VALID_SEARCH_COLUMNS = ['contact_name', 'email', 'phone'];

async process(input, services) {
  const { db } = services;

  if (!VALID_SEARCH_COLUMNS.includes(input.search_by)) {
    return {
      success: false,
      statusCode: 400,
      message: `Field pencarian tidak diizinkan. Pilih salah satu: ${VALID_SEARCH_COLUMNS.join(', ')}.`
    };
  }

  const result = await db.table('contact')
    .where(input.search_by, input.search_value)
    .where('is_active', true)
    .get();

  return { success: true, data: result };
}
```

### 4. Re-validate Header Authorization di Processor

Jika header menentukan authorization atau data scope, lakukan re-check di processor sebagai defense in depth — meskipun `enum` di payload sudah memfilter di router.

```js
const ALLOWED_APP_CODES = ['sales-app', 'inventory-app'];

async process(input, services) {
  if (!input.app_code || !ALLOWED_APP_CODES.includes(input.app_code)) {
    return { success: false, statusCode: 403, message: 'X-App-Code tidak valid.' };
  }
}
```

### 5. Jangan Log Data Sensitif

Logger pino di processor tidak melakukan auto-masking. Field sensitif tidak boleh masuk ke logger dalam bentuk plaintext.

```js
// ✅ BENAR
logger.info({ event: 'login', username: input.username, ip: req.ip }, 'Login attempt');

// ❌ SALAH — credential leak via log
logger.info({ event: 'login', username: input.username, password: input.password }, 'Login attempt');
```

### 6. Generic Error Response ke Client

Error ke client tidak boleh mengandung stack trace, SQL query, atau path file internal.

```js
try {
  const result = await db.executeQuery(sql, params);
  return { success: true, data: result.rows };
} catch (err) {
  logger.error({ event: 'approve_failed', orderId: input.so_id, error: err.message, stack: err.stack }, 'Failed');

  // Pesan generic ke client — detail ada di server log
  return { success: false, statusCode: 500, message: 'Terjadi kesalahan. Silakan coba lagi.' };
}
```

---

## Opt-Out Validasi Router

Use case legitimate seperti dynamic search atau advanced filter dapat menonaktifkan validasi router via `validate: false` di payload.

```json
{
  "request": {
    "validate": false,
    "body": {
      "search_by":    { "type": "string", "required": true },
      "search_value": { "type": "string", "required": true }
    }
  }
}
```

Saat opt-out diaktifkan, seluruh validasi mekanis menjadi tanggung jawab processor secara penuh.

```js
async process(input, services) {
  if (!input.order_id || typeof input.order_id !== 'string') {
    return { success: false, statusCode: 400, message: 'Parameter order_id wajib diisi.' };
  }

  const uuidRegex = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
  if (!uuidRegex.test(input.order_id)) {
    return { success: false, statusCode: 400, message: 'Format order_id tidak valid.' };
  }
}
```

> **Perlu diketahui:** Sensitive field masking di debug log tetap berfungsi meskipun `validate: false` diaktifkan.

---

## Anti-Pattern yang Harus Dihindari

| Pattern | Risiko | Alternatif |
|---------|--------|------------|
| Concat user input ke SQL string | SQL injection | Gunakan placeholder (`$1`, `?`, `:1`) |
| `eval(input.expression)` | Remote Code Execution | Whitelist operator, bangun expression secara struktural |
| `require(input.modulePath)` | Path traversal, RCE | Tidak ada use case yang sah |
| Return `error.stack` atau `error.message` ke client | Information disclosure | Generic message ke client, detail di server log |
| Log `input.password`, `input.token` | Credential leak via log | Whitelist field yang aman di-log |
| Forward `req.body` langsung ke `.insert()` tanpa field eksplisit | Field tamper, prototype pollution | Pick field satu per satu di processor |
| Asumsikan authorization sudah di-handle middleware | Privilege escalation jika middleware di-bypass | Cek ulang otorisasi di processor (defense in depth) |

---

## Checklist Pre-Merge

**Payload**

```
☐ Semua field di request.body/params/headers memiliki type eksplisit
☐ Field wajib ditandai required: true
☐ Field sensitif (password, token, KYC, PII) ditandai sensitive: true
☐ Field string punya maxLength yang reasonable
☐ Field dengan nilai terbatas menggunakan enum
☐ Field UUID pakai type: "uuid", email pakai format: "email"
☐ request.validate: false hanya digunakan jika benar-benar perlu
```

**Processor**

```
☐ Business rule (status transition, authorization, cross-entity) divalidasi di processor
☐ Semua SQL value via placeholder, tidak ada concat user input
☐ Identifier dinamis (kolom/tabel) di-whitelist di processor sebelum ke query
☐ Header yang menentukan authorization di-re-validate di processor
☐ Tidak ada data sensitif di logger pino
☐ Error response ke client generic, tanpa stack trace atau SQL
☐ Jika request.validate: false, validasi mekanis dilakukan manual di processor
```

---

Lihat juga: [Payload Processor](payload.md) · [Menulis Processor](handler.md) · [Konvensi Penamaan](naming.md)
