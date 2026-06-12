# Anatomi Halaman CRUD — `pages[]`

> Struktur lengkap satu page object untuk `pageType: "crud"` (tipe default).

## Sintaks

```json
{
    "pageId": "contact",
    "pageTitle": "Contact",
    "pageSubtitle": "Kelola data kontak",
    "pageIcon": "users",
    "pageGroup": ["Master"],
    "pageType": "crud",
    "apiPath": "contact",
    "primaryKey": "contact_id",
    "displayField": "contact_name",
    "features": { /* ... */ },
    "fields": [ /* ... */ ],
    "fieldRows": [ /* ... */ ],
    "fieldStates": [ /* ... */ ],
    "details": [ /* ... */ ],
    "workflow": { /* ... */ },
    "workflowActions": [ /* ... */ ]
}
```

## Properti

### Identitas dan Tampilan

| Properti | Tipe | Default | Wajib | Keterangan |
|----------|------|---------|:-----:|-----------|
| `pageId` | string | — | ✓ | Identifier unik halaman. Pattern: `^[a-zA-Z0-9_-]+$` |
| `pageTitle` | string | — | ✓ | Judul halaman di header |
| `pageSubtitle` | string | — | ✗ | Subjudul di bawah `pageTitle` |
| `pageIcon` | string | — | ✗ | Nama ikon (Feather Icons). Contoh: `users`, `package`, `truck` |
| `pageGroup` | array of string | — | ✗ | Hierarki grup di sidebar (auto-derive navigation). Maksimal 2 level |
| `pageType` | string | `"crud"` | ✗ | Tipe halaman. Valid: `"crud"`, `"dashboard"` |

`pageId` dipakai sebagai filename output HTML. Pattern `^[a-zA-Z0-9_-]+$` memblokir karakter path traversal (`/`, `\`, `..`). Karakter yang ditolak menghasilkan error:

```
[<pageId>] pageId '<value>' is invalid. Must match pattern ^[a-zA-Z0-9_-]+$
(letters, digits, hyphen, underscore only; no path separators)
```

### Koneksi ke Backend

| Properti | Tipe | Default | Wajib | Keterangan |
|----------|------|---------|:-----:|-----------|
| `apiPath` | string | — | ✓ | Path endpoint API, digabung dengan `apiBaseUrl` |
| `primaryKey` | string | — | ✓ | Nama field primary key untuk operasi CRUD. Tidak perlu ada di `fields` |
| `displayField` | string | — | ✓ | Nama field di `fields` yang dipakai sebagai label record (dialog konfirmasi, breadcrumb) |

### Definisi Data

| Properti | Tipe | Default | Wajib | Keterangan |
|----------|------|---------|:-----:|-----------|
| `fields` | array | — | ✓ | Array definisi field (minimal 1). Lihat [`field-attributes.md`](./field-attributes.md) |
| `fieldRows` | array | — | ✗ | Konfigurasi grid layout. Lihat [`field-rows.md`](./field-rows.md) |
| `fieldStates` | array | — | ✗ | Conditional state per field. Lihat [`field-states.md`](./field-states.md) |
| `features` | object | `{}` | ✗ | Fitur halaman (search, filter, layout). Lihat [`features.md`](./features.md) |

### Pola Lanjutan

| Properti | Tipe | Default | Wajib | Keterangan |
|----------|------|---------|:-----:|-----------|
| `details` | array | — | ✗ | Array tabel detail untuk pola master-detail. Lihat [`master-detail.md`](./master-detail.md) |
| `workflow` | object | — | ✗ | Konfigurasi state machine (`statusField`). Lihat [`workflow-actions.md`](./workflow-actions.md) |
| `workflowActions` | array | — | ✗ | Tombol aksi yang mengubah status record. Lihat [`workflow-actions.md`](./workflow-actions.md) |

## Aturan Validasi Page CRUD

| Aturan | Kondisi Error |
|--------|---------------|
| `pageId` valid pattern | `"[<pageId>] pageId '<value>' is invalid. Must match pattern ^[a-zA-Z0-9_-]+$ ..."` |
| `pageType` valid | `"[<pageId>] pageType '<value>' is invalid. Valid types: crud, dashboard"` |
| `fields[]` valid (lihat field validation) | Per-field error: lihat [`field-attributes.md`](./field-attributes.md) |
| `displayField` ada di `fields[]` | Warning: `"[<pageId>] displayField '<value>' is not found in the 'fields' definition"` |

## `pageGroup` dan Auto-Derive Sidebar

Field `pageGroup` memungkinkan sidebar auto-derive (otomatis dibentuk dari semua page) tanpa menulis blok `navigation` manual di level root.

```json
{
    "pageId": "contact",
    "pageGroup": ["Master", "People"],
    "pageTitle": "Contact"
}
```

Aturan:

| Aturan | Keterangan |
|--------|-----------|
| Maksimal level | 2 (page sendiri menempati level ke-3) |
| Tipe element | String non-empty |
| Casing | Sensitif. Casing berbeda dianggap grup terpisah (validator menghasilkan warning) |
| Konflik dengan `navigation` | Jika `navigation` ada di root, `pageGroup` diabaikan (validator menghasilkan warning) |

Jika label di pageGroup dieja dengan casing berbeda antar page, validator menghasilkan warning:

```
pageGroup label '<duplicate>' differs from previously seen '<first>' only by case;
these will be treated as separate groups. Use a single consistent casing if grouping is intended.
```

## Contoh Page CRUD Minimal

```json
{
    "pageId": "contact",
    "pageTitle": "Contact",
    "apiPath": "contact",
    "primaryKey": "contact_id",
    "displayField": "contact_name",
    "fields": [
        {
            "name": "contact_name",
            "label": "Nama Kontak",
            "type": "text",
            "required": true,
            "inTable": true,
            "tableOrder": 1
        }
    ]
}
```

## Contoh Page CRUD Lengkap

```json
{
    "pageId": "contact",
    "pageTitle": "Contact",
    "pageSubtitle": "Kelola data kontak",
    "pageIcon": "users",
    "pageGroup": ["Master"],
    "apiPath": "contact",
    "primaryKey": "contact_id",
    "displayField": "contact_name",
    "features": {
        "enableSearch": true,
        "enableStatusFilter": true,
        "statusFilter": {
            "field": "is_active",
            "label": "Status",
            "options": [
                {"value": "true", "text": "Active"},
                {"value": "false", "text": "Inactive"}
            ]
        },
        "fieldLayout": "vertical"
    },
    "fields": [
        {
            "name": "contact_name",
            "label": "Nama Kontak",
            "type": "text",
            "required": true,
            "inTable": true,
            "tableOrder": 1,
            "maxlength": 255
        },
        {
            "name": "email",
            "label": "Email",
            "type": "text",
            "inTable": true,
            "tableOrder": 2,
            "placeholder": "email@contoh.com"
        },
        {
            "name": "is_active",
            "label": "Status",
            "type": "checkbox",
            "inTable": true,
            "tableOrder": 3,
            "defaultValue": true,
            "checkboxText": {
                "checked": "Active",
                "unchecked": "Inactive"
            }
        }
    ],
    "fieldRows": [
        {"fields": ["contact_name", "email"]},
        {"fields": ["is_active"]}
    ]
}
```

---

← [`app-config.md`](./app-config.md) | [Selanjutnya: `field-types.md`](./field-types.md) →
