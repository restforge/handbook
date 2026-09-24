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
| `pageSubject` | string | nilai `pageTitle` | ✗ | Nama objek tunggal yang disebut tombol tambah, judul form, tombol penyimpan, dan pesan sukses. Ditulis hanya bila judul halaman terpaksa berbeda dari nama objeknya |
| `addVerb` | string | `"Add"` | ✗ | Verba aksi tambah. Nilai tertutup: `"Add"`, `"Create"`, `"Upload"`, `"Invite"` |
| `showNewBadge` | boolean | `true` | ✗ | Badge `New` pada baris yang dibuat hari ini menurut tanggal server. Butuh kolom `created_at` di data baris dan field `timestamp` pada response `/datatables` |

`pageId` dipakai sebagai filename output HTML. Pattern `^[a-zA-Z0-9_-]+$` memblokir karakter path traversal (`/`, `\`, `..`). Karakter yang ditolak menghasilkan error:

```
[<pageId>] pageId '<value>' is invalid. Must match pattern ^[a-zA-Z0-9_-]+$
(letters, digits, hyphen, underscore only; no path separators)
```

#### Label Alur Tambah Data

Satu alur tambah data memakai kalimat yang sama di empat titik: tombol tambah di baris judul halaman, judul panel form, tombol penyimpan di footer panel, dan tombol pada daftar kosong. Kalimatnya `{addVerb} {pageSubject}`, misalnya `Add Contact`. Form ubah memakai judul `Edit {pageSubject}` dengan tombol penyimpan `Save Changes`, dan pesan sukses menyebut objeknya: `{pageSubject} added` (`created`, `uploaded`, atau `invited` mengikuti verba) serta `{pageSubject} updated`. Perilaku selengkapnya ada di [`features.md`](./features.md#perilaku-form-tambah-dan-ubah).

`pageSubject` hanya perlu ditulis bila judul halaman berbeda dari nama objeknya:

```json
{
    "pageTitle": "Master Category",
    "pageSubject": "Category"
}
```

Tanpa `pageSubject`, tombol pada halaman itu berbunyi `Add Master Category`. Nama objek ditulis tunggal karena satu kali aksi menghasilkan satu record.

Pemilihan verba:

| `addVerb` | Dipakai saat | Contoh |
|-----------|--------------|--------|
| `"Add"` | Objek sudah ada konsepnya di dunia nyata dan sistem hanya mencatatnya ke dalam kumpulan (nilai default) | `Add Employee`, `Add Product` |
| `"Create"` | Objek lahir dari aksi itu sendiri, punya nomor dan siklus status | `Create Purchase Order`, `Create Invoice` |
| `"Upload"` | Objeknya file yang datang dari luar sistem | `Upload Attachment` |
| `"Invite"` | Objeknya orang, dan pencatatan selesai setelah orang itu menerima undangan | `Invite Member` |

Nilai di luar keempatnya ditolak, termasuk `"New"`, `"Add New"`, `"Register"`, dan `"Submit"`.

Badge `New` di kolom identitas dimatikan per halaman dengan `showNewBadge: false`. Prasyarat dan bentuk badge dijelaskan di [`features.md`](./features.md#badge-new).

### Koneksi ke Backend

| Properti | Tipe | Default | Wajib | Keterangan |
|----------|------|---------|:-----:|-----------|
| `apiPath` | string | — | ✓ | Path endpoint API, digabung dengan `apiBaseUrl` |
| `primaryKey` | string | — | ✓ | Nama field primary key untuk operasi CRUD. Tidak perlu ada di `fields` |
| `displayField` | string | — | ✓ | Nama field di `fields` yang dipakai sebagai label record (dialog konfirmasi, breadcrumb) |
| `versionField` | string | — | ✗ | Nama field token versi baris (misal `row_version`), dipakai form ubah untuk optimistic concurrency. Tidak perlu ada di `fields` |

Tanpa `versionField`, form ubah mengirim body seperti sekarang, tanpa pemeriksaan versi apa pun. Dengan `versionField`, `loadRecord()` menyimpan nilai field itu dari respons `/first`, dan `Save Changes` mode ubah mengirim `options.expectedVersion` berisi nilai tersimpan itu, lalu menolak menimpa data bila server membalas konflik versi. Properti ini murni sisi frontend: backend-nya sendiri mensyaratkan RDF endpoint mendeklarasikan blok `concurrency` (lihat [`../rdf/concurrency.md`](../rdf/concurrency.md)) agar server benar-benar memeriksa token itu. Perilaku selengkapnya ada di [`features.md`](./features.md#perilaku-form-tambah-dan-ubah).

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
| `workflow` | object | — | ✗ | Konfigurasi state machine (`statusField`, `transitions`). Lihat [`workflow-actions.md`](./workflow-actions.md) |
| `workflowActions` | array | — | ✗ | Tombol aksi yang mengubah status record. Lihat [`workflow-actions.md`](./workflow-actions.md) |

## Aturan Validasi Page CRUD

| Aturan | Kondisi Error |
|--------|---------------|
| `pageId` valid pattern | `"[<pageId>] pageId '<value>' is invalid. Must match pattern ^[a-zA-Z0-9_-]+$ ..."` |
| `pageType` valid | `"[<pageId>] pageType '<value>' is invalid. Valid types: crud, dashboard"` |
| `fields[]` valid (lihat field validation) | Per-field error: lihat [`field-attributes.md`](./field-attributes.md) |
| `displayField` ada di `fields[]` | Warning: `"[<pageId>] displayField '<value>' is not found in the 'fields' definition"` |

### Pesan Validator `addVerb`, `pageSubject`, dan `showNewBadge`

Error yang menggagalkan validasi:

```
[<pageId>] addVerb must be a string, one of: Add, Create, Upload, Invite
[<pageId>] addVerb "<value>" is not allowed; must be one of: Add, Create, Upload, Invite
[<pageId>] pageSubject must be a string
[<pageId>] showNewBadge must be a boolean
```

Warning yang tidak menggagalkan validasi:

```
[<pageId>] pageSubject is empty; pageTitle will be used
[<pageId>] pageSubject "<value>" looks plural; use the singular object name (e.g. "Employee", not "Employees")
[<pageId>] pageSubject "<value>" contains "New"; the verb already implies a new entry
[<pageId>] add-flow label "<label>" exceeds 24 characters; shorten pageSubject so the button label stays readable
[<pageId>] showNewBadge is enabled but "created_at" is not declared in fields; the badge only renders when the list data carries created_at (add it to datatablesQuery on the backend payload)
```

Panjang label dihitung dari verba dan nama objek yang benar-benar dipakai, jadi halaman berjudul panjang tetap lolos dengan menyetel `pageSubject` yang lebih pendek. Warning `created_at` terbit hanya bila `showNewBadge: true` ditulis eksplisit di page.

### Pesan Validator `versionField`

Error yang menggagalkan validasi:

```
[<pageId>] versionField must be a non-empty string
```

Warning yang tidak menggagalkan validasi:

```
[<pageId>] versionField "<value>" is not declared in fields; the token is read from the /first response, make sure the backend selects it
[<pageId>] versionField "<value>" is the same as primaryKey; declare a separate concurrency token column
```

`versionField` tidak wajib ada di `fields[]` karena nilainya dibaca dari seluruh kolom respons `/first`, bukan dari input form, sehingga ketiadaannya hanya menghasilkan warning: pengingat agar backend benar-benar menyertakan kolom itu di response, bukan kesalahan pemakaian.

## `pageGroup` dan Auto-Derive Sidebar

Field `pageGroup` memungkinkan sidebar auto-derive (secara otomatis dibentuk dari semua page) tanpa menulis blok `navigation` manual di level root.

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
