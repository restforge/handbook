# Pola Widget

Widget dashboard menghasilkan response berdasarkan aturan deterministik yang ditentukan oleh struktur payload. Payload tidak mendeklarasikan `type` widget. Bentuk response ditentukan oleh pilihan antara `query` dan `queries`, dan setiap key di `queries` dapat menetapkan bentuknya sendiri lewat `shape`.

---

## Aturan Pemetaan Payload ke Response

| Payload | Response |
|---------|----------|
| `query` (singular) | Selalu dibungkus `{ items: [...] }` apapun shape SQL result |
| `queries.X` (apapun nama key X) | Menjadi key `X` di response, dengan aturan scalar collapse atau sesuai `shape` key tersebut |

### Aturan Scalar Collapse

Berlaku untuk setiap key di `queries`. Widget `query` singular tidak mengikuti aturan ini karena selalu menghasilkan `{ items: [...] }`.

| SQL result shape | Response yang dihasilkan |
|------------------|--------------------------|
| 1 row × 1 column | Scalar primitive — misalnya `"value": "69700"` |
| 1 row × multi column | Object — misalnya `"trend": { "direction": "up", "pct": "2.2" }` |
| N rows | Array of objects — misalnya `"items": [...]`, `"points": [...]` |

Aturan ini membuat bentuk key ikut berubah mengikuti jumlah baris. Key daftar seperti `items` berupa array saat datanya tiga baris, tetapi berubah menjadi object saat datanya tinggal satu baris. Widget daftar di frontend lalu tidak menampilkan apa pun. Untuk key seperti ini, tetapkan bentuknya dengan `shape`.

### Menetapkan Bentuk Key dengan `shape`

Nilai `queries.<key>` boleh ditulis sebagai object `{ "file": "...", "shape": "..." }`. Field `file` berisi file reference yang sama dengan bentuk string, sedangkan `shape` menetapkan bentuk key tersebut di response tanpa bergantung pada jumlah baris. Bentuk string tetap berarti `shape: "auto"`, sehingga payload lama menghasilkan response yang sama.

| `shape` | 0 row | 1 row | Lebih dari 1 row |
|---------|-------|-------|------------------|
| `auto` (default) | `[]` | Scalar atau object sesuai aturan scalar collapse | Array of objects |
| `array` | `[]` | Array berisi 1 object | Array of objects |
| `object` | `null` | Object | Widget menjadi `{ "error": "..." }` |
| `scalar` | `null` | Nilai kolom, bila hasilnya 1 column | Widget menjadi `{ "error": "..." }` |

Untuk `shape: "scalar"`, hasil 1 row dengan lebih dari 1 column juga membuat widget menjadi `{ "error": "..." }`. Pesan error menyebut key dan jumlah row atau column yang dihasilkan SQL, misalnya `queries.trend expects shape 'object' (at most 1 row), but the query returned 3 rows`. Widget lain tetap berjalan normal.

Contoh berikut menampilkan daftar order pending sekaligus jumlah totalnya dalam satu widget.

```json
{
  "id": "pending_orders",
  "queries": {
    "value": "file:query/dashboard-overview/pending-count.sql",
    "items": { "file": "file:query/dashboard-overview/pending-orders.sql", "shape": "array" }
  }
}
```

Saat order pending tinggal satu, response `items` tetap berupa array seperti berikut.

```json
"pending_orders": {
  "value": "1",
  "items": [
    { "title": "ORD-0012", "description": "Budi Santoso", "value": "450000" }
  ]
}
```

Validator menolak nilai `shape` di luar `auto`, `array`, `object`, dan `scalar`, juga field selain `file` dan `shape` di dalam object tersebut.

### Contoh Lengkap

**Widget single query:**

```json
{ "id": "sales_by_city", "query": "file:query/sales-by-city.sql" }
```

SQL menghasilkan N rows × 2 col. Response:

```json
"sales_by_city": {
  "items": [
    { "category": "Jakarta",  "value": "288000000" },
    { "category": "Surabaya", "value": "229600000" }
  ]
}
```

**Widget multi query:**

```json
{
  "id": "expected_earnings",
  "queries": {
    "value": "file:query/earnings-value.sql",
    "trend": "file:query/earnings-trend.sql",
    "items": "file:query/earnings-breakdown.sql"
  }
}
```

