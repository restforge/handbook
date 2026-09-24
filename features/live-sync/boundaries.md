# Batasan dan Pemilihan Mekanisme

> Perbedaan Live Sync, Kafka, dan Webhook — kapan memakai masing-masing.

Kembali ke [ikhtisar Live Sync](README.md).

---

## Live Sync Bukan untuk Komunikasi Antar Microservice

Live Sync dirancang khusus untuk sinkronisasi tampilan antar browser client yang membuka resource yang sama. Fitur ini tidak dirancang untuk komunikasi server-to-server.

Contoh arsitektur dengan tiga microservice:

```
MS1: Sales Order   (port 3001)  ← RESTForge runtime
MS2: Email Service (port 3002)  ← RESTForge runtime
MS3: Accounting    (port 3003)  ← RESTForge runtime
```

Saat user melakukan SAVE sales order di MS1, ada dua kebutuhan yang berbeda:

| Kebutuhan | Mekanisme yang tepat |
|-----------|---------------------|
| Browser lain yang membuka form Sales Order perlu refresh | **Live Sync** |
| MS2 perlu kirim email konfirmasi | **Kafka** atau **Webhook** |
| MS3 perlu buat jurnal akuntansi | **Kafka** atau **Webhook** |

### Mengapa Live Sync Tidak Cocok untuk Antar Service

Tiga alasan teknis mengapa Live Sync tidak cocok untuk komunikasi server-to-server:

1. **Setiap RESTForge runtime adalah server independen.** MS1 di port 3001 tidak memiliki koneksi WebSocket ke MS2 di port 3002. Live Sync hanya mengelola koneksi dari browser ke server tempat mereka terhubung.

2. **WebSocket tidak menyediakan delivery guarantee.** Komunikasi antar service membutuhkan mekanisme yang reliable, idempotent, dan dapat retry jika gagal. WebSocket tidak memiliki jaminan ini.

3. **WebSocket menciptakan coupling yang ketat.** Jika MS2 harus subscribe ke channel MS1 via WebSocket, maka MS2 bergantung pada ketersediaan MS1. Setiap restart MS1 menyebabkan koneksi putus dan MS2 kehilangan event.

---

## Mekanisme yang Tepat per Kebutuhan

### Kafka — Event-Driven, Disarankan untuk Antar Service

Kafka menyediakan message persistence dan delivery guarantee untuk komunikasi asinkron antar service.

```env
# Konfigurasi di MS1 (.env)
KAFKA_ENABLED=true
KAFKA_BROKERS=localhost:9092
KAFKA_TOPIC_PATTERN={module}.{endpoint}.events
```

```
MS1 save sales order
  → Kafka publish ke topic "sales-order.sales-order.events"
  → MS2 consume event → kirim email
  → MS3 consume event → buat jurnal
```

Keunggulan: message persistence, delivery guarantee, decoupling penuh antar service, consumer bisa replay event lama.

### Webhook — HTTP Callback, Tanpa Broker

Webhook memanggil endpoint service lain secara langsung via HTTP setelah operasi selesai.

```
MS1 save sales order
  → POST http://localhost:3002/api/notification/email/create
  → POST http://localhost:3003/api/accounting/journal/create
```

Keunggulan: tidak memerlukan dependency tambahan seperti Kafka broker, konfigurasi sederhana.

---

## Ringkasan Pemilihan Mekanisme

| Kebutuhan | Gunakan |
|-----------|---------|
| Browser A tahu bahwa Browser B baru saja edit data yang sama | **Live Sync** |
| Dashboard menampilkan data real-time dari semua endpoint | **Live Sync** (wildcard) |
| Service A memberitahu Service B bahwa ada event baru | **Kafka** |
| Menjalankan proses background setelah data berubah | **Kafka** consumer |
| Service A memanggil endpoint Service B setelah operasi selesai | **Webhook** |

---

## Skenario: Ketiga Mekanisme Berjalan Bersamaan

Dalam deployment production, ketiga mekanisme melayani kebutuhan yang berbeda secara bersamaan.

```
User klik SAVE di form Sales Order (browser)
    │
    ▼
MS1 — Sales Order (port 3001)
    │
    ├─ 1. addData() → INSERT → COMMIT
    │
    ├─ 2. Kafka publishInsert()                    [server → server]
    │      → topic: "sales-order.sales-order.events"
    │      → MS2 consume → kirim email konfirmasi
    │      → MS3 consume → buat jurnal akuntansi
    │
    ├─ 3. Live Sync broadcast()                    [server → browser]
    │      → channel: "sales-order:sales-order"
    │      → Browser lain yang membuka form → reload DataTable
    │
    └─ 4. return 201 → browser yang melakukan SAVE
```

Setiap mekanisme bertanggung jawab atas satu lapisan komunikasi yang berbeda dan tidak bisa digantikan satu sama lain.

---

Lihat juga: [Konfigurasi](configuration.md) · [ikhtisar Live Sync](README.md)
