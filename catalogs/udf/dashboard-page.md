# Halaman Dashboard — `pageType: "dashboard"`

> Halaman grid widget (chart, KPI, list) yang membaca data dari endpoint API.

## Konsep

Halaman dashboard memiliki struktur yang berbeda dengan halaman CRUD. Tidak ada `fields[]`, `fieldRows`, atau `features`. Yang ada adalah:

| Komponen | Deskripsi |
|----------|-----------|
| `dataSources` | Mapping nama → konfigurasi fetch (URL, method, body) |
| `rows[]` | Layout grid responsif (mengikuti Bootstrap grid system) |
| `rows[].columns[]` | Kolom dalam satu row, masing-masing punya `colSpan` per breakpoint |
| `rows[].columns[].widgets[]` | Widget yang dirender di kolom: chart, KPI, list |

## Sintaks

```json
{
    "pageId": "overview",
    "pageType": "dashboard",
    "pageTitle": "Overview",
    "pageSubtitle": "Ringkasan operasional",
    "pageIcon": "bar-chart-2",
    "dataSources": {
        "revenueMonthly": {
            "url": "/api/dashboard/revenue-monthly",
            "method": "GET"
        }
    },
    "rows": [
        {
            "gap": "1rem",
            "columns": [
                {
                    "colSpan": {"base": 12, "md": 6, "lg": 4},
                    "widgets": [
                        {
                            "widgetId": "revenue-chart",
                            "widgetType": "column",
                            "title": "Revenue per Bulan",
                            "subtitle": "Year-to-date",
                            "dataSource": "revenueMonthly",
                            "chartEngine": "apexcharts",
                            "chart": {
                                "xField": "month",
                                "yField": "revenue"
                            }
                        }
                    ]
                }
            ]
        }
    ]
}
```

## Field yang Dilarang di Dashboard

Validator menolak field CRUD-only ketika `pageType: "dashboard"`:

```
apiPath, primaryKey, displayField, fields, fieldRows,
details, workflowActions, workflow, fieldStates, features
```

Jika salah satu ada di page dashboard, validator menghasilkan error:

```
[<pageId>] field '<name>' is not valid for pageType='dashboard' (CRUD-only field).
```

## Field yang Diizinkan di Dashboard

Field root yang valid di page dashboard:

```
dataSources, pageGroup, pageIcon, pageId, pageSubject,
pageSubtitle, pageTitle, pageType, rows
```

Field selain di atas akan menghasilkan warning:

```
[<pageId>] contains fields not recognized for pageType='dashboard': <names>.
These will be ignored by the generator.
```

## Aturan `pageId` Khusus Dashboard

`pageId` dashboard memakai pattern yang lebih ketat daripada CRUD:

```
^[a-z][a-z0-9-]*$
```

