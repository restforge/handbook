# Aturan Validasi UDF

> Ringkasan lengkap aturan validasi yang dievaluasi oleh `rfd validate` dan `rfd generate`.

## Konteks

Validator UDF (source: `validator/payload_validator.rs`) dievaluasi pada beberapa entry point:

| Command | Behavior |
|---------|----------|
| `rfd validate --payload <file>` | Validasi standalone, output error/warning ke console |
| `rfd preview --payload <file>` | Validasi sebelum preview daftar file |
| `rfd generate --payload <file>` | Validasi sebelum write file. Error membatalkan generate |

Validator membedakan dua tingkat keparahan:

| Tingkat | Behavior |
|---------|----------|
| **Error** | Membatalkan operasi. UDF dianggap invalid |
| **Warning** | Tidak membatalkan. Memberi peringatan untuk konsistensi atau best practice |

## Scope Validasi: `app` vs `form`

Generator mendukung dua scope yang mempengaruhi behavior validasi:

| Scope | Pemakaian |
|-------|-----------|
| `app` (default) | Validasi penuh: semua page, navigation, homepage |
| `form` | Validasi terfokus pada satu page. `pageRef`/`homepage` orphan jadi warning, bukan error |

Scope `form` mendukung workflow generate per-page tanpa harus regenerate seluruh aplikasi.

## Validasi Envelope (Root)

| Aturan | Tingkat | Pesan |
|--------|---------|-------|
| `appConfig` ada | Error | `appConfig is not defined in the payload` |
| `appConfig.appName` non-empty | Error | `appConfig.appName must be provided` |
| `appConfig.appCode` non-empty | Error | `appConfig.appCode must be provided` |
| `appConfig.plugin` non-empty | Error | `appConfig.plugin must be provided` |
| `appConfig.apiBaseUrl` non-empty | Error | `appConfig.apiBaseUrl must be provided` |
| `appConfig.port` integer 1–65535 jika ada | Error | `appConfig.port must be an integer between 1 and 65535, got <value>` |
| `appConfig.numberFormat` object jika ada | Error | `appConfig.numberFormat must be an object when provided, got <type>` |
| `appConfig.numberFormat.decimalPlaces` integer 0–20 | Error | `appConfig.numberFormat.decimalPlaces must be an integer between 0 and 20, got <value>` |
| `pages` ada non-empty | Error | `pages is not defined or the array is empty` |
| `pages` harus array | Error | `pages must be an array containing at least 1 page` |

## Validasi `liveSync`

| Aturan | Tingkat | Pesan |
|--------|---------|-------|
| `liveSync.url` non-empty jika blok ada | Error | `liveSync.url must be a non-empty string` |
| `liveSync.url` scheme `ws://`/`wss://` | Error | `liveSync.url must start with 'ws://' or 'wss://', got: '<value>'` |
| `liveSync.apiKey` non-empty jika blok ada | Error | `liveSync.apiKey must be a non-empty string` |
| Blok ada tetapi tidak ada page yang aktifkan | Warning | `liveSync block is defined but no page has features.enableLiveSync enabled` |
| Page aktifkan tetapi blok tidak ada | Error per page | `[<pageId>] features.enableLiveSync is true but top-level 'liveSync' block is not defined` |

## Validasi Page

### Page Umum

| Aturan | Tingkat | Pesan |
|--------|---------|-------|
| `pageId` pattern valid | Error | `[<pageId>] pageId '<value>' is invalid. Must match pattern ^[a-zA-Z0-9_-]+$ ...` |
| `pageType` valid (`crud` atau `dashboard`) | Error | `[<pageId>] pageType '<value>' is invalid. Valid types: crud, dashboard` |
| `pageGroup` array of string | Error | `[<pageId>] pageGroup must be an array of strings, got <type>` |
| `pageGroup` elemen string non-empty | Error | `[<pageId>] pageGroup[<i>] must be a non-empty string` |
| `pageGroup` maks 2 level | Error | `[<pageId>] pageGroup has <n> levels but the maximum allowed is 2 ...` |
| `pageGroup` casing collision | Warning | `pageGroup label '<dup>' differs from previously seen '<first>' only by case ...` |
| `navigation` ada + `pageGroup` ada | Warning | `pageGroup is defined on the following pages but will be ignored because manual 'navigation' block is present: <ids> ...` |

