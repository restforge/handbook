# Field States — `fieldStates[]`

> Aturan yang mengunci baris data berdasarkan nilai field status, sehingga aksi `Edit` dan `Delete` tidak dapat dipakai.

## Konsep

Dokumen transaksi biasanya tidak boleh diubah lagi setelah statusnya berpindah, misalnya penerimaan barang yang sudah `posted` atau order yang sudah `paid`. `fieldStates` menyatakan status mana saja yang membuat baris menjadi readonly.

Baris readonly tetap tampil di list. Item `Edit` dan `Delete` di menu Actions tampil mati, disertai ikon informasi yang menjelaskan alasannya. Karena form ubah tidak dapat dibuka, header dan line item sama-sama tidak dapat diubah. Aksi View dan aksi workflow tetap tersedia.

## Sintaks

```json
{
    "workflow": {
        "statusField": "status",
        "transitions": {
            "draft": ["posted", "cancelled"],
            "posted": ["cancelled"],
            "cancelled": []
        }
    },
    "workflowActions": [ /* ... */ ],
    "fieldStates": [
        {
            "when": { "status": ["posted", "cancelled"] },
            "state": "readonly"
        }
    ]
}
```

## Properti `fieldStates[]`

| Properti | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `when` | object | ✓ | Key berupa nama field status sesuai `workflow.statusField`, nilainya array status yang membuat baris readonly |
| `state` | string | ✓ | Isi dengan `"readonly"` |

Nama field status diambil dari `workflow.statusField`. Bila `workflow.statusField` tidak ada, nama field status bernilai `status`.

## Syarat Berlaku

Aturan `fieldStates` hanya berlaku bila semua syarat berikut terpenuhi:

| Syarat | Keterangan |
|--------|-----------|
| Plugin | `vanilla-js-auth` atau `vanilla-js-custom`. Plugin `vanilla-js-basic` mengabaikan `fieldStates` |
| `workflowActions` | Page punya minimal satu item di [`workflowActions`](./workflow-actions.md) |
| `state` | Bernilai `"readonly"` |
| Key `when` | Sama dengan nama field status. Key lain diabaikan |
| Nilai `when` | Array string. Nilai lain diabaikan |

Validator menerima `state: "hidden"`, atribut `fields`, dan `when` berbentuk string, tetapi generator tidak memakainya. Aturan dengan bentuk seperti itu tidak menghasilkan perubahan apa pun pada halaman.

## Perilaku di Menu Actions

| Aspek | Perilaku |
|-------|----------|
| `Edit` dan `Delete` | Tampil mati dengan `disabled` dan `aria-disabled="true"`, tanpa aksi saat diklik |
| Ikon informasi | Tampil di samping item mati pertama. Tooltip-nya menyebut label field status dan nilai baris, misalnya `Editing and deletion are unavailable while Status is Posted.` |
| `Deactivate` dan `Activate` | Ikut tampil mati bersama `Edit`, bila halaman punya item tersebut (lihat [`features.md`](./features.md#nonaktifkan-dan-aktifkan-kembali)) |
| Izin pengguna | Pengguna tanpa izin ubah atau hapus tidak melihat item tersebut sama sekali. Izin menang atas readonly |

Status readonly dibaca dari nilai baris saat list digambar. Form ubah yang sudah terbuka sebelum status berpindah tidak ikut terkunci.

## Beberapa Aturan

Beberapa aturan readonly boleh ditulis terpisah. Status dari semua aturan digabung menjadi satu daftar.

```json
{
    "fieldStates": [
        { "when": { "status": ["paid", "shipped"] }, "state": "readonly" },
        { "when": { "status": ["completed", "cancelled"] }, "state": "readonly" }
    ]
}
```

Hasilnya sama dengan satu aturan berisi `["paid", "shipped", "completed", "cancelled"]`.

## Aturan Validasi

| Aturan | Error |
|--------|-------|
| `when` wajib diisi | `"[<pageId>].fieldStates[<i>] when must be provided"` |
| `state` valid (`readonly` atau `hidden`) | `"[<pageId>].fieldStates[<i>] state is invalid: '<value>'. Use 'readonly' or 'hidden'"` |

Validator tidak memeriksa bentuk `when`.

## Penegakan di Backend

`fieldStates` hanya mengatur tampilan frontend. Runtime backend tidak membacanya, sehingga request API langsung dan form ubah yang sudah terbuka tetap dapat menyimpan perubahan. Aturan yang sama ditegakkan di backend lewat handler [component engine](../../features/component-engine/README.md), misalnya `onBeforeCompositeUpdate` dan `onBeforeDelete` yang menolak dokumen di luar status awal.

```js
async function goodReceiptBeforeCompositeUpdate(tableName, requestData, oldData) {
  if (oldData.status !== 'draft') {
    return { success: false, message: `Penerimaan ${oldData.receipt_number} sudah berstatus '${oldData.status}' sehingga tidak dapat diubah` };
  }
  return { success: true };
}
```

## Contoh Penggunaan Praktis

### Penerimaan Barang

```json
{
    "pageId": "good-receipt",
    "workflow": {
        "statusField": "status",
        "transitions": {
            "draft": ["posted", "cancelled"],
            "posted": ["cancelled"],
            "cancelled": []
        }
    },
    "workflowActions": [
        {
            "actionId": "posted",
            "label": "Post",
            "api": {
                "endpoint": "good-receipt/change-status",
                "payload": { "good_receipt_id": "$primaryKey", "status": "posted" }
            }
        },
        {
            "actionId": "cancelled",
            "label": "Cancel",
            "api": {
                "endpoint": "good-receipt/change-status",
                "payload": { "good_receipt_id": "$primaryKey", "status": "cancelled" }
            }
        }
    ],
    "fieldStates": [
        {
            "when": { "status": ["posted", "cancelled"] },
            "state": "readonly"
        }
    ]
}
```

| `status` | `Edit` dan `Delete` | Aksi workflow |
|----------|---------------------|---------------|
| `draft` | Aktif | Post, Cancel |
| `posted` | Mati, dengan ikon informasi | Cancel |
| `cancelled` | Mati, dengan ikon informasi | Tidak ada, dialog menampilkan `No transitions available for this status.` |

---

← [`workflow-actions.md`](./workflow-actions.md) | [Selanjutnya: `navigation.md`](./navigation.md) →
