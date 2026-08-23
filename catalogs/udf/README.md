# UDF — UI Definition File

> Kebutuhan halaman frontend yang akan di-generate menjadi aplikasi web statis.

## Ikhtisar

**RESTForge Designer** menyediakan generator yang membaca UDF dan menghasilkan aplikasi web (HTML, JavaScript, CSS) siap pakai. UDF bertanggung jawab atas empat hal:

1. Mendeklarasikan metadata aplikasi (`appName`, `appCode`, `apiBaseUrl`, plugin target)
2. Mendeklarasikan halaman CRUD yang berisi form, table, toolbar, master-detail, dan workflow action
3. Mendeklarasikan halaman dashboard yang berisi grid widget chart dan KPI
4. Mendeklarasikan struktur sidebar navigation dan homepage default

Satu UDF dapat menampung satu aplikasi multi-page. Path fisik konvensi: `ui-payload/<aplikasi>.json` atau pemecahan multi-file via `extends` dan `pages[].include`.

UDF bekerja sebagai input untuk generator. Hasil generation adalah file HTML/JS/CSS yang dapat langsung dilayani oleh HTTP server statis tanpa build step tambahan.

### Karakteristik UDF

| Aspek | Penjelasan |
|-------|-----------|
| App-as-config | Seluruh aplikasi didefinisikan secara deklaratif di file JSON, bukan ditulis ulang per halaman |
| Rujukan tunggal per aplikasi | Satu file UDF (atau satu set file dengan `extends`) menjadi referensi untuk seluruh aplikasi frontend |
| Plugin-driven generator | Output bergantung pada plugin target (`vanilla-js-basic`, `vanilla-js-auth`, dll.) yang dinyatakan di `appConfig.plugin` |
| Convention over configuration | Default value otomatis untuk `port`, `fieldLayout`, `pageType`, sehingga payload minimal sudah valid |
| Modular page | Halaman bisa disertakan via `{"include": "path"}` untuk pemecahan file per modul |
| Validasi formal | UDF divalidasi di tahap `validate`, `preview`, dan `generate` (lihat [`validation-rules.md`](./validation-rules.md)) |

### Lingkup yang TIDAK Dicakup

- Definisi struktur tabel database (ditangani oleh SDF di [`../sdf/`](../sdf/))
- Definisi endpoint REST API backend (ditangani oleh RDF di [`../rdf/`](../rdf/))
- Custom business logic non-CRUD di backend (ditangani oleh processor RDF, lihat [`../rdf/processor.md`](../rdf/processor.md))
- Logic JavaScript custom selain yang di-generate oleh plugin

## Struktur File UDF

Aplikasi frontend dideklarasikan dalam satu file JSON utama. Nama file mengikuti nama aplikasi (kebab-case direkomendasikan).

```
ui-payload/
├── contact-mgmt.json
├── app-config.json              (file base untuk extends)
└── pages/
    ├── contact.json
    └── city.json
```

Setiap file UDF adalah dokumen JSON valid dengan struktur object di level root:

```json
{
    "appConfig": {
        "appName": "Contact Management",
        "appCode": "contact-mgmt",
        "plugin": "vanilla-js-basic",
        "apiBaseUrl": "http://localhost:3031/api/dbcontact",
        "port": 3000
    },
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

Folder `pages/` adalah konvensi untuk menyimpan file page eksternal yang direferensikan via `{"include": "<path>"}` dari array `pages` file utama. Setiap file eksternal tersebut wajib berupa fragment UDF yang memuat array `pages` (bukan object page polos) — lihat [`payload-envelope.md`](./payload-envelope.md).

## Anatomi UDF

Daftar field top-level yang dikenali oleh generator dan validator, dikelompokkan berdasarkan tingkat kepentingan.

### Field Wajib

| Field | Tipe | Keterangan |
|-------|------|-----------|
| `appConfig` | object | Konfigurasi level aplikasi (lihat [`app-config.md`](./app-config.md)) |
| `pages` | array | Array halaman yang akan di-generate (minimal 1 page) |

### Field Opsional Umum

| Field | Tipe | Keterangan |
|-------|------|-----------|
| `extends` | string | Path ke file UDF base yang akan di-extend (lihat [`payload-envelope.md`](./payload-envelope.md)) |
| `navigation` | object | Konfigurasi sidebar navigation manual (override auto-derive dari `pageGroup`) |
| `homepage` | string | `pageId` yang dijadikan default ketika URL root diakses |
| `liveSync` | object | Konfigurasi WebSocket Live Sync (lihat [`live-sync.md`](./live-sync.md)) |

### Page Object — Field per Halaman

Setiap entry di `pages` adalah object yang mendeskripsikan satu halaman. Tipe halaman ditentukan oleh field `pageType`:

| `pageType` | Keterangan | Field utama |
|------------|-----------|-------------|
| `"crud"` (default) | Halaman list + form CRUD | `fields`, `fieldRows`, `features`, `details`, `workflowActions` |
| `"dashboard"` | Halaman grid widget chart/KPI | `dataSources`, `rows` |

Lihat [`page-anatomy.md`](./page-anatomy.md) untuk field lengkap per halaman CRUD, dan [`dashboard-page.md`](./dashboard-page.md) untuk halaman dashboard.

## Navigasi

### Konfigurasi Aplikasi

| Topik | File |
|-------|------|
| Struktur Top-Level (`extends`, `pages`, include) | [`payload-envelope.md`](./payload-envelope.md) |
| Konfigurasi Aplikasi (`appConfig`) | [`app-config.md`](./app-config.md) |
| Anatomi Halaman CRUD (`pages[]`) | [`page-anatomy.md`](./page-anatomy.md) |
| Anatomi Halaman Dashboard (`pageType: "dashboard"`) | [`dashboard-page.md`](./dashboard-page.md) |

### Field dan Layout

| Topik | File |
|-------|------|
| Tipe Field (`fields[].type`) | [`field-types.md`](./field-types.md) |
| Atribut Field (umum + per tipe) | [`field-attributes.md`](./field-attributes.md) |
| Sumber Data Select (`dataSource`) | [`data-source.md`](./data-source.md) |
| ID Generation (`defaultValue.source: "idgen"`) | [`id-generation.md`](./id-generation.md) |
| Grid Layout (`fieldRows`) | [`field-rows.md`](./field-rows.md) |
| Fitur Halaman (`features`) | [`features.md`](./features.md) |

### Pola Lanjutan

| Topik | File |
|-------|------|
| Master Detail (`details[]`) | [`master-detail.md`](./master-detail.md) |
| Workflow Actions (`workflow` + `workflowActions[]`) | [`workflow-actions.md`](./workflow-actions.md) |
| Field States (`fieldStates[]`) | [`field-states.md`](./field-states.md) |
| Sidebar Navigation Manual (`navigation`) | [`navigation.md`](./navigation.md) |
| Homepage Default (`homepage`) | [`homepage.md`](./homepage.md) |
| Live Sync via WebSocket (`liveSync`) | [`live-sync.md`](./live-sync.md) |

### Konvensi dan Validasi

| Topik | File |
|-------|------|
| Konvensi Penamaan | [`naming-convention.md`](./naming-convention.md) |
| Aturan Validasi UDF | [`validation-rules.md`](./validation-rules.md) |
| Contoh Lengkap | [`complete-examples.md`](./complete-examples.md) |

## Generator CLI

Command CLI yang berinteraksi dengan UDF disediakan oleh dua binary terpisah:

**Binary `restforge-designer` (frontend):** generate, validasi, dan setup aplikasi frontend dari UDF.

| Verb | Tujuan | Halaman Command |
|------|--------|-----------------|
| `restforge-designer init` | Scaffold project baru dari plugin scaffold | [`commands/restforge-frontend/init.md`](../../commands/restforge-frontend/init.md) |
| `restforge-designer validate` | Validasi UDF terhadap schema plugin | [`commands/restforge-frontend/validate.md`](../../commands/restforge-frontend/validate.md) |
| `restforge-designer preview` | Preview daftar file yang akan di-generate tanpa write ke disk | [`commands/restforge-frontend/preview.md`](../../commands/restforge-frontend/preview.md) |
| `restforge-designer generate` | Generate aplikasi frontend dari UDF | [`commands/restforge-frontend/generate.md`](../../commands/restforge-frontend/generate.md) |
| `restforge-designer plugins list` | Daftar plugin yang ter-install | [`commands/restforge-frontend/plugins-list.md`](../../commands/restforge-frontend/plugins-list.md) |
| `restforge-designer plugins inspect` | Detail satu plugin | [`commands/restforge-frontend/plugins-inspect.md`](../../commands/restforge-frontend/plugins-inspect.md) |

**Binary `restforge` (backend):** konversi RDF → UDF sebagai starting point UDF baru.

| Verb | Tujuan | Halaman Command |
|------|--------|-----------------|
| `npx restforge payload migrate` | Konversi payload backend (RDF) menjadi payload frontend (UDF) | [`commands/restforge-backend/payload/migrate.md`](../../commands/restforge-backend/payload/migrate.md) |

> **Catatan:** Konversi RDF → UDF dilakukan oleh binary backend (`restforge`), bukan oleh binary frontend (`restforge-designer`). Alasan: konversi memerlukan akses ke konfigurasi database backend (mis. `SERVER_PORT`, `SERVER_ADDRESS`) yang lebih natural ditangani di sisi backend.

---

**Lihat juga**: [`catalogs/`](../) · [`README`](../../README.md)