### Page CRUD: `fields[]`

| Aturan | Tingkat | Pesan |
|--------|---------|-------|
| `name` wajib | Error | `Field does not have a 'name' property` |
| `label` wajib | Error | `Field '<name>' does not have a 'label' property` |
| `type` wajib | Error | `Field '<name>' does not have a 'type' property` |
| `type` valid (8 tipe) | Error | `Field '<name>' has an invalid type: '<value>'. Valid types are: checkbox, date, number, select, text, textarea, time, timestamp` |
| `editorMode` valid (`hidden`/`readonly`) | Error | `Field '<name>' has an invalid editorMode: '<value>'. Valid modes are: hidden, readonly` |
| `editorMode: hidden` tanpa `defaultValue` | Warning | `Field '<name>' uses editorMode:'hidden' but has no defaultValue` |
| `inTable: true` tanpa `tableOrder` | Warning | `Field '<name>' has inTable:true but no tableOrder is defined` |
| `type: select` tanpa `dataSource` | Warning | `Field '<name>' is of type 'select' but has no dataSource` |

### Page CRUD: `displayField`

| Aturan | Tingkat | Pesan |
|--------|---------|-------|
| `displayField` ada di `fields[]` | Warning | `[<pageId>] displayField '<value>' is not found in the 'fields' definition` |

### Page CRUD: `fieldRows[]`

| Aturan | Tingkat | Pesan |
|--------|---------|-------|
| `fieldRows` di-ignore jika `fieldLayout: horizontal` | Warning | `fieldRows is defined but will be ignored because features.fieldLayout is set to 'horizontal'` |
| Field di `fields[]` tidak direferensikan | Warning | `The following fields are defined in 'fields' but are not referenced in 'fieldRows' and will not be rendered in the form: <names>. Add them to 'fieldRows' to make them visible, or set editorMode:'hidden' if they are intentionally excluded from the form.` |
| `fieldRows` referensi field yang tidak ada | Warning | `fieldRows references fields that are not defined in 'fields': <names>` |

### Page CRUD: `details[]`

| Aturan | Tingkat | Pesan |
|--------|---------|-------|
| `detailId` wajib | Error | `[<pageId>].details[<i>] detailId must be provided` |
| `primaryKey` wajib | Error | `[<pageId>].details[<detailId>] primaryKey must be provided` |
| `fields` wajib non-empty | Error | `[<pageId>].details[<detailId>] fields must be provided and cannot be empty` |
| Field per detail wajib `name`/`label`/`type` | Error | `[<pageId>].details[<detailId>] field '<name>' has an invalid type: '<value>'` |

### Page CRUD: `workflowActions[]`

| Aturan | Tingkat | Pesan |
|--------|---------|-------|
| `workflow.statusField` wajib jika `workflowActions[]` ada | Error | `[<pageId>] workflow.statusField must be provided when workflowActions is defined` |
| `actionId` wajib | Error | `[<pageId>].workflowActions[<i>] actionId must be provided` |
| `actionId` unik per page | Error | `[<pageId>].workflowActions[<i>] actionId '<value>' is duplicated` |
| `label` wajib | Error | `[<pageId>].workflowActions[<i>] label must be provided` |
| `api` wajib | Error | `[<pageId>].workflowActions[<i>] api must be provided` |
| `api.endpoint` wajib | Error | `[<pageId>].workflowActions[<i>] api.endpoint must be provided` |
| `confirm.summary[].field` ada di `fields[]` | Warning | `[<pageId>].workflowActions[<i>] confirm.summary references field '<name>' which is not found in 'fields'` |

### Page CRUD: `fieldStates[]`

| Aturan | Tingkat | Pesan |
|--------|---------|-------|
| `when` wajib | Error | `[<pageId>].fieldStates[<i>] when must be provided` |
| `state` valid (`readonly`/`hidden`) | Error | `[<pageId>].fieldStates[<i>] state is invalid: '<value>'. Use 'readonly' or 'hidden'` |

### Page CRUD: `defaultValue.source: "idgen"`

