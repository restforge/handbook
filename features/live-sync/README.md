# Live Sync

> Notifikasi real-time ke semua browser yang membuka resource yang sama, tanpa polling.

Live Sync berfungsi untuk menyinkronkan tampilan data antar client secara otomatis melalui WebSocket pub/sub. Saat satu client melakukan operasi CRUD, semua client lain yang sedang membuka resource yang sama menerima notifikasi `data_change` dan dapat memperbarui tampilan tanpa refresh manual.

---

## Operasi yang Didukung

| Operasi | Broadcast | Keterangan |
|---------|:---------:|------------|
| CREATE | Ya | Setelah INSERT berhasil dan di-commit |
| UPDATE | Ya | Setelah UPDATE berhasil dan di-commit |
| DELETE | Ya | Setelah DELETE berhasil dan di-commit |
| ADJUST | Ya | Broadcast sebagai action `update` |
| Composite INSERT/UPDATE | Ya | Melalui `executeTransactionWithEvents` |
| READ, Datatables, Lookup | Tidak | Operasi baca tidak mengubah data |

Broadcast terjadi di level base model sehingga berlaku secara otomatis untuk semua endpoint tanpa modifikasi kode.

---

## Prasyarat

| Komponen | Keterangan |
|----------|------------|
| RESTForge | Versi 2.2.0 ke atas |
| API Key | Wajib — Live Sync menolak start tanpa API Key |
| Redis | Wajib — untuk komunikasi antara REST API process dan Live Sync process |

API Key bersifat wajib karena koneksi WebSocket persisten dan data broadcast dapat mengandung field sensitif. Live Sync tidak dapat diaktifkan dalam mode tanpa autentikasi.

---

## Quick Setup

### 1. Pastikan Redis aktif

Live Sync memerlukan Redis untuk relay broadcast antara REST API process dan WebSocket process.

### 2. Konfigurasi `.env`

```env
KEY=rk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

LIVE_SYNC_ENABLED=true
LIVE_SYNC_PORT=3033

REDIS_HOST=localhost
REDIS_PORT=6380
```

### 3. Jalankan server

```bash
npx restforge serve --project=mini-inventory --config=db-connection.env
```

RESTForge secara otomatis menjalankan dua process dari satu perintah: REST API di `SERVER_PORT` dan Live Sync WebSocket di `LIVE_SYNC_PORT`.

### 4. Hubungkan client

```javascript
const ws = new WebSocket('ws://localhost:3033/ws?x-api-key=rk_xxx');

ws.onopen = function() {
  ws.send(JSON.stringify({
    type: 'subscribe',
    channels: ['mini-inventory:category']
  }));
};

ws.onmessage = function(event) {
  const msg = JSON.parse(event.data);
  if (msg.type === 'data_change') {
    dataTable.ajax.reload(null, false);
  }
  if (msg.type === 'ping') {
    ws.send(JSON.stringify({ type: 'pong' }));
  }
};
```

> **Perlu diketahui:** Jika `LIVE_SYNC_ENABLED=true` dikonfigurasi tanpa API Key, server menolak start dengan pesan error yang menyertakan cara generate API Key.

---

## Dokumentasi Lengkap

| Dokumen | Isi |
|---------|-----|
| [Konfigurasi](configuration.md) | Semua variable `.env`, arsitektur dedicated process, cluster mode, health check |
| [Protokol WebSocket](protocol.md) | Format URL koneksi, pesan client ke server, pesan server ke client, broadcast message spec |
| [Integrasi Client-Side](client-integration.md) | Pola subscription, contoh form dan dashboard, self-notification filtering |
| [Menangani Stale Data](stale-data.md) | Penanganan form yang terbuka saat data berubah oleh user lain |
| [Batasan dan Pemilihan Mekanisme](boundaries.md) | Perbedaan Live Sync vs Kafka vs Webhook, kapan memakai masing-masing |
