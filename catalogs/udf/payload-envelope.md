# Struktur Top-Level (Payload Envelope)

> Bentuk root JSON UDF: field yang dikenali loader sebelum validasi page-level.

## Bentuk Root JSON

UDF adalah dokumen JSON tunggal di level root dengan envelope berikut:

```json
{
    "extends": "app-config.json",
    "appConfig": { /* ... */ },
    "pages": [ /* ... */ ],
    "navigation": { /* ... */ },
    "homepage": "contact",
    "liveSync": { /* ... */ }
}
```

Field yang dikenali secara langsung oleh loader UDF:

| Field | Tipe | Wajib | Keterangan |
|-------|------|-------|-----------|
| `extends` | string | ✗ | Path ke file UDF base (relatif terhadap file ini) untuk inheritance config |
| `appConfig` | object | ✓ | Konfigurasi level aplikasi (lihat [`app-config.md`](./app-config.md)) |
| `pages` | array | ✓ | Array page object atau include reference (minimal 1 entry) |
| `navigation` | object | ✗ | Konfigurasi sidebar manual (lihat [`navigation.md`](./navigation.md)) |
| `homepage` | string | ✗ | `pageId` default ketika URL root diakses (lihat [`homepage.md`](./homepage.md)) |
| `liveSync` | object | ✗ | Konfigurasi WebSocket Live Sync (lihat [`live-sync.md`](./live-sync.md)) |

Field selain di atas diteruskan sebagai metadata tambahan dan dapat dibaca oleh plugin tertentu (mis. plugin `vanilla-js-auth` membaca field `auth` di level root).

## Entry `pages[]`

Setiap entry di array `pages` dapat berbentuk dua bentuk berbeda:

### Bentuk 1: Inline Page Object

Page object ditulis langsung di dalam array:

```json
{
    "pages": [
        {
            "pageId": "contact",
            "pageTitle": "Contact",
            "apiPath": "contact",
            "primaryKey": "contact_id",
            "displayField": "contact_name",
            "fields": [ /* ... */ ]
        }
    ]
}
```

### Bentuk 2: Include Reference

Page object disimpan di file eksternal dan direferensikan dari array `pages` file utama:

```json
{
    "pages": [
        { "include": "pages/contact.json" },
        { "include": "pages/city.json" }
    ]
}
```

**File yang disertakan wajib berupa fragment UDF yang memuat array `pages`**, bukan object page polos. Saat load, loader membaca file yang ditunjuk, **mengekstrak array `pages` di dalamnya**, lalu menyisipkan (append) page-page tersebut menggantikan entry `include`. Path bersifat relatif terhadap file UDF utama.

Contoh isi `pages/contact.json` yang benar (objek page dibungkus array `pages`):

```json
{
    "pages": [
        {
            "pageId": "contact",
            "pageTitle": "Contact",
            "apiPath": "contact",
            "primaryKey": "contact_id",
            "displayField": "contact_name",
            "fields": [ /* ... */ ]
        }
    ]
}
```

Karena loader mengekstrak array `pages`, satu file include dapat memuat lebih dari satu page (beberapa object di dalam array `pages`), dan resolusi bersifat rekursif (file include boleh memuat entry `include` lain di array `pages`-nya). File include yang tidak memiliki array `pages` ditolak dengan error `Included file '<path>' does not contain a valid 'pages' array`.

Dua bentuk dapat dicampur dalam satu array `pages`:

```json
{
    "pages": [
        { "include": "pages/contact.json" },
        {
            "pageId": "city",
            "pageTitle": "City",
            "apiPath": "city",
            "fields": [ /* ... */ ]
        }
    ]
}
```

## Inheritance via `extends`

Field `extends` memungkinkan pemecahan `appConfig` (dan field root lain) ke file base yang dapat dipakai bersama oleh banyak UDF. Pattern ini berguna untuk aplikasi multi-module yang memiliki konfigurasi `appName`, `apiBaseUrl`, dan `plugin` yang sama.

**File base (`app-config.json`):**

```json
{
    "appConfig": {
        "appName": "My Application",
        "appCode": "my-app",
        "plugin": "vanilla-js-basic",
        "apiBaseUrl": "http://localhost:3031/api/my-app",
        "port": 3000
    }
}
```

**File UDF yang extends:**

```json
{
    "extends": "app-config.json",
    "pages": [
        {
            "pageId": "contact",
            "pageTitle": "Contact",
            "apiPath": "contact",
            "primaryKey": "contact_id",
            "displayField": "contact_name",
            "fields": [ /* ... */ ]
        }
    ]
}
```

### Perilaku Merge

| Properti | Sumber Final | Logika |
|----------|--------------|--------|
| `appConfig` | File base | Diambil seluruhnya dari file yang diwarisi |
| `pages` | File payload | Diambil dari file payload — tidak digabungkan dengan base |
| Properti root lain (mis. `auth`, `navigation`) | Gabungan | Field di file payload menimpa field bernama sama di base |

Path `extends` bersifat relatif terhadap lokasi file payload. Jika `app-config.json` berada di folder yang sama dengan UDF utama, cukup tulis nama file saja.

## Validasi Envelope

Aturan validasi yang langsung mengevaluasi envelope:

| Aturan | Kondisi Error |
|--------|---------------|
| `appConfig` wajib | `"appConfig is not defined in the payload"` |
| `pages` wajib non-empty | `"pages is not defined or the array is empty"` |
| `pages` harus array | `"pages must be an array containing at least 1 page"` |

Jika `liveSync` didefinisikan tetapi tidak ada page yang mengaktifkan `features.enableLiveSync`, validator akan menampilkan warning. Sebaliknya, jika ada page dengan `features.enableLiveSync: true` tetapi blok `liveSync` tidak didefinisikan, validator menghasilkan error per page.

---

← [`README`](./README.md) | [Selanjutnya: `app-config.md`](./app-config.md) →