| Aturan | Tingkat | Pesan |
|--------|---------|-------|
| Field type valid untuk idgen | Error | `Field '<name>' cannot use IdGen defaultValue with type '<type>'. Supported types: number, text, textarea` |
| `mode` wajib | Error | `Field '<name>' defaultValue.mode must be provided when source is 'idgen'` |
| `mode` valid (5 mode) | Error | `Field '<name>' defaultValue.mode is invalid: '<value>'. Valid modes: code, number, pin, random, serial` |
| `resource` wajib + tidak `:` | Error | `Field '<name>' defaultValue.resource must be provided when source is 'idgen'` / `Field '<name>' defaultValue.resource is invalid: must be a string and must not contain ':'` |
| `mode: number` butuh `format` valid | Error | `Field '<name>' defaultValue.format must be provided when mode is 'number'` / `Field '<name>' defaultValue.format is invalid: '<value>'. Valid formats: text, yyyy, yyyymm, yyyymmdd` |
| `mode: number` butuh `numDigits` 1–12 | Error | `Field '<name>' defaultValue.numDigits must be an integer between 1 and 12` |
| `mode: number` `separator` tidak `:` | Error | `Field '<name>' defaultValue.separator must not contain ':'` |
| `mode: pin` `digits` 4–12 | Error | `Field '<name>' defaultValue.digits must be an integer between 4 and 12` |
| `mode: code/serial/random` butuh `pattern` non-empty | Error | `Field '<name>' defaultValue.pattern must be a non-empty string when mode is '<mode>'` |
| `reserve: true` hanya `mode: number` | Error | `Field '<name>' defaultValue.reserve is only supported when mode is 'number'` |
| `reserve: true` butuh `ttl` >= 10 | Error | `Field '<name>' defaultValue.ttl must be an integer >= 10 when reserve is true` |
| Maks 1 field `reserve: true` per page | Error | `Only one field per page may use defaultValue.reserve=true. Found: <names>` |

## Validasi `navigation`

| Aturan | Tingkat | Pesan |
|--------|---------|-------|
| `navigation` object | Error | `navigation must be an object, got <type>` |
| `navigation.items` array | Error | `navigation.items must be an array of navigation items` |
| `type` valid (`group`/`page`/`separator`) | Error | `<path>.type is invalid: '<value>'. Valid types: group, page, separator` |
| `badgeColor` valid jika ada | Error | `<path>.badgeColor is invalid: '<value>'. Valid colors: danger, info, primary, success, warning` |
| `icon` di depth > 1 | Warning | `<path>.icon is set at depth <n> but icons are only rendered at depth 1; this icon will be ignored.` |
| `pageRef` wajib (`type: page`) | Error | `<path>.pageRef must be provided when type='page'` |
| `pageRef` ada di `pages[]` (scope `app`) | Error | `<path>.pageRef '<value>' does not match any pageId in 'pages'. Add the page or fix the reference.` |
| `pageRef` orphan (scope `form`) | Warning | `<path>.pageRef '<value>' does not match any pageId in 'pages'. Sidebar will filter at runtime based on existing output files.` |
| `label` wajib (`type: group`) | Error | `<path>.label must be provided when type='group'` |
| `children` non-empty (`type: group`) | Error | `<path>.children must be a non-empty array when type='group'` |
| Depth maks 3 | Error | `<path> exceeds the maximum nesting depth of 3. Flatten the structure or remove the deepest group.` |

## Validasi `homepage`

| Aturan | Tingkat | Pesan |
|--------|---------|-------|
| String non-empty jika ada | Error | `homepage must be a non-empty string (pageRef) when provided` |
| Ada di `pages[]` (scope `app`) | Error | `homepage '<value>' does not match any pageId in 'pages'. Add the page or fix the reference.` |
| Orphan (scope `form`) | Warning | `homepage '<value>' does not match any pageId in 'pages'. It will resolve at runtime if the page is generated later in a separate session.` |
| `pageType` halaman target valid | Error | `homepage '<value>' references a page with unsupported pageType '<type>'. Supported types: crud, dashboard.` |

## Validasi Dashboard

### Field Wajib dan Forbidden

