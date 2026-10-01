# Konvensi Penamaan

> Aturan penamaan `--project`, `--name`, `processor[].name`, dan kapan memakai suffix `-process`.

Kembali ke [ikhtisar Processor](README.md).

---

## Anatomi URL yang Dihasilkan

Generator menghasilkan URL dengan pola tetap:

```
/api / {--project} / {--name} / {processor[].name}
  │         │            │              │
  │         │            │              └── Aksi spesifik
  │         │            └── Resource group
  │         └── Domain bisnis
  └── Prefix API global
```

Contoh konkret:

```
/api/sales/order-process/submit-order
      │         │               └── processor[].name  — aksi spesifik
      │         └── --name                            — resource group
      └── --project                                   — domain bisnis
```

---

## Aturan per Parameter

### `--project` — Domain Bisnis

Merepresentasikan domain atau bounded context dari endpoint yang dibuat.

| Kriteria | Keterangan |
|----------|------------|
| Format | `kebab-case`, huruf kecil |
| Jenis kata | Kata benda, nama domain |
| Contoh benar | `sales`, `hr`, `inventory`, `procurement` |
| Contoh salah | `salesService`, `do-sales`, `handleHr` |

---

### `--name` — Resource Group

Menjadi nama router file dan nama folder processor. Harus merepresentasikan entitas atau kelompok logis dari endpoint-endpoint di dalamnya, bukan aksi.

| Kriteria | Keterangan |
|----------|------------|
| Format | `kebab-case`, huruf kecil |
| Jenis kata | Kata benda (noun) — bukan kata kerja |
| Pertanyaan panduan | "Entitas atau kelompok apa yang dioperasikan?" |

```
✅ BENAR
  order-process          → kelompok operasi pemrosesan order
  employee               → kelompok operasi karyawan
  leave-request-process  → kelompok operasi pengajuan cuti

❌ SALAH
  handler-process        → terlalu generik
  sales-submit           → ini aksi, bukan resource
  do-process             → mengandung kata kerja
  orderHandler           → camelCase
```

---

### `processor[].name` — Aksi Spesifik

Menjadi nama file processor dan segmen terakhir URL. Harus merepresentasikan aksi spesifik tanpa mengulang nama project atau endpoint.

| Kriteria | Keterangan |
|----------|------------|
| Format | `kebab-case`, huruf kecil |
| Jenis kata | Kata kerja atau frasa aksi (verb-phrase) |
| Pertanyaan panduan | "Aksi apa yang dilakukan terhadap resource?" |

```
✅ BENAR
  submit-order       → /api/sales/order-process/submit-order
  approve-order      → /api/sales/order-process/approve-order
  deactivate         → /api/hr/employee/deactivate
  calculate-payroll  → /api/hr/payroll-process/calculate-payroll

❌ SALAH
  sales-submit       → prefix "sales-" redundan (sudah ada di --project)
  order-submit       → prefix "order-" redundan (sudah ada di --name)
  doSubmit           → camelCase
```

---

## Suffix `-process` untuk Menghindari Konflik

### Masalah

CRUD module dan processor sama-sama menghasilkan router file dengan nama `{name}.js`. Jika keduanya menggunakan nama yang identik, router processor akan menimpa router CRUD setiap kali generator dijalankan.

```
CRUD Module → src/modules/sales/sales-order.js   ← dibuat CRUD generator
Processor   → src/modules/sales/sales-order.js   ← DITIMPA processor generator ❌
```

### Solusi

Tambahkan suffix `-process` pada `--name` processor untuk membedakannya dari CRUD module. Suffix ini dibaca sebagai "proses bisnis [entitas]", bukan sebagai kata kerja.

```
sales-order           → digunakan CRUD Module
sales-order-process   → digunakan Processor
```

Kedua router berdampingan tanpa konflik:

```
src/modules/sales/
├── sales-order.js              ← CRUD router
├── sales-order-process.js      ← Processor router
└── processor/
    └── sales-order-process/
        ├── submit-order.js
        ├── approve-order.js
        └── cancel-order.js
```

