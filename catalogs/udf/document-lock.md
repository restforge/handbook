# Document Lock — `documentLock` + `documentLockTargets[]`

> Blok page yang mematikan aksi pada dokumen terkunci dan membuat form perujuk mengirim versi dokumen yang dirujuk.

## Konsep

Backend menolak ubah, hapus, dan batal atas dokumen yang final atau sudah dipakai dokumen lain (lihat [`../rdf/document-lock.md`](../rdf/document-lock.md)). Dua blok di page UDF membuat frontend ikut mengikuti aturan itu, sehingga pengguna melihat aksi yang mati beserta alasannya, bukan menerima error setelah menekan tombol.

| Blok | Dipakai oleh | Fungsi |
|------|--------------|--------|
| `documentLock` | Page dokumen yang dikunci, misalnya purchase order | Mematikan `Edit`, `Delete`, dan aksi workflow tertentu pada baris yang `is_lock`-nya `true` |
| `documentLockTargets[]` | Page dokumen perujuk, misalnya penerimaan barang | Mencatat `document_version` dari dokumen yang dipilih, lalu mengirimnya lewat `options.documentVersions` saat simpan |

Kedua blok ditulis `payload migrate` dari RDF yang punya [`documentLock` dan `documentLockTargets`](../rdf/document-lock.md), sehingga page tidak perlu ditulis manual. Page tanpa kedua blok menghasilkan frontend yang sama persis seperti sebelumnya.

## Sintaks

Page dokumen yang dikunci memuat `workflowActions` seperti biasa, ditambah `documentLock`.

```json
{
    "pageId": "purchase-order",
    "workflow": {
        "statusField": "status"
    },
    "workflowActions": [
        {
            "actionId": "ordered",
            "label": "Order",
            "api": {
                "endpoint": "purchase-order/change-status",
                "payload": {
                    "purchase_order_id": "$primaryKey",
                    "status": "ordered"
                }
            }
        },
        {
            "actionId": "cancelled",
            "label": "Cancel",
            "api": {
                "endpoint": "purchase-order/change-status",
                "payload": {
                    "purchase_order_id": "$primaryKey",
                    "status": "cancelled"
                }
            }
        }
    ],
    "documentLock": {
        "lockField": "is_lock",
        "reasonField": "lock_reason",
        "lockedActions": ["cancelled"]
    }
}
```

Page dokumen perujuk memuat `documentLockTargets`. Contoh berikut dari page `vendor-bill`, yang header-nya merujuk PO dan barisnya merujuk baris PO serta baris penerimaan barang.

```json
{
    "pageId": "vendor-bill",
    "documentLockTargets": [
        {
            "field": "purchase_order_id",
            "lockTable": "purchase_order"
        },
        {
            "detail": "vendor_bill_item",
            "field": "purchase_order_item_id",
            "lockTable": "purchase_order",
            "lockKeyField": "purchase_order_id"
        },
        {
            "detail": "vendor_bill_item",
            "field": "good_receipt_item_id",
            "lockTable": "good_receipt",
            "lockKeyField": "good_receipt_id"
        }
    ]
}
```

## Properti `documentLock`

| Properti | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `lockField` | string | ✓ | Field di response baca yang menandai baris terkunci. `payload migrate` menulis `is_lock` |
| `reasonField` | string | ✓ | Field di response baca yang berisi alasan. `payload migrate` menulis `lock_reason` |
| `lockedActions` | array of string | ✓ | Daftar `actionId` di `workflowActions` yang dimatikan pada baris terkunci. Boleh kosong |

`lockedActions` dihitung dari `workflowActions`. Aksi masuk daftar bila status tujuannya, yaitu nilai `api.payload.<statusField>`, ada di `voidStatuses` RDF. Pada contoh di atas, `cancelled` masuk daftar karena `cancelled` adalah status void. Aksi `ordered` tidak masuk, sehingga tetap bisa dijalankan pada PO yang sudah dirujuk.

