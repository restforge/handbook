# Konvensi Penamaan

> Aturan format nama untuk `pageId`, `widgetId`, `name` field, dan field lain di UDF.

## Ringkasan Pattern Resmi

| Identifier | Pattern Regex |
|------------|---------------|
| `pageId` (CRUD) | `^[a-zA-Z0-9_-]+$` |
| `pageId` (Dashboard) | `^[a-z][a-z0-9-]*$` |
| `widgetId` (Dashboard) | `^[a-z][a-z0-9_-]*$` |

## `pageId`

### Page CRUD

Pattern: `^[a-zA-Z0-9_-]+$`

| Karakter Valid | Karakter Tidak Valid |
|----------------|---------------------|
| Letters (uppercase/lowercase) | Spaces |
| Digits | Path separator (`/`, `\`) |
| Hyphen (`-`) | Path traversal (`..`) |
| Underscore (`_`) | Karakter khusus lain |

Contoh valid:

```
contact, contact-list, ContactList, contact_v2, page-2025-Q1
```

Contoh invalid:

```
foo/bar         → ditolak (path separator)
foo\bar         → ditolak (path separator)
../etc/passwd   → ditolak (path traversal)
my page         → ditolak (whitespace)
```

Pattern ini memblokir path traversal karena `pageId` dipakai sebagai filename HTML output. Tanpa validasi pattern, user berbahaya bisa menulis `pageId` yang menghasilkan file di lokasi tidak diinginkan.

### Page Dashboard

Pattern lebih ketat: `^[a-z][a-z0-9-]*$`

| Karakter Valid | Karakter Tidak Valid |
|----------------|---------------------|
| Lowercase letters (`a-z`) | Uppercase letters |
| Digits | Underscore |
| Hyphen | Path separator |

Harus diawali dengan huruf kecil.

Contoh valid:

```
overview, sales-dashboard, kpi-q1-2025, finance
```

Contoh invalid:

```
Overview         → ditolak (uppercase)
sales_dashboard  → ditolak (underscore)
1-finance        → ditolak (diawali digit)
```

Underscore tidak diizinkan di page dashboard karena `pageId` dipakai sebagai filename dan beberapa file system memperlakukan underscore khusus.

## `widgetId`

Pattern: `^[a-z][a-z0-9_-]*$`

| Karakter Valid | Karakter Tidak Valid |
|----------------|---------------------|
| Lowercase letters (`a-z`) | Uppercase letters |
| Digits | Path separator |
| Hyphen | Whitespace |
| Underscore | Karakter khusus lain |

Harus diawali dengan huruf kecil.

`widgetId` harus unik per page. Duplikat menghasilkan error:

```
<path>.widgetId '<value>' is duplicated; first defined at <previous-path>.
widgetId must be unique within a single dashboard page.
```

Contoh valid:

```
revenue-chart, kpi_total, sales-by-region, top-products-2025
```

Contoh invalid:

```
RevenueChart    → ditolak (uppercase)
1-kpi           → ditolak (diawali digit)
my widget       → ditolak (whitespace)
```

## `name` Field

Validator tidak memeriksa format `name` field, tetapi generator dan migrator backend memakai konvensi **snake_case**.

| Format | Contoh |
|--------|--------|
| snake_case | `contact_name`, `email_address`, `is_active`, `created_at` |

### Auto-Conversion oleh Generator

Generator mengonversi `name` field secara otomatis ke format lain untuk berbagai kebutuhan:

| Format Asal | Format Tujuan | Contoh | Digunakan Untuk |
|-------------|---------------|--------|-----------------|
| snake_case | kebab-case | `contact_name` → `contact-name` | HTML `id`, CSS class |
| snake_case | camelCase | `contact_name` → `contactName` | Variabel JavaScript |

Konvensi snake_case adalah default karena:

- Konsisten dengan kolom database PostgreSQL/MySQL (snake_case)
- Konsisten dengan field di RDF/SDF
- Mudah dikonversi secara otomatis ke kebab-case dan camelCase

## `appCode`

Tidak ada pattern formal yang divalidasi. Konvensi yang direkomendasikan: **kebab-case**.

| Format | Contoh |
|--------|--------|
| kebab-case | `contact-mgmt`, `inventory-system`, `wms-mobile` |

`appCode` dipakai sebagai identifier internal aplikasi, termasuk untuk:

- Namespace WebSocket Live Sync (jika pakai plugin yang mendukung)
- Identifier autentikasi (di plugin `vanilla-js-auth`)
- Prefix di nama file output (di beberapa plugin)

## `apiPath`

Konvensi yang direkomendasikan: **kebab-case** (mirror konvensi RDF endpoint).

| Format | Contoh |
|--------|--------|
| kebab-case | `contact`, `sales-order`, `stock-inbound` |

`apiPath` di-append ke `apiBaseUrl` untuk membentuk endpoint final:

```
apiBaseUrl: http://localhost:3031/api/dbcontact
apiPath:    sales-order
endpoint:   http://localhost:3031/api/dbcontact/sales-order
```

Pastikan konsisten dengan path endpoint di backend (RDF) untuk menghindari 404.

## `pageGroup` (Label Folder Sidebar)

Tidak ada pattern formal. Bebas format teks. Konvensi yang direkomendasikan:

| Skenario | Konvensi |
|----------|----------|
| Bahasa Indonesia | Title Case (`Master Data`, `Transaksi Penjualan`) |
| Bahasa Inggris | Title Case (`Master Data`, `Sales Transactions`) |

### Case Sensitivity Warning

`pageGroup` label bersifat case-sensitive di validator. Jika satu page memakai `"Master Data"` dan page lain memakai `"master data"`, keduanya dianggap grup terpisah, dan validator menghasilkan warning:

```
pageGroup label '<duplicate>' differs from previously seen '<first>' only by case;
these will be treated as separate groups. Use a single consistent casing if grouping is intended.
```

Pertahankan casing konsisten di seluruh page untuk menghindari grup terduplikasi di sidebar.

## Tabel Quick Reference

| Apa | Format | Contoh |
|-----|--------|--------|
| `pageId` (CRUD) | letters + digits + hyphen + underscore | `contact`, `sales-order-2025` |
| `pageId` (Dashboard) | lowercase + digits + hyphen, mulai huruf | `overview`, `sales-dashboard` |
| `widgetId` | lowercase + digits + hyphen + underscore, mulai huruf | `revenue-chart`, `kpi_total` |
| `name` field | snake_case | `contact_name`, `is_active` |
| `appCode` | kebab-case | `contact-mgmt` |
| `apiPath` | kebab-case | `sales-order` |
| `pageGroup` label | Title Case bebas | `Master Data` |

## Rekomendasi untuk Page Baru

Pattern `pageId` CRUD lebih longgar daripada `pageId` Dashboard karena menerima uppercase dan underscore. Pattern CRUD tetap dipertahankan agar aplikasi yang sudah ada tidak rusak.

Untuk page baru, **disarankan memakai pattern dashboard yang lebih ketat** (lowercase kebab-case) di page CRUD maupun dashboard agar konsisten.

---

← [`dashboard-page.md`](./dashboard-page.md) | [Selanjutnya: `validation-rules.md`](./validation-rules.md) →
