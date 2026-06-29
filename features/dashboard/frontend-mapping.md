# Pemetaan ke Widget Frontend

Dokumen ini menjelaskan cara memetakan response dashboard ke pola card dan chart yang umum di template admin modern (Metronic, AdminLTE, Tabler). Setiap contoh menyertakan payload dan response yang dibutuhkan frontend untuk me-render card atau chart tersebut.

---

## Statistics Card

Tiga pola statistics card: card berwarna dengan subtitle list, card minimalis dengan angka besar, dan card pastel dengan progress bar.

### Card Berwarna dengan Subtitle List Dinamis

Card jenis ini menampilkan judul tebal dan subtitle berupa daftar kategori yang bersumber dari database. Gunakan widget single-query yang menghasilkan array nama.

Contoh markup frontend:

```html
<a class="card bg-danger">
  <i class="ki-outline ki-basket text-white fs-2x"></i>
  <div class="text-white fw-bold fs-2">Shopping Cart</div>
  <div class="text-white"><!-- dari items.map(i => i.name).join(', ') --></div>
</a>
```

**Payload widget:**

```json
{
  "id": "shopping_categories",
  "query": "file:query/dashboard-stats/shopping-categories.sql"
}
```

**SQL:**

```sql
SELECT name FROM product_category WHERE is_active = 'true' ORDER BY name LIMIT 4;
```

**Response:**

```json
"shopping_categories": {
  "items": [
    { "name": "Lands"  },
    { "name": "Houses" },
    { "name": "Ranchos"},
    { "name": "Farms"  }
  ]
}
```

Frontend: `items.map(i => i.name).join(', ')`. Judul card, warna background (`bg-danger`), dan icon ditentukan frontend.

> **Perlu diketahui:** Bila subtitle bersifat statis dan tidak berubah berdasarkan data, dashboard tidak diperlukan. Frontend cukup render dari konfigurasi UI lokal.

---

### Card Minimalis dengan Angka Besar

Card jenis ini menampilkan satu angka besar sebagai headline metric. Pakai widget multi-query dengan satu key agar response berupa scalar primitive — lebih ergonomis untuk card scalar daripada single-query yang selalu membungkus `{ items: [...] }`.

Contoh markup frontend:

```html
<a class="card bg-body">
  <i class="ki-outline ki-chart-simple text-primary fs-2x"></i>
  <div class="text-gray-900 fw-bold fs-2"><!-- dari value, diformat oleh frontend --></div>
  <div class="text-gray-400">SAP UI Progress</div>
</a>
```

**Payload widget:**

```json
{
  "id": "total_revenue",
  "queries": {
    "value": "file:query/dashboard-stats/total-revenue.sql"
  }
}
```

**SQL:**

```sql
SELECT COALESCE(SUM(total_amount), 0) AS value
FROM stock_outbound
WHERE status != 'cancelled';
```

**Response:**

```json
"total_revenue": { "value": "500000000" }
```

Frontend memformat `500000000` menjadi `500M` atau `500,000,000` menggunakan `Intl.NumberFormat`. Icon dan label ditentukan frontend.

**Variasi dengan trend chip:**

```json
{
  "id": "total_revenue",
  "queries": {
    "value": "file:query/dashboard-stats/total-revenue.sql",
    "trend": "file:query/dashboard-stats/total-revenue-trend.sql"
  }
}
```

Response: `{ "value": "500000000", "trend": { "direction": "up", "pct": "12.5" } }`

---

### Card Pastel dengan Progress Bar