Page tanpa `workflowActions` mendapat `lockedActions` kosong. Page baru dan `payload migrate --overwrite` juga menghasilkan array kosong, karena `workflowActions` belum ada saat blok ditulis. Jalankan `payload migrate` sekali lagi setelah `workflowActions` ditambahkan agar `lockedActions` terisi.

## Properti `documentLockTargets[]`

| Properti | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `field` | string | ✓ | Nama field lookup yang menunjuk dokumen. Harus ada di `fields` page, atau di `fields` detail bila ada `detail` |
| `lockTable` | string | ✓ | Tabel header dokumen yang dirujuk. Menjadi kunci pertama pada `options.documentVersions` |
| `detail` | string | ✗ | `detailId` bila `field` berada di grid detail. Harus cocok dengan salah satu `detailId` di `details` |
| `lockKeyField` | string | ✗ | Kolom primary key header yang dibawa opsi lookup. Ditulis bila `field` menunjuk baris dokumen, bukan header |

Entri `detail` hanya ditulis untuk detail yang ikut dikonversi ke `details[]`. Detail yang tidak ikut dikonversi dilewati dengan warning saat `payload migrate`.

Pada entri ber-`lockKeyField`, `payload migrate` menambahkan kolom itu di akhir `dataSource.select` field lookup, sehingga baris opsi lookup membawa id header. Versi header dibaca dari `document_version` pada opsi yang sama.

## Perilaku di Menu Actions

Baris dengan `is_lock: true` menampilkan `Edit` dan `Delete` dalam keadaan mati, memakai mekanisme `disabled` dan ikon informasi yang sama dengan [`fieldStates`](./field-states.md). Tooltip ikon berisi `This document is locked: <lock_reason>.`.

| Aspek | Perilaku |
|-------|----------|
| `Edit` dan `Delete` | Mati dengan `disabled` dan `aria-disabled="true"` pada baris terkunci |
| Aksi di `lockedActions` | Mati pada dialog Change Status. Aksi lain di dialog yang sama tetap aktif |
| Aksi tanpa transisi | Tetap tidak tampil, seperti tanpa kunci dokumen |
| Gabungan dengan `fieldStates` | Item mati bila salah satu aturan terpenuhi. Bila keduanya berlaku, tooltip kunci dokumen didahulukan |

Menu Actions empat baris PO tampil seperti berikut.

| Baris | `Edit` dan `Delete` | `Cancel` | Tooltip |
|-------|---------------------|----------|---------|
| PO-2026-0005, `draft`, tanpa perujuk | Aktif | Aktif | Tidak ada |
| PO-2026-0007, `ordered`, dirujuk GR-2026-0003 | Mati | Mati | `This document is locked: Used by GR-2026-0003.` |
| PO-2026-0002, `received` | Mati | Tidak tampil, karena `received` tidak punya transisi | `This document is locked: Received.` |
| PO-2026-0009, `ordered`, hanya dirujuk dokumen berstatus `cancelled` | Aktif, `Delete` dijawab 409 oleh foreign key | Aktif | Tidak ada |

Nilai `is_lock` dibaca dari baris saat list digambar. Form ubah yang sudah terbuka sebelum dokumen terkunci tidak ikut mati, tetapi saat disimpan backend menjawab `409 Document locked`.

## Perilaku Plugin

Perilaku di atas berlaku di plugin `vanilla-js-auth` dan `vanilla-js-custom`. Plugin `vanilla-js-basic` hanya mematikan `Edit` dan `Delete` per baris, karena plugin itu tidak punya aksi workflow dan mengabaikan `fieldStates`.

| Plugin | `Edit` dan `Delete` | Aksi di `lockedActions` | Gabungan dengan `fieldStates` |
|--------|:-------------------:|:-----------------------:|:-----------------------------:|
| `vanilla-js-auth` | ✓ | ✓ | ✓ |
| `vanilla-js-custom` | ✓ | ✓ | ✓ |
| `vanilla-js-basic` | ✓ | ✗ | ✗ |

## Tampilan Error 409

Dua jenis response 409 punya tampilan berbeda. Tidak ada dialog baru.

