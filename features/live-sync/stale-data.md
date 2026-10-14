# Menangani Stale Data

> Cara mendeteksi dan merespons perubahan pada record yang sedang terbuka di form.

Kembali ke [ikhtisar Live Sync](README.md).

---

## Masalah

Saat user membuka record dalam mode View atau Edit, data ditampilkan berdasarkan snapshot saat record di-klik. Jika user lain melakukan update atau delete terhadap record yang sama, terjadi kondisi stale data:

```
t=0s  : User A klik record CAT001 → form tampil "Elektronik"
t=5s  : User B update CAT001 → nama menjadi "Electronic Devices"
t=5s  : User A terima data_change → DataTable reload di background
t=10s : User A masih melihat "Elektronik" di form ← data sudah tidak akurat
```

DataTable di background sudah ter-refresh, namun form yang sedang terbuka tidak ikut ter-update karena datanya sudah berada di DOM.

Penanganan kondisi ini sepenuhnya menjadi tanggung jawab frontend. Backend sudah menyediakan informasi yang cukup di setiap broadcast message:

| Informasi | Sumber di broadcast | Kegunaan |
|-----------|---------------------|----------|
| Record mana yang berubah | `msg.data.{primary_key}` | Dibandingkan dengan record yang sedang terbuka |
| Aksi apa yang terjadi | `msg.action` | Menentukan respons: refresh atau tutup form |
| Data terbaru | `msg.data` | Langsung mengisi form tanpa REST API call tambahan |

---

## Implementasi

### Langkah 1 — Track Record yang Sedang Terbuka

Simpan primary key dan mode form saat user membuka sebuah record.

```javascript
let openRecordId   = null;
let openRecordMode = null;  // 'view' atau 'edit'

function openRecord(recordId, mode) {
  openRecordId   = recordId;
  openRecordMode = mode;
  // load data dan tampilkan form...
}

function closePanel() {
  openRecordId   = null;
  openRecordMode = null;
  // tutup form...
}
```

### Langkah 2 — Deteksi Perubahan pada Record yang Sedang Terbuka

Di handler `data_change`, bandingkan record yang berubah dengan record yang sedang terbuka.

```javascript
ws.onmessage = function(event) {
  const msg = JSON.parse(event.data);

  if (msg.type === 'ping') {
    ws.send(JSON.stringify({ type: 'pong' }));
    return;
  }

  if (msg.type !== 'data_change') return;
  if (msg.requestId && msg.requestId === currentRequestId) return;

  // Refresh DataTable (selalu dilakukan)
  dataTable.ajax.reload(null, false);

  // Cek apakah form yang sedang terbuka terkena dampak
  if (!openRecordId) return;

  const changedId = extractPrimaryKey(msg);
  if (changedId !== openRecordId) return;

  if (msg.action === 'delete') {
    handleOpenRecordDeleted();
  } else if (msg.action === 'update') {
    handleOpenRecordUpdated(msg.data);
  }
};

function extractPrimaryKey(msg) {
  // action delete: data = array
  if (msg.action === 'delete') {
    return msg.data[0] ? msg.data[0].category_id : null;
  }
  // action create/update: data = object
  return msg.data ? msg.data.category_id : null;
}
```

> **Perlu diketahui:** Nama primary key berbeda per endpoint. Contoh di atas menggunakan `category_id`. Gunakan primary key yang sesuai untuk endpoint masing-masing, mis. `warehouse_id`, `supplier_id`.

### Langkah 3 — Tangani Record yang Dihapus

Tutup form dan tampilkan notifikasi saat record yang sedang terbuka dihapus oleh user lain.

```javascript
function handleOpenRecordDeleted() {
  closePanel();
  showNotification('warning',
    'Record yang sedang dilihat telah dihapus oleh pengguna lain.');
}
```

### Langkah 4 — Tangani Record yang Di-update

Respons berbeda berdasarkan mode form saat update terjadi.

**Mode View (read-only):** perbarui form langsung tanpa konfirmasi.

```javascript
function handleOpenRecordUpdated(newData) {
  if (openRecordMode === 'view') {
    populateForm(newData);
    showNotification('info',
      'Data yang sedang ditampilkan telah diperbarui oleh pengguna lain.');
    return;
  }

  // Mode edit: tampilkan konfirmasi
  showStaleDataConfirmation(newData);
}
```

**Mode Edit:** tampilkan dialog konfirmasi karena user mungkin sudah mengubah field yang belum tersimpan.

```javascript
function showStaleDataConfirmation(newData) {
  Swal.fire({
    title: 'Data Telah Berubah',
    text: 'Record ini telah diperbarui oleh pengguna lain. ' +
          'Muat ulang data terbaru? (Perubahan yang belum disimpan akan hilang)',
    icon: 'warning',
    showCancelButton: true,
    confirmButtonText: 'Ya, muat ulang',
    cancelButtonText: 'Tidak, lanjutkan edit'
  }).then(function(result) {
    if (result.isConfirmed) {
      populateForm(newData);
    } else {
      showStaleIndicator();
    }
  });
}

function showStaleIndicator() {
  $('#stale-warning').show();
  // Tampilkan banner: "Data ini telah diperbarui oleh pengguna lain."
}
```

> **Perlu diketahui:** Dialog konfirmasi bersifat opsional. Untuk aplikasi sederhana, cukup reload form tanpa konfirmasi. Contoh di atas menggunakan SweetAlert2 — bisa diganti dengan library lain seperti Bootstrap Modal atau dialog bawaan framework UI.

### Langkah 5 — Isi Form dari Data Broadcast

Data dari broadcast message dapat langsung digunakan untuk mengisi form tanpa REST API call tambahan.

```javascript
function populateForm(data) {
  $('#category-code').val(data.category_code);
  $('#category-name').val(data.category_name);
  $('#description').val(data.description);
  $('#is-active').prop('checked', data.is_active);
}
```

Field `data` pada action `update` berisi seluruh record dari hasil `RETURNING *`, sehingga tidak perlu memanggil endpoint `/first` ke server.

---

## Alur Lengkap

**Skenario 1 — Record diperbarui saat form terbuka:**

```
User A buka View CAT001          User B
  │                                │
  ├ openRecordId = 'CAT001'        │
  ├ form tampil "Elektronik"       │
  │                                │
  │                          User B update CAT001
  │                          nama → "Electronic Devices"
  │                                │
  ◄─── data_change ───────────────┘
  │  action: "update"
  │  data.category_name: "Electronic Devices"
  │
  ├ dataTable.ajax.reload()       ← tabel di-refresh
  ├ changedId === openRecordId    ← Ya
  │
  ├ openRecordMode === 'view'?
  │   Ya  → populateForm(msg.data) → form langsung ter-update
  │   Tidak (edit) → showStaleDataConfirmation()
  │                   ├ "Ya, muat ulang" → populateForm(msg.data)
  │                   └ "Tidak" → showStaleIndicator()
```

**Skenario 2 — Record dihapus saat form terbuka:**

```
User A buka Edit CAT003          User B
  │                                │
  ├ openRecordId = 'CAT003'        │
  ├ user mulai edit...             │
  │                                │
  │                          User B delete CAT003
  │                                │
  ◄─── data_change ───────────────┘
  │  action: "delete"
  │  data: [{ category_id: "CAT003", ... }]
  │
  ├ dataTable.ajax.reload()       ← CAT003 hilang dari tabel
  ├ changedId === openRecordId    ← Ya
  ├ action === 'delete'
  └ closePanel() + showNotification("Record telah dihapus")
```

---

Lihat juga: [Integrasi Client-Side](client-integration.md) · [Protokol WebSocket](protocol.md)
