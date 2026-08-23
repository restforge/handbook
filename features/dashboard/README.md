# Dashboard

Dashboard berfungsi untuk membuat endpoint multi-widget aggregator dari payload JSON. SQL setiap widget di-embed di handler generated pada generation time, menghilangkan disk I/O per request. Semua widget dieksekusi secara paralel via `Promise.allSettled`.

> Endpoint runtime: `POST /api/{project}/{name}/dashboard` — nilai `{name}` wajib mengandung prefix `dash-`, misalnya `dash-sales`.

---

## Referensi Cepat

| Properti | Nilai |
|----------|-------|
| Subcommand CLI | `npx restforge dashboard create` |
| Endpoint runtime | `POST /api/{project}/{name}/dashboard` |
| Database | PostgreSQL, MySQL, Oracle (SQLite tidak didukung) |
| Disk I/O per request | Tidak ada; SQL di-embed di module |
| Eksekusi widget | Paralel via `Promise.allSettled` |
| Error per widget | Inline `{ "error": "..." }` per widget; top-level `success: true` tetap |
| Scalar collapse | Otomatis berdasarkan shape SQL result |
| Args CLI wajib | `--project`, `--name`, `--payload` |
| Args CLI opsional | `--database` (default `postgres`), `--force` (default `false`) |
| Konvensi response | Semua nilai numeric diubah ke string, semua key diubah ke lowercase |

---

## Quick Start

### 1. Buat payload

Buat file `payload/dashboard-sales.json`:

```json
{
  "params": {
    "from": { "type": "date", "required": true },
    "to":   { "type": "date", "required": true }
  },
  "widgets": [
    {
      "id": "sales_by_city",
      "query": "file:query/dashboard-sales/sales-by-city.sql"
    },
    {
      "id": "revenue_summary",
      "queries": {
        "value":  "file:query/dashboard-sales/revenue-value.sql",
        "points": "file:query/dashboard-sales/revenue-points.sql"
      }
    }
  ]
}
```

### 2. Siapkan SQL widget

`payload/query/dashboard-sales/sales-by-city.sql`:

```sql
SELECT c.city AS category, COALESCE(SUM(so.total_amount), 0) AS value
FROM stock_outbound so
JOIN customer c ON c.customer_id = so.customer_id
WHERE so.status != 'cancelled'
  AND so.outbound_date BETWEEN :from AND :to
GROUP BY c.city
ORDER BY value DESC
LIMIT 12;
```

`payload/query/dashboard-sales/revenue-value.sql`:

```sql
SELECT COALESCE(SUM(total_amount), 0) AS value
FROM stock_outbound
WHERE status != 'cancelled'
  AND outbound_date BETWEEN :from AND :to;
```

`payload/query/dashboard-sales/revenue-points.sql`:

```sql
SELECT
  to_char(outbound_date, 'YYYY-MM-DD') AS period,
  COALESCE(SUM(total_amount), 0)       AS value
FROM stock_outbound
WHERE status != 'cancelled'
  AND outbound_date BETWEEN :from AND :to
GROUP BY outbound_date
ORDER BY outbound_date;
```

### 3. Generate endpoint

```bash
npx restforge dashboard create \
  --project=mini-inventory \
  --name=dash-sales \
  --payload=dashboard-sales
```

Output:

```
Dashboard module created: src/modules/mini-inventory/dash-sales.js
URL: POST /api/mini-inventory/dash-sales/dashboard
```

### 4. Panggil endpoint

```bash
curl -X POST http://127.0.0.1:3032/api/mini-inventory/dash-sales/dashboard \
  -H "Content-Type: application/json" \
  -d '{ "params": { "from": "2026-04-01", "to": "2026-04-30" } }'
```

Response:

```json
{
  "success": true,
  "data": {
    "sales_by_city": {
      "items": [
        { "category": "Jakarta",  "value": "288000000" },
        { "category": "Surabaya", "value": "229600000" }
      ]
    },
    "revenue_summary": {
      "value": "821400000",
      "points": [
        { "period": "2026-04-01", "value": "27600000" },
        { "period": "2026-04-02", "value": "31200000" }
      ]
    }
  }
}
```

---

## Dokumen Lanjutan

| Dokumen | Isi |
|---------|-----|
| [Payload](payload.md) | Struktur payload, params contract, cache, validasi |
| [Pola Widget](widget-patterns.md) | Scalar collapse, tiga pola widget umum |
| [Pemetaan Frontend](frontend-mapping.md) | Statistics card, mixed widget, multi-series chart, pemisahan concern |
| [Generate & Best Practices](generate.md) | CLI args, setup, output, multi-dialect, best practices, troubleshooting |
