# Halaman Beranda — `homepage`

> Page default yang dibuka ketika URL root aplikasi diakses.

## Konsep

Tanpa `homepage`, halaman pertama di `pages[]` menjadi default ketika user mengakses URL root aplikasi. Field `homepage` memungkinkan override eksplisit ke `pageId` tertentu.

```json
{
    "homepage": "dashboard",
    "pages": [
        {"pageId": "customer", "pageType": "crud", "pageTitle": "Customer"},
        {"pageId": "supplier", "pageType": "crud", "pageTitle": "Supplier"},
        {"pageId": "dashboard", "pageType": "dashboard", "pageTitle": "Overview"}
    ]
}
```

Behavior: ketika user buka `http://localhost:3000/`, aplikasi otomatis redirect ke halaman `dashboard`.

## Sintaks

| Properti | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `homepage` | string | ✗ | `pageId` yang dijadikan default. Harus ada di `pages[]` |

## Aturan Validasi

| Aturan | Hasil |
|--------|-------|
| Harus non-empty string jika ada | Error: `"homepage must be a non-empty string (pageRef) when provided"` |
| Harus ada di `pages[]` (scope `app`) | Error: `"homepage '<value>' does not match any pageId in 'pages'. Add the page or fix the reference."` |
| Orphan di scope `form` | Warning: `"homepage '<value>' does not match any pageId in 'pages'. It will resolve at runtime if the page is generated later in a separate session."` |
| `pageType` halaman target harus didukung | Error: `"homepage '<value>' references a page with unsupported pageType '<type>'. Supported types: crud, dashboard."` |

### Tipe Halaman yang Didukung

Hanya dua `pageType` yang valid sebagai homepage:

```
crud, dashboard
```

`pageType` lain (jika ada plugin custom yang menambah tipe) tidak boleh dipakai sebagai homepage.

## Scope `app` vs `form` (Generate Mode)

| Scope | Perilaku jika `homepage` orphan |
|-------|--------------------------------|
| `app` | Error: page wajib ada di `pages[]` |
| `form` | Warning: page boleh belum ada (akan di-resolve runtime jika di-generate di session terpisah) |

Pembedaan ini mendukung workflow generate per-page tanpa harus regenerate seluruh aplikasi.

## Pola Penggunaan

### Dashboard sebagai Default

Pola paling umum: dashboard berisi KPI dan ringkasan, dijadikan landing page.

```json
{
    "homepage": "dashboard",
    "pages": [
        {
            "pageId": "dashboard",
            "pageType": "dashboard",
            "pageTitle": "Overview",
            "dataSources": { /* ... */ },
            "rows": [ /* ... */ ]
        },
        {"pageId": "customer", "pageType": "crud", /* ... */ },
        {"pageId": "supplier", "pageType": "crud", /* ... */ }
    ]
}
```

### CRUD Page sebagai Default

Untuk aplikasi sederhana tanpa dashboard, page CRUD utama bisa jadi homepage.

```json
{
    "homepage": "todo",
    "pages": [
        {"pageId": "todo", "pageType": "crud", "pageTitle": "To-Do List", /* ... */ }
    ]
}
```

### Tanpa `homepage`

Default behavior: page pertama di `pages[]` menjadi landing page. Tidak ada error jika `homepage` tidak ditulis.

## Interaksi dengan Plugin

Plugin generator bertanggung jawab membuat redirect dari URL root ke halaman target. Implementasi spesifik (mis. memakai meta refresh, JavaScript redirect, atau server-side rewrite) bergantung pada plugin. Lihat dokumentasi plugin untuk detail.

## Best Practice

| Skenario | Rekomendasi |
|----------|-------------|
| Aplikasi dengan dashboard | Set `homepage` ke `pageId` dashboard |
| Aplikasi single-page | Tidak perlu set `homepage`. Page satu-satunya otomatis menjadi default |
| Multi-page tanpa dashboard | Set `homepage` ke page paling sering dipakai |
| Aplikasi dengan auth | Plugin auth biasanya menampilkan login page sebelum homepage |

---

← [`navigation.md`](./navigation.md) | [Selanjutnya: `live-sync.md`](./live-sync.md) →