| Aturan | Tingkat | Pesan |
|--------|---------|-------|
| `pageId` valid pattern dashboard | Error | `[<pageId>] pageId '<value>' is invalid for pageType='dashboard'. Must match pattern ^[a-z][a-z0-9-]*$ ...` |
| `pageTitle` non-empty | Error | `[<pageId>] pageTitle must be a non-empty string` |
| CRUD-only field di dashboard | Error | `[<pageId>] field '<name>' is not valid for pageType='dashboard' (CRUD-only field).` |
| Field tidak dikenali | Warning | `[<pageId>] contains fields not recognized for pageType='dashboard': <names>. These will be ignored by the generator.` |

CRUD-only field yang ditolak di dashboard:

```
apiPath, primaryKey, displayField, fields, fieldRows,
details, workflowActions, workflow, fieldStates, features
```

### `dataSources`

| Aturan | Tingkat | Pesan |
|--------|---------|-------|
| `dataSources` wajib | Error | `[<pageId>] dataSources must be provided for pageType='dashboard'` |
| Harus object | Error | `<path> must be an object (mapping name -> config), got <type>` |
| Minimal 1 entry | Error | `<path> must contain at least 1 entry for pageType='dashboard'` |
| Key non-empty | Error | `<path> key must be a non-empty string, got '<value>'` |
| Setiap config harus object | Error | `<path>.<key> must be an object, got <type>` |
| `url` wajib non-empty | Error | `<path>.url must be a non-empty string` |
| `method` valid (`GET`/`POST`) | Error | `<path>.method '<value>' is invalid. Valid methods: GET, POST` |
| `body` object jika ada | Error | `<path>.body must be an object, got <type>` |
| `body` di `GET` | Warning | `<path>.body is set but method is 'GET'; body will be ignored by the runtime fetch.` |

### `rows[]`

| Aturan | Tingkat | Pesan |
|--------|---------|-------|
| `rows` wajib | Error | `[<pageId>] rows must be provided for pageType='dashboard'` |
| Array | Error | `<path> must be an array, got <type>` |
| Minimal 1 row | Error | `<path> must contain at least 1 row for pageType='dashboard'` |
| Setiap row object | Error | `<path>[<i>] must be an object, got <type>` |
| `gap` string jika ada | Error | `<path>.gap must be a string when provided` |
| `columns` wajib | Error | `<path>.columns must be provided` |
| `columns` array non-empty | Error | `<path>.columns must contain at least 1 column` |
| `colSpan` wajib | Error | `<path>.colSpan must be provided` |
| `colSpan` object | Error | `<path>.colSpan must be an object (breakpoint -> int), got <type>` |
| Breakpoint valid | Error | `<path>.colSpan.<bp> is not a valid breakpoint. Valid breakpoints: base, sm, md, lg, xl, xxl` |
| Nilai colSpan integer 1–12 | Error | `<path>.colSpan.<bp> must be between 1 and 12, got <value>` |
| `widgets` wajib non-empty | Error | `<path>.widgets must be provided` / `<path>.widgets must contain at least 1 widget` |

### `widgets[]`

