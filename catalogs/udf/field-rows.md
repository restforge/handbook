# Grid Layout — `fieldRows`

> Pengaturan tata letak form dalam multi-kolom menggunakan CSS Grid.

## Konsep

Tanpa `fieldRows`, semua field di-render secara sekuensial (satu field per baris, full width). Dengan `fieldRows`, beberapa field dapat ditempatkan berdampingan dalam satu baris grid untuk menghemat ruang vertikal.

```json
"fieldRows": [
    {"fields": ["contact_name", "email"]},
    {"fields": ["phone", "contact_type"]},
    {"fields": ["address"]}
]
```

## Sintaks

| Properti | Tipe | Keterangan |
|----------|------|-----------|
| `fieldRows` | array | Array konfigurasi baris grid. Didefinisikan di level page (sibling dari `fields`) |
| `fieldRows[].fields` | array of string | Array nama field (merujuk ke `name` di `fields[]`) untuk satu baris |

## Jumlah Kolom Per Baris

Jumlah kolom ditentukan otomatis dari jumlah field dalam satu baris:

| Field Per Baris | CSS Grid | Hasil |
|----------------|----------|-------|
| 1 field | `repeat(1, 1fr)` | Full width |
| 2 field | `repeat(2, 1fr)` | 2 kolom sama lebar |
| 3 field | `repeat(3, 1fr)` | 3 kolom sama lebar |

Jumlah kolom dapat bervariasi antar baris dalam satu page:

```json
"fieldRows": [
    {"fields": ["full_name"]},
    {"fields": ["email", "phone"]},
    {"fields": ["city", "province", "postal_code"]},
    {"fields": ["address"]}
]
```

Hasil: 4 baris dengan layout 1 → 2 → 3 → 1 kolom.

## Visualisasi

**Tanpa `fieldRows` (sekuensial):**

```
┌─────────────────────────────────────┐
│ Nama Kontak  [___________________]  │
│ Email        [___________________]  │
│ Telepon      [___________________]  │
│ Tipe Kontak  [▼ dropdown_________]  │
│ Status       [● toggle ]            │
└─────────────────────────────────────┘
```

**Dengan `fieldRows` (grid 2 kolom):**

```
┌─────────────────────┬──────────────────────┐
│ Nama Kontak         │ Email                │
│ [________________]  │ [________________]   │
├─────────────────────┼──────────────────────┤
│ Telepon             │ Tipe Kontak          │
│ [________________]  │ [▼ dropdown______]   │
├─────────────────────┴──────────────────────┤
│ Status  [● toggle ]                        │
└────────────────────────────────────────────┘
```

## Bentuk Alternatif: Array of Strings atau Object

Setiap entry `fields` di dalam row dapat berupa string (nama field) atau object (untuk metadata tambahan). Validator menerima keduanya:

```json
"fieldRows": [
    {"fields": ["contact_name", "email"]},
    [
        {"name": "phone"},
        {"name": "city_id"}
    ]
]
```

> **Catatan:** Bentuk array-langsung tanpa wrapper `{"fields": [...]}` juga diurai oleh validator (untuk kompatibilitas mundur). Bentuk object dengan `fields` direkomendasikan untuk konsistensi.

## HTML yang Di-generate

Setiap row di-render sebagai `<div class="form-row">` dengan CSS grid inline:

```html
<div class="form-row" style="display: grid; grid-template-columns: repeat(2, 1fr); gap: 1rem;">
    <div class="form-group" id="form-group-contact-name">
        <!-- Field Nama Kontak -->
    </div>
    <div class="form-group" id="form-group-email">
        <!-- Field Email -->
    </div>
</div>
```

## Aturan `fieldRows`

| Aturan | Behavior |
|--------|----------|
| Nama field harus sesuai dengan `name` di `fields[]` | Warning jika tidak cocok: `"fieldRows references fields that are not defined in 'fields': <names>"` |
| Field tidak di-include tetap dirender setelah grid rows | Warning: `"The following fields are defined in 'fields' but are not referenced in 'fieldRows' and will not be rendered in the form: <names>. Add them to 'fieldRows' to make them visible, or set editorMode:'hidden' if they are intentionally excluded from the form."` |
| Field dengan `editorMode: "hidden"` dikecualikan dari warning orphan | Hidden field tidak perlu masuk fieldRows |
| Urutan tampilan field mengikuti `fieldRows` | Bukan urutan `fields[]` |
| `fieldRows` hanya memengaruhi form | Tabel list tidak terpengaruh |

## Interaksi dengan `features.fieldLayout`

Jika `features.fieldLayout: "horizontal"`, blok `fieldRows` diabaikan dan validator menghasilkan warning:

```
fieldRows is defined but will be ignored because features.fieldLayout is set to 'horizontal'
```

Layout `horizontal` (label di samping input) memakai single-column rendering, sehingga grid multi-kolom tidak berlaku.

## Best Practice

| Skenario | Rekomendasi |
|----------|-------------|
| Form dengan 1–4 field | `fieldRows` opsional. Tampilan sekuensial sudah cukup |
| Form dengan 5–10 field | Pakai `fieldRows` 2 kolom untuk mengurangi scroll |
| Form dengan 10+ field | Pakai `fieldRows` 2–3 kolom dan kelompokkan field terkait |
| Field `textarea` panjang | Tempatkan di baris sendiri (1 kolom) agar lebar maksimal |
| Field `checkbox` | Bisa di kolom yang sama dengan input pendek lain |

## Contoh Lengkap

```json
{
    "pageId": "contact",
    "fields": [
        {"name": "contact_name", "label": "Nama", "type": "text", "required": true},
        {"name": "email", "label": "Email", "type": "text"},
        {"name": "phone", "label": "Telepon", "type": "text"},
        {"name": "city_id", "label": "Kota", "type": "select", "dataSource": {"type": "api", "resource": "city"}},
        {"name": "address", "label": "Alamat", "type": "textarea", "rows": 4},
        {"name": "is_active", "label": "Status", "type": "checkbox", "defaultValue": true}
    ],
    "fieldRows": [
        {"fields": ["contact_name", "email"]},
        {"fields": ["phone", "city_id"]},
        {"fields": ["address"]},
        {"fields": ["is_active"]}
    ]
}
```

Hasil: form dengan 4 baris (2 kolom → 2 kolom → 1 kolom → 1 kolom).

---

← [`id-generation.md`](./id-generation.md) | [Selanjutnya: `features.md`](./features.md) →