| Karakter Valid | Karakter Tidak Valid |
|----------------|---------------------|
| Lowercase letters (`a-z`) | Uppercase (`A-Z`) |
| Digits (`0-9`) | Underscore (`_`) |
| Hyphen (`-`) | Path separator (`/`, `\`) |

Underscore tidak diperbolehkan karena `pageId` dipakai sebagai filename HTML output, dan beberapa file system memperlakukan underscore khusus.

Pattern violation menghasilkan error:

```
[<pageId>] pageId '<value>' is invalid for pageType='dashboard'.
Must match pattern ^[a-z][a-z0-9-]*$
(lowercase letters, digits, hyphen; underscore not allowed because pageId becomes the output HTML filename).
```

## `dataSources` (Wajib)

Mapping nama → konfigurasi HTTP request. Setiap widget mereferensikan satu `dataSource` lewat namanya.

```json
"dataSources": {
    "salesByRegion": {
        "url": "/api/dashboard/sales-by-region",
        "method": "GET"
    },
    "topProducts": {
        "url": "/api/dashboard/top-products",
        "method": "POST",
        "body": {
            "limit": 10,
            "sortBy": "revenue"
        }
    }
}
```

### Properti per Entry

| Properti | Tipe | Default | Wajib | Keterangan |
|----------|------|---------|:-----:|-----------|
| `url` | string | — | ✓ | URL endpoint backend |
| `method` | string | `"GET"` | ✗ | HTTP method. Valid: `GET`, `POST` |
| `body` | object | — | ✗ | Request body untuk method `POST`. Diabaikan untuk `GET` |

### Aturan Validasi

| Aturan | Hasil |
|--------|-------|
| `dataSources` wajib non-empty | Error: `"[<pageId>] dataSources must be provided for pageType='dashboard'"` |
| `dataSources` harus object | Error: `"<path> must be an object (mapping name -> config), got <type>"` |
| Minimal 1 entry | Error: `"<path> must contain at least 1 entry for pageType='dashboard'"` |
| `url` wajib non-empty | Error: `"<path>.url must be a non-empty string"` |
| `method` valid (`GET` atau `POST`) | Error: `"<path>.method '<value>' is invalid. Valid methods: GET, POST"` |
| `body` harus object jika ada | Error: `"<path>.body must be an object, got <type>"` |
| `body` di method `GET` | Warning: `"<path>.body is set but method is 'GET'; body will be ignored by the runtime fetch."` |

## `rows[]` (Wajib)

Array baris layout. Setiap row berisi kolom-kolom dengan widget.

```json
"rows": [
    {
        "gap": "1rem",
        "columns": [
            {
                "colSpan": {"base": 12, "md": 6, "lg": 4},
                "widgets": [ /* ... */ ]
            },
            {
                "colSpan": {"base": 12, "md": 6, "lg": 8},
                "widgets": [ /* ... */ ]
            }
        ]
    }
]
```

### Properti Row

| Properti | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `gap` | string | ✗ | Spacing antar kolom (CSS-like, mis. `"1rem"`, `"16px"`) |
| `columns` | array | ✓ | Array kolom di row ini (minimal 1) |

### Properti Column

| Properti | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `colSpan` | object | ✓ | Lebar kolom per breakpoint Bootstrap |
| `widgets` | array | ✓ | Array widget di kolom ini (minimal 1) |

### `colSpan` per Breakpoint

Breakpoint yang dikenali (mengikuti Bootstrap 5):

```
base, sm, md, lg, xl, xxl
```

Range valid nilai per breakpoint: **1 sampai 12** (mengikuti grid 12 kolom Bootstrap).

```json
"colSpan": {
    "base": 12,
    "sm": 12,
    "md": 6,
    "lg": 4
}
```

Interpretasi: di layar `xs`/`sm` (kecil) full width (12 dari 12), di `md` setengah lebar (6), di `lg` ke atas sepertiga lebar (4).

#### Aturan Validasi `colSpan`

| Aturan | Hasil |
|--------|-------|
| `colSpan` wajib | Error: `"<path>.colSpan must be provided"` |
| Harus object, bukan array/string | Error: `"<path>.colSpan must be an object (breakpoint -> int), got <type>"` |
| Minimal 1 breakpoint | Error: `"<path>.colSpan must contain at least 1 breakpoint entry"` |
| Breakpoint valid | Error: `"<path>.colSpan.<bp> is not a valid breakpoint. Valid breakpoints: base, sm, md, lg, xl, xxl"` |
| Nilai integer 1–12 | Error: `"<path>.colSpan.<bp> must be between 1 and 12, got <value>"` |

## `widgets[]`

Tipe widget yang dikenali:

```
column, bar, pie, area, mini-bar, progress, list
```

| Tipe | Kategori | Butuh `chart` | Atribut Khusus |
|------|----------|:-------------:|----------------|
| `column` | Chart vertical bar | ✓ | `chart.xField`, `chart.yField` |
| `bar` | Chart horizontal bar | ✓ | `chart.xField`, `chart.yField` |
| `pie` | Chart pie/donut | ✓ | `chart.xField`, `chart.yField`, `chart.shape` |
| `area` | Chart area line | ✓ | `chart.xField`, `chart.yField` |
| `mini-bar` | Sparkline kecil | ✓ | `chart.xField`, `chart.yField` |
| `progress` | Progress bar/KPI | ✗ | — |
| `list` | List view ringkas | ✗ | — |

### Properti Umum Widget

| Properti | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `widgetId` | string | ✓ | Identifier unik per page. Pattern: `^[a-z][a-z0-9_-]*$` |
| `widgetType` | string | ✓ | Tipe widget (dari daftar valid) |
| `title` | string | ✓ | Judul widget |
| `subtitle` | string | ✗ | Subjudul di bawah title |
| `dataSource` | string | ✓ | Nama dataSource yang dirujuk (harus ada di `dataSources`) |
| `chart` | object | ✓ untuk chart widget | Konfigurasi chart (lihat sub-section) |
| `chartEngine` | string | ✓ untuk chart widget | Engine yang dipakai. Valid: `apexcharts`, `amcharts` |
| `display` | object | ✗ | Konfigurasi tampilan tambahan |

### Aturan `widgetId`

Pattern: `^[a-z][a-z0-9_-]*$`. Harus diawali huruf kecil, lanjut dengan huruf, digit, hyphen, atau underscore.

Validator menolak pattern lain:

```
<path>.widgetId '<value>' is invalid. Must match pattern ^[a-z][a-z0-9_-]*$
(lowercase letters, digits, hyphen, underscore; must start with a letter).
```

`widgetId` harus unik per page. Duplikat menghasilkan error:

```
<path>.widgetId '<value>' is duplicated; first defined at <previous-path>.
widgetId must be unique within a single dashboard page.
```

### Konfigurasi `chart` untuk Widget Chart

```json
{
    "widgetId": "revenue-chart",
    "widgetType": "column",
    "title": "Revenue per Bulan",
    "dataSource": "revenueMonthly",
    "chartEngine": "apexcharts",
    "chart": {
        "xField": "month",
        "yField": "revenue",
        "format": {
            "yAxis": {
                "prefix": "Rp ",
                "decimalPlaces": 0
            },
            "tooltip": {
                "prefix": "Rp ",
                "decimalPlaces": 2
            }
        }
    }
}
```

#### Atribut `chart`

| Atribut | Tipe | Wajib | Keterangan |
|---------|------|:-----:|-----------|
| `xField` | string | ✓ untuk `column`, `bar`, `pie`, `area` | Nama field di response untuk axis X / kategori pie |
| `yField` | string | ✓ untuk `column`, `bar`, `pie`, `area` | Nama field di response untuk axis Y / nilai pie |
| `shape` | string | ✓ untuk `pie` | Bentuk pie. Valid: `pie`, `donut`, `semi-pie`, `semi-donut` |
| `format` | object | ✗ | Konfigurasi format angka per location |

#### `chart.shape` untuk Pie Widget

Pie widget wajib menyertakan `chart.shape`:

```
pie, donut, semi-pie, semi-donut
```

`semi-pie` dan `semi-donut` tidak didukung native oleh ApexCharts. Jika `chartEngine: apexcharts` dipakai, validator menghasilkan warning dan runtime fallback ke `pie`/`donut`:

```
Widget '<id>' uses chart.shape='<shape>' with chartEngine='apexcharts'.
ApexCharts does not natively support semi-circle pie; runtime will fallback to '<fallback>'.
Use chartEngine='amcharts' for native semi-circle support.
```

#### `chart.format` (Format Angka per Location)

Format angka untuk 4 location yang berbeda di chart:

```
dataLabel, tooltip, xAxis, yAxis
```

Masing-masing location dapat memiliki 3 properti:

```
decimalPlaces, prefix, suffix
```

```json
"chart": {
    "format": {
        "yAxis": {
            "prefix": "Rp ",
            "decimalPlaces": 0
        },
        "tooltip": {
            "prefix": "Rp ",
            "suffix": " (YTD)",
            "decimalPlaces": 2
        }
    }
}
```

| Properti | Tipe | Range/Constraint |
|----------|------|------------------|
| `decimalPlaces` | integer | 0–20 |
| `prefix` | string | — |
| `suffix` | string | — |

#### Field Deprecated yang Dilarang

Field `chart` yang sudah di-remove di v5.x (akan menghasilkan error wajib migrasi):

| Field Lama | Field Baru |
|------------|-----------|
| `chart.library` | `chartEngine` (sibling dari `widgetType`) |
| `chart.orientation` | Pakai `widgetType: column` atau `widgetType: bar` |
| `chart.yAxisPrefix` | `chart.format.yAxis.prefix` |
| `chart.yAxisSuffix` | `chart.format.yAxis.suffix` |
| `chart.xAxisSuffix` | `chart.format.xAxis.suffix` |
| `chart.tooltipPrefix` | `chart.format.tooltip.prefix` |
| `chart.tooltipSuffix` | `chart.format.tooltip.suffix` |

### Widget Non-Chart

Widget `progress` dan `list` tidak memerlukan `chart`. Jika `chart` atau `chartEngine` diberikan, validator menghasilkan error:

```
Widget '<id>' (widgetType: <type>) does not support 'chartEngine'. Remove the field for non-chart widget types.
```

## Contoh Dashboard Lengkap

```json
{
    "pageId": "overview",
    "pageType": "dashboard",
    "pageTitle": "Sales Overview",
    "pageSubtitle": "Performance dashboard",
    "pageIcon": "bar-chart-2",
    "dataSources": {
        "kpi": {"url": "/api/dashboard/kpi", "method": "GET"},
        "revenueMonthly": {"url": "/api/dashboard/revenue-monthly", "method": "GET"},
        "topProducts": {"url": "/api/dashboard/top-products", "method": "POST", "body": {"limit": 5}}
    },
    "rows": [
        {
            "gap": "1rem",
            "columns": [
                {
                    "colSpan": {"base": 12, "md": 4},
                    "widgets": [
                        {
                            "widgetId": "kpi-revenue",
                            "widgetType": "progress",
                            "title": "Total Revenue",
                            "dataSource": "kpi"
                        }
                    ]
                },
                {
                    "colSpan": {"base": 12, "md": 8},
                    "widgets": [
                        {
                            "widgetId": "revenue-chart",
                            "widgetType": "column",
                            "title": "Revenue per Bulan",
                            "dataSource": "revenueMonthly",
                            "chartEngine": "apexcharts",
                            "chart": {
                                "xField": "month",
                                "yField": "revenue",
                                "format": {
                                    "yAxis": {"prefix": "Rp ", "decimalPlaces": 0}
                                }
                            }
                        }
                    ]
                }
            ]
        },
        {
            "gap": "1rem",
            "columns": [
                {
                    "colSpan": {"base": 12, "md": 6},
                    "widgets": [
                        {
                            "widgetId": "top-products",
                            "widgetType": "list",
                            "title": "Top 5 Products",
                            "dataSource": "topProducts"
                        }
                    ]
                },
                {
                    "colSpan": {"base": 12, "md": 6},
                    "widgets": [
                        {
                            "widgetId": "sales-share",
                            "widgetType": "pie",
                            "title": "Sales by Region",
                            "dataSource": "topProducts",
                            "chartEngine": "amcharts",
                            "chart": {
                                "xField": "region",
                                "yField": "revenue",
                                "shape": "donut"
                            }
                        }
                    ]
                }
            ]
        }
    ]
}
```

---

← [`live-sync.md`](./live-sync.md) | [Selanjutnya: `naming-convention.md`](./naming-convention.md) →
