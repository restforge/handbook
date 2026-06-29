# `npx restforge-designer catalog`

> Emit catalog UDF machine-readable (JSON) dari konstanta validator. Berisi daftar nilai valid (field types, enum) dan aturan struktural (required fields, range) yang dipakai validator, sebagai acuan otoritatif saat menyusun payload UDF.

## Sintaks

```bash
npx restforge-designer catalog [OPTIONS]
```

## Flag

| Flag | Tipe | Wajib | Keterangan |
|------|------|:-----:|-----------|
| `--section <SECTION>` | enum | ✗ | Section catalog yang dikeluarkan. Nilai: `fields`, `navigation`, `app-config`, `dashboard`, `all`. Default: `all` |
| `--plugins-dir <PATH>` | string | ✗ | Flag global (diwarisi semua verb); tidak berpengaruh pada output catalog karena catalog bersumber dari konstanta statis |

## Output

Verb mencetak **pretty JSON ke stdout**. Sumber data adalah konstanta validator yang sama yang dipakai verb `validate`, sehingga catalog tidak pernah menyimpang dari aturan validasi aktual.

Top-level catalog (`--section=all`):

| Key | Isi |
|-----|-----|
| `udfCatalogVersion` | Versi kontrak catalog (integer). Naik saat ada perubahan breaking pada struktur catalog |
| `fields` | Whitelist tipe field, editor mode, dan opsi id-generation, plus badge color |
| `navigation` | Aturan navigasi: tipe item, kedalaman maksimum, pattern `pageId` |
| `appConfig` | Required fields `appConfig` + range `port` dan `numberFormat.decimalPlaces` |
| `dashboard` | Opsi halaman dashboard: tipe widget, chart engine, pie shape, colspan, allowed/forbidden/deprecated fields |

## Section

| `--section` | Top-level key output | Kegunaan |
|-------------|----------------------|----------|
| `fields` | `fields` | Saat menentukan `type` field dan opsi id-generation |
| `navigation` | `navigation` | Saat menyusun `navigation` (depth, item types) |
| `app-config` | `appConfig` | Saat mengisi `appConfig` (field wajib, range port) |
| `dashboard` | `dashboard` | Saat menyusun halaman `pageType: dashboard` (widget, chart) |
| `all` (default) | semua key di atas | Acuan lengkap |

## Contoh Penggunaan

### Catalog Penuh

```bash
npx restforge-designer catalog
```

Output (dipotong):

```json
{
  "udfCatalogVersion": 1,
  "fields": {
    "validFieldTypes": ["checkbox", "date", "number", "select", "text", "textarea", "time", "timestamp"],
    "validEditorModes": ["hidden", "readonly"],
    "validIdgenFieldTypes": ["number", "text", "textarea"],
    "validIdgenModes": ["code", "number", "pin", "random", "serial"],
    "validIdgenFormats": ["text", "yyyy", "yyyymm", "yyyymmdd"],
    "validBadgeColors": ["danger", "info", "primary", "success", "warning"]
  },
  "navigation": {
    "validNavItemTypes": ["group", "page", "separator"],
    "navMaxDepth": 3,
    "pageGroupMaxLevels": 2,
    "pageIdPattern": "^[a-zA-Z0-9_-]+$"
  },
  "appConfig": { "...": "..." },
  "dashboard": { "...": "..." }
}
```

### Satu Section

```bash
npx restforge-designer catalog --section=app-config
```

```json
{
  "appConfig": {
    "requiredFields": ["appName", "appCode", "plugin", "apiBaseUrl"],
    "port": { "min": 1, "max": 65535 },
    "numberFormat": { "decimalPlaces": { "min": 0, "max": 20 } }
  }
}
```

```bash
npx restforge-designer catalog --section=dashboard
```

```json
{
  "dashboard": {
    "pageTypes": ["crud", "dashboard"],
    "widgetTypes": ["column", "bar", "pie", "area", "mini-bar", "progress", "list"],
    "chartEngines": ["amcharts", "apexcharts"],
    "pieShapes": ["donut", "pie", "semi-donut", "semi-pie"],
    "widgetsRequiringFields": ["column", "bar", "pie", "area"],
    "widgetsWithoutChart": ["progress", "list"],
    "dataSourceMethods": ["GET", "POST"],
    "colspan": { "breakpoints": ["base", "sm", "md", "lg", "xl", "xxl"], "min": 1, "max": 12 },
    "allowedPageFields": ["dataSources", "pageGroup", "pageIcon", "pageId", "pageSubject", "pageSubtitle", "pageTitle", "pageType", "rows"],
    "forbiddenFields": ["apiPath", "primaryKey", "displayField", "fields", "fieldRows", "details", "workflowActions", "workflow", "fieldStates", "features"],
    "deprecatedFields": ["chart.library", "chart.orientation"]
  }
}
```

## Behavior

Verb `catalog`:

1. **Membangun catalog** dari konstanta validator (`fields`, `navigation`, `appConfig`, `dashboard`).
2. **Memfilter** ke satu section bila `--section` selain `all` diberikan.
3. **Mencetak** pretty JSON ke stdout. Tidak membaca payload, tidak menulis file.

## Exit Code

| Exit Code | Kondisi |
|-----------|---------|
| `0` | Catalog berhasil dicetak |
| `2` | Nilai `--section` tidak valid (ditolak oleh parser argumen) |

## Hubungan dengan `catalogs/udf/`

Catalog verb ini adalah representasi **machine-readable** dari aturan yang dijelaskan secara naratif di [`catalogs/udf/`](../../catalogs/udf/). Keduanya bersumber dari validator yang sama:

| Section catalog | Dokumentasi prosa |
|-----------------|-------------------|
| `fields` | [`catalogs/udf/field-types.md`](../../catalogs/udf/field-types.md), [`field-attributes.md`](../../catalogs/udf/field-attributes.md), [`id-generation.md`](../../catalogs/udf/id-generation.md) |
| `navigation` | [`catalogs/udf/navigation.md`](../../catalogs/udf/navigation.md) |
| `appConfig` | [`catalogs/udf/app-config.md`](../../catalogs/udf/app-config.md) |
| `dashboard` | [`catalogs/udf/dashboard-page.md`](../../catalogs/udf/dashboard-page.md) |
| (aturan validasi) | [`catalogs/udf/validation-rules.md`](../../catalogs/udf/validation-rules.md) |

## Use Case

| Skenario | Rekomendasi |
|----------|-------------|
| Butuh daftar nilai valid sebelum menulis UDF (programatik/tooling) | `catalog` |
| Butuh penjelasan naratif lengkap sebuah aturan | `catalogs/udf/` |
| Cek apakah sebuah UDF konkret valid | `validate` |

---

**Lihat juga**: [`README`](./README.md) · [`validate.md`](./validate.md) · [`catalogs/udf/`](../../catalogs/udf/)