### Kapan Suffix Diperlukan

Suffix `-process` hanya digunakan jika ketiga syarat berikut terpenuhi:

| Syarat | Keterangan |
|--------|------------|
| Entitas sudah memiliki CRUD Module | Jika belum ada CRUD, gunakan nama entitas langsung |
| Nama entitas ada di depan | Pola wajib: `{entitas}-process`, bukan hanya `-process` |
| Operasi yang dibuat adalah bisnis custom | Bukan pengganti CRUD — submit, approve, reject, dsb. |

```
✅ BENAR
  sales-order-process    → entitas "sales-order" sudah punya CRUD
  leave-request-process  → entitas "leave-request" sudah punya CRUD

⚠️  TIDAK PERLU SUFFIX
  employee               → tidak ada CRUD "employee", pakai nama langsung

❌ SALAH
  order-process          → entitas tidak jelas, "order" atau "sales-order"?
  process-order          → urutan terbalik, entitas harus di depan
  sales-process          → terlalu lebar, tidak spesifik ke entitas mana
```

---

## Pengelompokan Multi-Processor

Kelompokkan processor berdasarkan entitas yang dioperasikan, bukan kesamaan teknis implementasinya.

```
✅ BENAR — dikelompokkan berdasarkan entitas
  --name=order-process   → submit-order, approve-order, reject-order, cancel-order
  --name=employee        → register, update-profile, deactivate, reactivate

❌ SALAH — dikelompokkan berdasarkan teknis
  --name=handler-process → sales-submit, sales-approve, hr-register, ...
```

### Kapan Membuat `--name` Baru vs Menambah ke yang Ada

| Kondisi | Keputusan |
|---------|-----------|
| Aksi beroperasi pada entitas yang sama | Tambahkan ke `--name` yang ada |
| Aksi beroperasi pada entitas berbeda | Buat `--name` baru |
| Jumlah processor dalam satu `--name` > 6 | Pertimbangkan untuk dipecah |
| Entitas sudah punya CRUD Module | Gunakan suffix `-process` |

---

## Tabel Rujukan Cepat

| Parameter | Jenis Kata | Format | Contoh Benar | Contoh Salah |
|-----------|------------|--------|--------------|--------------|
| `--project` | Kata benda, domain | `kebab-case` | `sales`, `hr` | `salesService`, `doSales` |
| `--name` | Kata benda, resource | `kebab-case` | `order-process`, `employee` | `handler-process`, `sales-submit` |
| `--name` (ada konflik CRUD) | Kata benda + suffix | `{entitas}-process` | `sales-order-process` | `process-order`, `sales-process` |
| `processor[].name` | Kata kerja, aksi | `kebab-case` | `submit-order`, `deactivate` | `sales-submit`, `doSubmit` |

---

## Contoh Lengkap

### Modul `sales`

```bash
npx restforge processor create --project=sales --name=order-process     --payload=order_process.json
npx restforge processor create --project=sales --name=invoice-process   --payload=invoice_process.json
```

Struktur hasil:

```
src/modules/sales/
├── order-process.js
├── invoice-process.js
└── processor/
    ├── order-process/
    │   ├── submit-order.js
    │   ├── approve-order.js
    │   ├── reject-order.js
    │   └── cancel-order.js
    └── invoice-process/
        ├── generate.js
        ├── send.js
        └── mark-paid.js
```

### Modul `hr`

```bash
npx restforge processor create --project=hr --name=employee              --payload=hr_employee.json
npx restforge processor create --project=hr --name=leave-request-process --payload=hr_leave_request.json
```

Struktur hasil:

```
src/modules/hr/
├── employee.js
├── leave-request-process.js
└── processor/
    ├── employee/
    │   ├── register.js
    │   ├── update-profile.js
    │   └── deactivate.js
    └── leave-request-process/
        ├── submit.js
        ├── approve.js
        └── reject.js
```

---

Lihat juga: [Payload Processor](payload.md) · [Menulis Processor](handler.md) · [Keamanan](security.md)