| Aturan | Tingkat | Pesan |
|--------|---------|-------|
| `widgetId` wajib non-empty | Error | `<path>.widgetId must be provided` |
| `widgetId` valid pattern | Error | `<path>.widgetId '<value>' is invalid. Must match pattern ^[a-z][a-z0-9_-]*$ ...` |
| `widgetId` unik per page | Error | `<path>.widgetId '<value>' is duplicated; first defined at <previous-path>...` |
| `widgetType` wajib | Error | `<path>.widgetType must be provided` |
| `widgetType` valid (`column`/`bar`/`pie`/`area`/`mini-bar`/`progress`/`list`) | Error | `<path>.widgetType '<value>' is invalid. Valid types: ...` |
| `widgetType: donut` (removed) | Error | `Widget '<id>' uses removed widgetType 'donut'. Use widgetType 'pie' with chart.shape='donut' instead.` |
| `title` non-empty | Error | `<path>.title must be a non-empty string` |
| `subtitle` string jika ada | Error | `<path>.subtitle must be a string when provided, got <type>` |
| `dataSource` wajib | Error | `<path>.dataSource must be provided` |
| `dataSource` cocok di `dataSources` | Error | `<path>.dataSource '<value>' is not found in page.dataSources. Available: <list>` |
| `chart` wajib untuk chart widget | Error | `<path>.chart must be provided for widgetType='<type>'` |
| `chartEngine` wajib chart widget | Error | `Widget '<id>' (widgetType: <t>) requires field 'chartEngine'. Allowed values: 'apexcharts', 'amcharts'.` |
| `chartEngine` valid | Error | `Widget '<id>' has invalid chartEngine '<value>'. Allowed values: 'apexcharts', 'amcharts'.` |
| `chartEngine` tidak boleh di non-chart widget | Error | `Widget '<id>' (widgetType: <t>) does not support 'chartEngine'. Remove the field for non-chart widget types.` |
| `chart.library` (removed) | Error | `Widget '<id>' uses deprecated field 'chart.library'. Move the value to top-level 'chartEngine' field.` |
| `chart.orientation` (removed) | Error | `Widget '<id>' uses deprecated field 'chart.orientation'. Use widgetType 'column' for vertical bars or 'bar' for horizontal bars instead.` |
| Deprecated chart fields F5.5 | Error | `Widget '<id>' uses removed chart field 'chart.<field>'. Use 'chart.format.<location>.<part>' instead.` |
| `chart.xField`/`yField` wajib (`column`/`bar`/`pie`/`area`) | Error | `<path>.chart.xField must be provided for widgetType='<type>'` |
| `chart.shape` wajib untuk `pie` | Error | `Widget '<id>' (widgetType: pie) requires chart.shape. Allowed values: ['donut', 'pie', 'semi-donut', 'semi-pie'].` |
| `chart.shape` valid | Error | `Widget '<id>' has invalid chart.shape '<value>'. Allowed values: ...` |
| `chart.shape` semi-circle dengan apexcharts | Warning | `Widget '<id>' uses chart.shape='<shape>' with chartEngine='apexcharts'. ApexCharts does not natively support semi-circle pie; runtime will fallback to '<fallback>'...` |
| `chart.shape` di non-pie widget | Error | `Widget '<id>' (widgetType: <t>) does not support 'chart.shape'. Field is only valid for widgetType 'pie'.` |
| `chart.format` di non-chart widget | Error | `Widget '<id>' (widgetType: <t>) does not support 'chart.format'. Remove the field for non-chart widget types.` |
| `chart.format` object jika ada | Error | `<path>.chart.format must be an object when provided, got <type>` |
| `chart.format.<loc>` valid (`dataLabel`/`tooltip`/`xAxis`/`yAxis`) | Warning | `<path>.chart.format.<loc> is an unknown format location and will be ignored. Allowed locations: dataLabel, tooltip, xAxis, yAxis` |
| `chart.format.<loc>.decimalPlaces` integer 0–20 | Error | `<path>.chart.format.<loc>.decimalPlaces must be an integer between 0 and 20, got <value>` |
| `chart.format.<loc>.prefix`/`suffix` string | Error | `<path>.chart.format.<loc>.<key> must be a string when provided, got <type>` |
| `display` object jika ada | Error | `<path>.display must be an object when provided, got <type>` |

## Validasi Scope

| Aturan | Tingkat | Pesan |
|--------|---------|-------|
| `scope` valid (`app`/`form`) | Error | `scope is invalid: '<value>'. Use 'app' or 'form'` |
| `page_id` wajib jika `scope=form` | Error | `page_id must be provided when scope is 'form'` |
| `page_id` ada di `pages[]` jika `scope=form` | Error | `page_id '<value>' is not found in the payload. Available page IDs: <list>` |

## Tip Debugging Validasi

1. **Pakai `rfd validate` standalone** sebelum `generate` untuk dapatkan output validation lebih jelas
2. **Validasi error muncul dengan prefix `[<pageId>]`** untuk page-level error, atau `<path>` (mis. `pages[0].fields[2]`) untuk granular
3. **Warning tidak membatalkan** generate, tetapi sebaiknya diselesaikan untuk konsistensi
4. **Error muncul di urutan struktural**, bukan severity. Field error mungkin muncul setelah error lain di section yang sama
5. **Tidak ada validasi sintaks ekspresi `when`** di `fieldStates[]`. Pastikan sintaks JavaScript valid manual

---

← [`naming-convention.md`](./naming-convention.md) | [Selanjutnya: `complete-examples.md`](./complete-examples.md) →