| Response | Tampilan |
|----------|----------|
| `409 Document locked` atau `409 Document busy` saat simpan | Ringkasan error di form dan toast, berisi `message` dari server. Form tetap terbuka |
| `409 Conflict` atas dokumen yang dirujuk | Banner peringatan di form berisi `message` dan ajakan memilih ulang dokumen yang dirujuk |

Banner konflik memakai komponen yang sama dengan konflik versi baris. Form juga mengosongkan field perujuk yang basi dengan pesan `Please reselect <label> to load the latest version.`, lalu memuat ulang data lookup agar pilihan berikutnya membawa versi terbaru. Bila konflik datang saat form sudah tertutup, pesan tampil sebagai notifikasi peringatan.

Konflik dokumen yang dirujuk didahulukan sebelum konflik versi baris dokumen yang sedang disimpan.

## Form Perujuk

Form perujuk mencatat `document_version` dari opsi lookup yang dipilih, baik di field header maupun di grid detail. Saat simpan, versi dikirim lewat `options.documentVersions`.

```json
{
    "data": {
        "purchase_order_id": "po_0007",
        "doc_number": "GR-2026-0003"
    },
    "options": {
        "documentVersions": {
            "purchase_order": {
                "po_0007": 3
            }
        }
    }
}
```

Pencatatan versi mengikuti aturan berikut.

- Versi dicatat saat opsi dipilih, baik di lookup select, grid detail, maupun modal pilih baris.
- Versi dibuang saat panel form dibuka, sehingga tidak ada versi sisa dari dokumen sebelumnya.
- Saat simpan, hanya versi untuk nilai yang masih terpakai yang dikirim.

Perilaku ini bergantung pada field perujuk yang berupa lookup. Jalankan `payload sync --expand-fk` sebelum `payload migrate`. Tanpa itu, foreign key menjadi field teks biasa dan tidak ada opsi lookup yang membawa versi.

## Aturan Validasi

Validator designer memeriksa kedua blok. Pesan diawali `[<pageId>]`.

| Aturan | Pesan Error |
|--------|-------------|
| `documentLock` harus object | `documentLock must be an object` |
| `lockField` dan `reasonField` string tidak kosong | `documentLock.lockField must be a non-empty string` |
| `lockedActions` harus array | `documentLock.lockedActions must be an array of strings` |
| Isi `lockedActions` berupa string | `documentLock.lockedActions[<i>] must be a string` |
| Isi `lockedActions` ada di `workflowActions` | `documentLock.lockedActions[<i>] '<actionId>' is not an actionId in workflowActions` |
| `documentLockTargets` harus array | `documentLockTargets must be an array` |
| Entri berupa object | `documentLockTargets[<i>] must be an object` |
| `field` dan `lockTable` string tidak kosong | `documentLockTargets[<i>].field must be a non-empty string` |
| `lockKeyField` string tidak kosong bila ada | `documentLockTargets[<i>].lockKeyField must be a non-empty string` |
| `field` ada di `fields` page | `documentLockTargets[<i>].field '<field>' is not declared in fields` |
| `detail` cocok dengan `detailId` | `documentLockTargets[<i>].detail '<detail>' does not match any detailId in details` |
| `field` ada di `fields` detail | `documentLockTargets[<i>].field '<field>' is not declared in fields of detail '<detail>'` |

## Milik Generator

Kedua blok milik `payload migrate`. Nilai yang disunting manual ditimpa tanpa peringatan pada migrate berikutnya, dan blok dihapus bila RDF tidak lagi memilikinya. Untuk menambah atau mengubah aksi yang dimatikan, ubah `workflowActions`, lalu jalankan `payload migrate` ulang.

## Batasan

- Field perujuk yang memakai `dependsOn` (cascade) belum mencatat versi.
- Detail satu baris (`one`) belum mencatat versi.
- Mode duplikasi membawa foreign key tanpa pemilihan ulang, sehingga backend meminta versi dan pengguna harus memilih ulang dokumen yang dirujuk.
- Lookup modal statis pada field header tidak dimuat ulang setelah konflik.

---

← [`field-states.md`](./field-states.md) | [Selanjutnya: `navigation.md`](./navigation.md) →