- `earnings-value.sql` menghasilkan 1 row × 1 col → scalar
- `earnings-trend.sql` menghasilkan 1 row × 2 col → object
- `earnings-breakdown.sql` menghasilkan N rows → array

Response:

```json
"expected_earnings": {
  "value": "69700",
  "trend": { "direction": "up", "pct": "2.2" },
  "items": [
    { "label": "Shoes",  "value": "7660" },
    { "label": "Gaming", "value": "2820" }
  ]
}
```

---

## Pola 1: Metric + Breakdown (Scalar + Object + Array)

Pola ini menghasilkan headline metric, trend chip, dan breakdown per kategori. Cocok untuk widget KPI seperti total penjualan dengan rincian per produk atau per region.

**Payload widget:**

```json
{
  "id": "expected_earnings",
  "queries": {
    "value": "file:query/dashboard-ecommerce/earnings-value.sql",
    "trend": "file:query/dashboard-ecommerce/earnings-trend.sql",
    "items": { "file": "file:query/dashboard-ecommerce/earnings-breakdown.sql", "shape": "array" }
  }
}
```

**Shape SQL per key:**

| Key | Shape | Hasil collapse |
|-----|-------|----------------|
| `value` | 1 row × 1 col | Scalar primitive |
| `trend` | 1 row × 2 col (`direction`, `pct`) | Object |
| `items` | N rows × 2 col (`label`, `value`) | Array of objects, tetap array untuk 0 atau 1 row karena `shape: "array"` |

**Contoh SQL `earnings-value.sql` (PostgreSQL):**

```sql
SELECT COALESCE(SUM(amount), 0) AS value
FROM sales_order
WHERE status != 'cancelled'
  AND order_date BETWEEN :from AND :to;
```

**Contoh SQL `earnings-trend.sql` (PostgreSQL):**

```sql
SELECT
  CASE WHEN curr > prev THEN 'up' ELSE 'down' END          AS direction,
  ROUND(ABS(curr - prev) / NULLIF(prev, 0) * 100, 1)       AS pct
FROM (
  SELECT
    SUM(CASE WHEN date_trunc('month', order_date) = date_trunc('month', CURRENT_DATE)
             THEN amount ELSE 0 END) AS curr,
    SUM(CASE WHEN date_trunc('month', order_date) = date_trunc('month', CURRENT_DATE - INTERVAL '1 month')
             THEN amount ELSE 0 END) AS prev
  FROM sales_order WHERE status != 'cancelled'
) t;
```

**Contoh SQL `earnings-breakdown.sql` (PostgreSQL):**

```sql
SELECT category AS label, COALESCE(SUM(amount), 0) AS value
FROM sales_order
WHERE status != 'cancelled'
  AND order_date BETWEEN :from AND :to
GROUP BY category
ORDER BY value DESC
LIMIT 5;
```

**Response:**

```json
"expected_earnings": {
  "value": "69700",
  "trend": { "direction": "up", "pct": "2.2" },
  "items": [
    { "label": "Shoes",  "value": "7660" },
    { "label": "Gaming", "value": "2820" },
    { "label": "Others", "value": "45257" }
  ]
}
```

Frontend menentukan jenis chart (donut, pie, atau bar), warna per kategori, dan urutan label. Persentase per kategori dihitung di frontend dari `items[i].value / sum(items[*].value)`.

---

## Pola 2: Metric + Sparkline (Scalar + Object + Timeseries)

Pola ini menghasilkan headline metric, trend chip, dan timeseries untuk sparkline. Cocok untuk widget seperti "Average Daily Sales" dengan window 7 hari atau 30 hari.

**Payload widget:**

```json
{
  "id": "avg_daily_sales",
  "queries": {
    "value":  "file:query/dashboard-ecommerce/avg-daily-sales-value.sql",
    "trend":  "file:query/dashboard-ecommerce/avg-daily-sales-trend.sql",
    "points": { "file": "file:query/dashboard-ecommerce/avg-daily-sales-points.sql", "shape": "array" }
  }
}
```

**Shape SQL per key:**

| Key | Shape | Hasil collapse |
|-----|-------|----------------|
| `value` | 1 row × 1 col | Scalar primitive |
| `trend` | 1 row × 2 col (`direction`, `pct`) | Object |
| `points` | N rows × 2 col (`period`, `value`) | Array of objects, tetap array untuk 0 atau 1 row karena `shape: "array"` |

