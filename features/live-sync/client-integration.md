# Integrasi Client-Side

> Pola subscription, contoh koneksi form dan dashboard, serta self-notification filtering.

Kembali ke [ikhtisar Live Sync](README.md).

---

## Pola Subscription

### Channel Spesifik

Digunakan pada form atau halaman yang hanya menampilkan data dari satu endpoint.

```javascript
ws.send(JSON.stringify({
  type: 'subscribe',
  channels: ['mini-inventory:category']
}));
```

### Multi-Channel

Digunakan pada halaman yang menampilkan data dari beberapa endpoint terkait sekaligus.

```javascript
ws.send(JSON.stringify({
  type: 'subscribe',
  channels: [
    'mini-inventory:stock-inbound',
    'mini-inventory:stock-outbound'
  ]
}));
```

### Wildcard untuk Dashboard

Digunakan pada dashboard atau halaman ringkasan yang bergantung pada banyak resource dalam satu project.

```javascript
ws.send(JSON.stringify({
  type: 'subscribe',
  channels: ['mini-inventory:*']
}));
```

Dengan wildcard, broadcast ke channel apa pun dalam project akan diterima:

| Channel yang di-broadcast | Diterima? |
|--------------------------|:---------:|
| `mini-inventory:category` | Ya |
| `mini-inventory:stock-inbound` | Ya |
| `other-project:users` | Tidak — project berbeda |

| Kebutuhan | Pattern |
|-----------|---------|
| Form satu resource | `{project}:{endpoint}` |
| Dashboard seluruh project | `{project}:*` |
| Beberapa halaman terkait | Array beberapa channel |

---

## Contoh: Form Category

Contoh ini mencakup koneksi WebSocket, subscribe ke channel, penanganan pesan, auto-reconnect, dan tracking `requestId` untuk self-notification filtering.

```javascript
let ws = null;
let currentRequestId = null;

// Dipanggil saat halaman dibuka
$(document).ready(function() {
  initDataTable();
  connectLiveSync('mini-inventory:category');
});

function connectLiveSync(channel) {
  const apiKey = getApiKey();
  const protocol = location.protocol === 'https:' ? 'wss:' : 'ws:';
  const wsUrl = `${protocol}//${location.host}/ws?x-api-key=${apiKey}`;

  ws = new WebSocket(wsUrl);

  ws.onopen = function() {
    ws.send(JSON.stringify({
      type: 'subscribe',
      channels: [channel]
    }));
  };

  ws.onmessage = function(event) {
    const msg = JSON.parse(event.data);

    if (msg.type === 'data_change') {
      // Skip notifikasi dari request sendiri
      if (msg.requestId && msg.requestId === currentRequestId) return;

      dataTable.ajax.reload(null, false);
      showNotification('info', 'Data telah diperbarui oleh pengguna lain.');
    }

    if (msg.type === 'ping') {
      ws.send(JSON.stringify({ type: 'pong' }));
    }
  };

  ws.onclose = function() {
    setTimeout(() => connectLiveSync(channel), 3000);
  };
}

// Saat SAVE, kirim X-Request-ID untuk filter self-notification
function saveRecord() {
  currentRequestId = generateUUID();

  $.ajax({
    url: apiUrl,
    type: 'POST',
    contentType: 'application/json',
    headers: { 'X-Request-ID': currentRequestId },
    data: JSON.stringify(payload),
    success: function(response) {
      dataTable.ajax.reload(null, false);
      showNotification('success', 'Data berhasil disimpan.');
    }
  });
}
```

---

## Contoh: Dashboard dengan Wildcard

Dashboard yang me-refresh widget berbeda berdasarkan resource yang berubah.

```javascript
connectLiveSync('mini-inventory:*');

function handleDataChange(message) {
  if (message.requestId === currentRequestId) return;

  const resource = message.channel.split(':')[1];

  switch (resource) {
    case 'stock-inbound':
    case 'stock-outbound':
      refreshStockValueWidget();
      refreshRecentTransactionsWidget();
      break;
    case 'category':
    case 'item-product':
      refreshTopCategoryWidget();
      break;
  }
}
```

---

## Self-Notification Filtering

Saat client melakukan SAVE, client tersebut juga menerima notifikasi `data_change` dari server karena ikut subscribe ke channel yang sama. Tanpa filter, client akan reload data dua kali: sekali dari AJAX success callback dan sekali dari notifikasi WebSocket.

**Mekanisme filter menggunakan `requestId`:**

1. Client men-generate UUID dan mengirimnya sebagai header `X-Request-ID` pada REST API call.
2. Server meneruskan `requestId` ini ke dalam broadcast message.
3. Client membandingkan `requestId` di notifikasi dengan ID yang baru dikirim.
4. Jika cocok — notifikasi dari aksi sendiri, dilewati.
5. Jika tidak cocok — notifikasi dari client lain, proses reload.

```javascript
// Saat kirim request:
currentRequestId = generateUUID();
headers['X-Request-ID'] = currentRequestId;

// Saat terima notifikasi:
if (msg.requestId && msg.requestId === currentRequestId) return; // skip
```

> **Perlu diketahui:** Module harus di-regenerate agar `requestId` diteruskan dari endpoint ke base model. Module yang di-generate sebelum fitur ini tetap berfungsi, namun `requestId` akan bernilai `null` sehingga semua notifikasi diproses tanpa filter.

---

Lihat juga: [Protokol WebSocket](protocol.md) · [Menangani Stale Data](stale-data.md)