Card jenis ini menampilkan persentase dan progress bar terhadap target. Memetakan langsung ke [Pola 3: Metric + Progress to Goal](widget-patterns.md#pola-3-metric--progress-to-goal-scalar--object--scalar-target).

Contoh markup frontend:

```html
<div class="card bg-light-success">
  <span class="text-success fs-5">Project Progress</span>
  <span class="text-gray-900 fs-1 fw-bold"><!-- dari pct, dihitung frontend --></span>
  <div class="progress h-7px bg-success bg-opacity-50">
    <div class="progress-bar bg-success" style="width: /* pct */%"></div>
  </div>
</div>
```

**Payload widget:**

```json
{
  "id": "project_progress",
  "queries": {
    "value":  "file:query/dashboard-stats/project-progress-current.sql",
    "target": "file:query/dashboard-stats/project-progress-target.sql"
  }
}
```

**Response:**

```json
"project_progress": { "value": "50", "target": "100" }
```

Frontend menghitung `pct = Math.round(parseFloat(value) / parseFloat(target) * 100)`, render `{pct}%` sebagai display dan `style="width: {pct}%"` untuk progress bar. Warna card, judul, dan label ditentukan frontend.

---

## Mixed Widget

Empat pola mixed widget: circular progress dengan notes, gauge dengan multi-series, headline dengan line chart timeseries, dan pastel dengan sparkline.

### Card Action dengan Circular Progress dan Notes

Card jenis ini menampilkan lingkaran progress besar dan catatan teks di bawahnya.

**Payload widget — tiga key terpisah:**

```json
{
  "id": "profile_setup_progress",
  "queries": {
    "value":  "file:query/dashboard-stats/profile-setup-current.sql",
    "target": "file:query/dashboard-stats/profile-setup-target.sql",
    "notes":  "file:query/dashboard-stats/profile-setup-notes.sql"
  }
}
```

**SQL `profile-setup-notes.sql`:**

```sql
SELECT announcement_text AS notes
FROM sprint_announcements
WHERE is_active = true
ORDER BY created_at DESC
LIMIT 1;
```

**Response:**

```json
"profile_setup_progress": {
  "value":  "74",
  "target": "100",
  "notes":  "Current sprint requires stakeholders to approve newly amended policies"
}
```

Frontend menghitung `pct = Math.round(parseFloat(value) / parseFloat(target) * 100)`, me-render chart circular progress, dan menempel teks `notes` di bawah chart. Bila `notes` kosong, frontend menyembunyikan blok notes.

**Variasi — gabung ke satu SQL bila notes dan progress berasal dari sumber yang sama:**

```json
{
  "id": "profile_setup_progress",
  "queries": {
    "summary": "file:query/dashboard-stats/profile-setup-summary.sql"
  }
}
```

SQL menghasilkan 1 row × 3 col (`value`, `target`, `notes`) dan collapse ke object:

```json
"profile_setup_progress": {
  "summary": { "value": "74", "target": "100", "notes": "Current sprint..." }
}
```

---

### Card Gauge dengan Total dan Multi-Series

Card jenis ini menampilkan total scalar di tengah gauge (half-circle) dan array series untuk dua atau lebih arc.

**Payload widget:**

```json
{
  "id": "user_base",
  "queries": {
    "value": "file:query/dashboard-stats/user-base-total.sql",
    "items": "file:query/dashboard-stats/user-base-series.sql"
  }
}
```

**SQL `user-base-total.sql`:**

```sql
SELECT COUNT(*) AS value FROM customer;
```

**SQL `user-base-series.sql`:**

```sql
SELECT 'Active'  AS label, COUNT(*) AS value FROM customer WHERE status = 'active'
UNION ALL
SELECT 'Pending' AS label, COUNT(*) AS value FROM customer WHERE status = 'pending';
```

**Response:**

```json
"user_base": {
  "value": "8346",
  "items": [
    { "label": "Active",  "value": "5230" },
    { "label": "Pending", "value": "3116" }
  ]
}
```

Frontend me-render half-circle gauge dengan dua arc dari `items` dan menempel `value` sebagai angka tengah. Nilai `8346` berasal dari SQL terpisah sehingga frontend tidak perlu menjumlah ulang `items`.

> **Perlu diketahui:** Warna per arc, label legend, dan teks deskriptif card ditentukan frontend. Backend hanya menyediakan angka dan label kategori.

---

### Card Headline dengan Line Chart Timeseries

Card jenis ini menampilkan headline value besar dan line chart panjang dengan tooltip per periode.

**Payload widget:**

```json
{
  "id": "reports_revenue",
  "queries": {
    "value":  "file:query/dashboard-stats/reports-revenue-value.sql",
    "points": "file:query/dashboard-stats/reports-revenue-points.sql"
  }
}
```

**SQL `reports-revenue-points.sql` (contoh bulanan, PostgreSQL):**

```sql
SELECT
  to_char(date_trunc('month', tx_date), 'Mon') AS period,
  COALESCE(SUM(amount), 0)                      AS value
FROM transaction
WHERE tx_date >= date_trunc('year', CURRENT_DATE)
GROUP BY date_trunc('month', tx_date)
ORDER BY date_trunc('month', tx_date);
```

> **Perlu diketahui:** `to_char(..., 'Mon')` menghasilkan label manusiawi (`Jan`, `Feb`, ...) yang langsung bisa dipakai sebagai label axis. Untuk MySQL gunakan `DATE_FORMAT(tx_date, '%b')`.

**Response:**

```json
"reports_revenue": {
  "value": "24500",
  "points": [
    { "period": "Jan", "value": "12000" },
    { "period": "Feb", "value": "18500" },
    { "period": "Mar", "value": "15300" },
    { "period": "Apr", "value": "22000" },
    { "period": "May", "value": "40000" }
  ]
}
```

Frontend memformat `24500` menjadi `$24,500` dan memetakan `points` ke series chart. Format tooltip (`May - Net Profit: $40 thousands`) ditentukan library chart di frontend.

---

### Card Pastel dengan Headline dan Sparkline

Card jenis ini identik dengan [Pola 2: Metric + Sparkline](widget-patterns.md#pola-2-metric--sparkline-scalar--object--timeseries). Payload dan response sama; perbedaan hanya pada rendering frontend (background pastel, sparkline minimalis di belakang angka).

**Payload widget:**

```json
{
  "id": "earnings",
  "queries": {
    "value":  "file:query/dashboard-stats/earnings-value.sql",
    "trend":  "file:query/dashboard-stats/earnings-trend.sql",
    "points": "file:query/dashboard-stats/earnings-points.sql"
  }
}
```

**Response:**

```json
"earnings": {
  "value": "560",
  "trend": { "direction": "up", "pct": "28" },
  "points": [
    { "period": "2026-04-24", "value": "480" },
    { "period": "2026-04-25", "value": "510" },
    { "period": "2026-04-30", "value": "560" }
  ]
}
```

Frontend memformat `560` menjadi `$560`, me-render `+28% this week` dari `trend`, dan render sparkline dari `points.map(p => parseFloat(p.value))`. Background pastel dan judul card ditentukan frontend.

---

## Multi-Series Chart

Tiga opsi pemetaan untuk chart yang menampilkan beberapa series pada axis kategori yang sama: clustered column, stacked column, grouped bar, dan multi-line chart.

### Opsi 1: Wide Format via SQL Pivot (Direkomendasikan)

Wide format cocok bila daftar series diketahui pada saat design time (misalnya daftar kontinen yang tetap). SQL melakukan pivot dengan `CASE WHEN ... GROUP BY` sehingga response berupa wide array yang bisa langsung di-feed ke `chart.data`.

**Payload widget:**

```json
{
  "id": "population_by_continent",
  "query": "file:query/dashboard-stats/population-by-continent.sql"
}
```

**SQL (PostgreSQL):**

```sql
SELECT
  year                                                                   AS category,
  COALESCE(SUM(CASE WHEN continent = 'Europe'        THEN value END), 0) AS europe,
  COALESCE(SUM(CASE WHEN continent = 'Asia'          THEN value END), 0) AS asia,
  COALESCE(SUM(CASE WHEN continent = 'Africa'        THEN value END), 0) AS africa,
  COALESCE(SUM(CASE WHEN continent = 'North America' THEN value END), 0) AS north_america,
  COALESCE(SUM(CASE WHEN continent = 'South America' THEN value END), 0) AS south_america
FROM population
GROUP BY year
ORDER BY year;
```

**Response:**

```json
"population_by_continent": {
  "items": [
    { "category": "2003", "europe": "2.5", "asia": "2.1", "africa": "0.1", "north_america": "2.5", "south_america": "0.3" },
    { "category": "2004", "europe": "2.6", "asia": "2.2", "africa": "0.1", "north_america": "2.7", "south_america": "0.3" }
  ]
}
```

**Konsumsi di frontend (amCharts v5):**

```js
chart.data = response.data.population_by_continent.items.map(row => ({
  ...row,
  europe:        parseFloat(row.europe),
  asia:          parseFloat(row.asia),
  africa:        parseFloat(row.africa),
  north_america: parseFloat(row.north_america),
  south_america: parseFloat(row.south_america)
}));

const europeSeries = chart.series.push(...);
europeSeries.dataFields.valueY    = "europe";
europeSeries.dataFields.categoryX = "category";
europeSeries.name = "Europe"; // label manusiawi → frontend concern
```

> **Perlu diketahui:** Key snake_case dari SQL (misalnya `north_america`) menjadi key JSON yang stabil. Pemetaan dari `north_america` ke label `"North America"` adalah tanggung jawab frontend, bukan backend.

---

### Opsi 2: Long Format Tanpa Pivot

Long format cocok bila daftar series bersifat dinamis dari database (misalnya jumlah kontinen bisa bertambah atau berkurang). SQL menghasilkan satu row per kombinasi `(category, series)`.

**Payload widget:**

```json
{
  "id": "population_long",
  "query": "file:query/dashboard-stats/population-long.sql"
}
```

**SQL:**

```sql
SELECT
  year       AS category,
  continent  AS series,
  SUM(value) AS value
FROM population
GROUP BY year, continent
ORDER BY year, continent;
```

**Response:**

```json
"population_long": {
  "items": [
    { "category": "2003", "series": "Europe", "value": "2.5" },
    { "category": "2003", "series": "Asia",   "value": "2.1" },
    { "category": "2004", "series": "Europe", "value": "2.6" },
    { "category": "2004", "series": "Asia",   "value": "2.2" }
  ]
}
```

**Pre-process di frontend ke wide format:**

```js
const items = response.data.population_long.items;
const wide = Object.values(items.reduce((acc, row) => {
  if (!acc[row.category]) acc[row.category] = { category: row.category };
  acc[row.category][row.series] = parseFloat(row.value);
  return acc;
}, {}));
chart.data = wide;
```

> **Perlu diketahui:** Library seperti ECharts dan AntV/G2 dapat mengonsumsi long format secara native melalui konfigurasi `seriesField`, sehingga pre-process tidak selalu diperlukan.

---

### Opsi 3: Multi-Query per Series

Multi-query per series cocok bila tiap series berasal dari SQL atau tabel yang berbeda, atau tiap series membutuhkan parameter yang berbeda.

**Payload widget:**

```json
{
  "id": "sales_by_channel",
  "queries": {
    "online":  "file:query/dashboard-stats/sales-online-monthly.sql",
    "offline": "file:query/dashboard-stats/sales-offline-monthly.sql",
    "agent":   "file:query/dashboard-stats/sales-agent-monthly.sql"
  }
}
```

**Contoh SQL `sales-online-monthly.sql` (PostgreSQL):**

```sql
SELECT
  to_char(date_trunc('month', tx_date), 'YYYY-MM') AS period,
  COALESCE(SUM(amount), 0)                          AS value
FROM online_orders
WHERE tx_date BETWEEN :from AND :to
GROUP BY date_trunc('month', tx_date)
ORDER BY date_trunc('month', tx_date);
```

**Response:**

```json
"sales_by_channel": {
  "online":  [{ "period": "2026-01", "value": "12000" }, { "period": "2026-02", "value": "14500" }],
  "offline": [{ "period": "2026-01", "value": "8000"  }, { "period": "2026-02", "value": "7800"  }],
  "agent":   [{ "period": "2026-01", "value": "3500"  }, { "period": "2026-02", "value": "4100"  }]
}
```

**Zip di frontend ke wide format:**

```js
const data = response.data.sales_by_channel;
const periods = [...new Set(Object.values(data).flat().map(p => p.period))].sort();
chart.data = periods.map(period => ({
  category: period,
  online:  parseFloat(data.online.find(p => p.period === period)?.value  ?? 0),
  offline: parseFloat(data.offline.find(p => p.period === period)?.value ?? 0),
  agent:   parseFloat(data.agent.find(p => p.period === period)?.value   ?? 0)
}));
```

---

### Tabel Perbandingan Tiga Opsi

| Skenario | Opsi | Shape response | Pre-process frontend |
|----------|------|----------------|----------------------|
| Series tetap, satu sumber data | Wide pivot | `{ items: [{ category, sA, sB }] }` | Tidak perlu |
| Series dinamis dari database | Long | `{ items: [{ category, series, value }] }` | Pivot, atau library native |
| Series dari tabel berbeda | Multi-query | `{ sA: [{ period, value }], sB: [...] }` | Zip per kategori |

**Catatan untuk timeseries multi-series:**

Shape response untuk clustered column chart identik dengan stacked column dan multi-line chart. Yang berbeda hanya konfigurasi visual frontend (`stacked: true`, orientasi vertikal/horizontal). Backend tidak perlu menyesuaikan response untuk perbedaan visual tersebut.

---

## Pemisahan Concern

Backend dashboard hanya menyediakan data layer. Semua aspek visual ditentukan di frontend.

| Concern | Pihak | Contoh |
|---------|-------|--------|
| SQL aggregasi dan nilai data | Backend | `SELECT SUM(amount) FROM sales GROUP BY country` |
| Validasi parameter request | Backend | Type checking, required, default value |
| Widget id dan shape response | Backend | `author_sales`, `expected_earnings` |
| Tipe chart | Frontend | Donut, pie, bar, area, sparkline, gauge |
| Layout card | Frontend | Grid, kolom, responsif |
| Warna, judul, subtitle | Frontend | Theming, i18n |
| Format angka display | Frontend | `500M`, `+28%`, locale |
| Width progress bar | Frontend | `style="width: {pct}%"` |
| Konfigurasi library chart | Frontend | Animasi, smooth curve, tooltip, legend |

**Self-check sebelum menambah field ke payload backend:**

1. Apakah field ini berubah bila theme dashboard ganti dark mode? Bila YA, ini frontend concern.
2. Apakah field ini berubah bila bahasa UI di-translate? Bila YA, ini frontend concern.
3. Apakah field ini berubah bila platform render berbeda (mobile vs desktop vs PDF)? Bila YA, ini frontend concern.

Bila tiga pertanyaan di atas dijawab TIDAK, field tersebut boleh masuk ke payload backend.

> **Perlu diketahui:** Semua nilai numeric dikembalikan sebagai string di response (misalnya `"value": "500000000"`). Frontend wajib `parseFloat()` atau `Number()` sebelum operasi aritmatika atau sebelum feed ke library chart yang mengharapkan tipe number.