**Contoh SQL `avg-daily-sales-points.sql` (PostgreSQL) untuk 7 hari terakhir:**

```sql
SELECT
  to_char(d::date, 'YYYY-MM-DD') AS period,
  COALESCE(SUM(so.total_amount), 0) AS value
FROM generate_series(
  CURRENT_DATE - INTERVAL '6 days',
  CURRENT_DATE,
  INTERVAL '1 day'
) d
LEFT JOIN stock_outbound so
  ON so.outbound_date = d::date AND so.status != 'cancelled'
GROUP BY d
ORDER BY d;
```

> **Perlu diketahui:** `generate_series` menjamin jumlah baris konsisten meskipun ada hari tanpa transaksi. Untuk MySQL gunakan recursive CTE; untuk Oracle gunakan `CONNECT BY LEVEL`.

**Response:**

```json
"avg_daily_sales": {
  "value": "2420",
  "trend": { "direction": "up", "pct": "2.6" },
  "points": [
    { "period": "2026-04-24", "value": "1850" },
    { "period": "2026-04-25", "value": "2200" },
    { "period": "2026-04-26", "value": "1980" },
    { "period": "2026-04-27", "value": "2400" },
    { "period": "2026-04-28", "value": "3050" },
    { "period": "2026-04-29", "value": "3700" },
    { "period": "2026-04-30", "value": "2800" }
  ]
}
```

Frontend memetakan `points.map(p => parseFloat(p.value))` ke series sparkline. Field `period` tetap dibutuhkan untuk tooltip.

---

## Pola 3: Metric + Progress to Goal (Scalar + Object + Scalar Target)

Pola ini menghasilkan headline metric, trend chip, dan dua scalar untuk progress bar terhadap target. Cocok untuk widget "Orders This Month" atau "Monthly Revenue Goal".

**Payload widget:**

```json
{
  "id": "orders_this_month",
  "queries": {
    "value":  "file:query/dashboard-ecommerce/orders-current.sql",
    "trend":  "file:query/dashboard-ecommerce/orders-trend.sql",
    "target": "file:query/dashboard-ecommerce/orders-target.sql"
  }
}
```

**Shape SQL per key:**

| Key | Shape | Hasil collapse |
|-----|-------|----------------|
| `value` | 1 row × 1 col | Scalar primitive |
| `trend` | 1 row × 2 col (`direction`, `pct`) | Object |
| `target` | 1 row × 1 col | Scalar primitive |

**Contoh SQL `orders-target.sql` (PostgreSQL):**

```sql
SELECT target_value AS target
FROM monthly_targets
WHERE period = to_char(CURRENT_DATE, 'YYYY-MM')
  AND metric_name = 'orders';
```

**Response:**

```json
"orders_this_month": {
  "value": "1836",
  "trend": { "direction": "down", "pct": "2.2" },
  "target": "2884"
}
```

Frontend menghitung `pct = Math.round(parseFloat(value) / parseFloat(target) * 100)` untuk display persentase dan width progress bar.

**Variasi: Progress dengan Business Rule**

Bila kalkulasi progress melibatkan aturan bisnis kompleks (misalnya prorata terhadap hari kerja), gabung ke satu SQL multi-column agar `pct` merupakan fakta bisnis yang stabil:

```json
{
  "id": "orders_this_month",
  "queries": {
    "trend": "file:query/dashboard-ecommerce/orders-trend.sql",
    "goal":  "file:query/dashboard-ecommerce/orders-goal.sql"
  }
}
```

SQL `orders-goal.sql` menghasilkan 1 row × 3 col (`current`, `target`, `pct`), yang collapse ke object:

```json
"orders_this_month": {
  "trend": { "direction": "down", "pct": "2.2" },
  "goal":  { "current": "1836", "target": "2884", "pct": "63" }
}
```

---

## Rekomendasi Nama Kolom SQL

Konsistensi nama kolom SQL membuat response lebih predictable bagi konsumen frontend.

| Pola widget | Nama kolom yang direkomendasikan |
|-------------|----------------------------------|
| Bar/pie chart | `category` atau `label`, `value` |
| Timeseries | `period`, `value` |
| Goal progress | `current`, `target` |
| Trend metric | `direction`, `pct` |
| Gauge multi-series | `label`, `value` |
| Notes / teks | `notes` |
