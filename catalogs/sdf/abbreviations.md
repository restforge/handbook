# Kode Singkatan Otomatis (`abbreviations`)

> Blok di level model yang membentuk kode singkatan dari field teks lain saat insert, misalnya `LPK` untuk "Laptop & Komputer".

## Ikhtisar

Tabel master seperti kategori, brand, dan vendor biasanya punya kolom kode yang dibaca manusia. Nomor urut seperti `001` atau `002` tidak mencerminkan nama data, sedangkan mengisi kode manual rawan tidak seragam. Blok `abbreviations` menyerahkan pembentukan kode ke RESTForge: saat insert, kode diambil dari huruf-huruf field nama dengan panjang maksimal yang ditentukan.

Blok ini tidak mengubah DDL. `payload generate` dan `payload sync` menurunkannya ke RDF sebagai constraint [`abbreviation`](../rdf/field-validation.md#kode-singkatan-abbreviation), lalu runtime mengisi kode setiap kali baris baru disimpan. Algoritma yang sama juga tersedia lewat [mode IDGEN `abbr`](../../features/id-generator/README.md#membuat-kode-singkatan-dari-teks).

## Bentuk Blok

Blok `abbreviations` berupa array. Satu entri menyatakan satu field target, yaitu field yang diisi kode, dan satu field sumber, yaitu field teks yang menjadi bahan singkatan.

```javascript
// schema/product_category.js
module.exports = ({ defineModel }) => defineModel('product_category', {
  fields: {
    product_category_id:   'string:36 pk',
    product_category_code: 'string:32 unique notnull',
    product_category_name: 'string:128 notnull',
    is_active:             'boolean default:true'
  },
  abbreviations: [
    { field: 'product_category_code', from: 'product_category_name', maxLength: 3 }
  ]
});
```

Dengan SDF di atas, kategori "Laptop & Komputer" disimpan dengan kode `LPK`, dan "Alat Kebersihan Kantor" dengan kode `AKK`.

## Properti

| Properti | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `field` | string | ✓ | Field target. Harus bertipe `string` dan unique, baik lewat token `unique` maupun lewat unique gabungan di `uniques` |
| `from` | string | ✓ | Field sumber. Harus bertipe `string` atau `text` di tabel yang sama dan berbeda dari `field` |
| `maxLength` | integer | ✓ | Panjang maksimal kode, minimal 1 dan tidak melebihi panjang kolom field target |
| `ignoreWords` | array of string | ✗ | Kata tambahan yang dilewati, misalnya `['PT', 'CV', 'TBK']`. Setiap kata hanya berisi huruf dan angka |

## Cara Kode Dibentuk

Kode dibentuk dari nilai field sumber dengan langkah berikut. Hasilnya disebut kode dasar.

1. Akronim di dalam kurung dipakai langsung bila hanya berisi huruf besar dan angka, memuat minimal satu huruf, dan muat di `maxLength`. "Alat Pelindung Diri (APD)" menghasilkan `APD`.
2. Bila tidak ada akronim seperti itu, isi kurung dibuang, aksen dihapus, dan teks diubah ke huruf besar. "Café & Résto" menjadi `CAFE & RESTO`.
3. Teks dipecah menjadi kata pada setiap karakter selain huruf `A` sampai `Z` dan angka. Simbol seperti `&` tidak pernah menjadi kata.
4. Kata `DAN`, `ATAU`, `DI`, `KE`, `DARI`, `YANG`, `UNTUK`, `AND`, `OF`, `THE`, `FOR`, dan kata di `ignoreWords` dilewati tanpa membedakan huruf besar dan kecil. Bila semua kata terlewati, kata-kata itu dipakai kembali.
5. Bila jumlah kata sama dengan atau lebih dari `maxLength`, kode diambil dari huruf pertama kata-kata terdepan. "Sparepart Mesin Produksi Berat" menghasilkan `SMP`.
6. Bila kata lebih sedikit, setiap kata menyumbang huruf pertamanya, lalu sisa tempat diisi bergiliran mulai dari kata pertama dengan karakter berikutnya yang bukan vokal. Huruf setiap kata tetap berada di posisi kata itu. "Laptop & Komputer" menghasilkan `LPK`, dan "Kabel" menghasilkan `KBL`.
7. Kode boleh lebih pendek dari `maxLength` bila karakter bukan vokal sudah habis. "TV" tetap `TV`.

Tabel berikut berisi contoh hasilnya.

| Teks | `maxLength` | `ignoreWords` | Kode dasar |
|------|:-----------:|---------------|-----------|
| Aksesoris Komputer | 3 | | `AKK` |
| Furnitur dan Interior | 3 | | `FRI` |
| Kesehatan dan Keselamatan Kerja (K3) | 3 | | `K3` |
| Peralatan Listrik (Elektrikal) | 3 | | `PRL` |
| Kabel 3 Meter | 3 | | `K3M` |
| PT Sinar Jaya Abadi | 3 | | `PSJ` |
| PT Sinar Jaya Abadi | 3 | `PT`, `CV`, `TBK` | `SJA` |
| Laptop & Komputer | 5 | | `LPTKM` |
| Kemasan | 5 | | `KMSN` |

## Kode yang Sudah Dipakai

Bila kode dasar sudah dipakai baris lain, ujung kode diganti angka mulai dari `2` tanpa melewati `maxLength`. Kode dasar yang lebih pendek dari `maxLength` cukup ditambah angka di belakangnya.

| Kondisi tabel | Kode dasar | Kode tersimpan |
|---------------|-----------|----------------|
| `AKK` belum dipakai | `AKK` | `AKK` |
| `AKK` sudah dipakai | `AKK` | `AK2` |
| `AKK` sampai `AK9` sudah dipakai | `AKK` | `A10` |
| `AKK` sampai `A99` sudah dipakai | `AKK` | Ditolak `ABBREVIATION_EXHAUSTED` |
| `TV` sudah dipakai | `TV` | `TV2` |

Pemeriksaan ini membaca isi tabel, bukan Redis, dengan cakupan yang sama seperti unique field target. Pada project multi-tenant, unique field target berbentuk `(tenant_id, kode)`, sehingga setiap tenant punya urutan kodenya sendiri. Kode milik baris yang sudah dihapus boleh dipakai lagi oleh baris baru.

## Perilaku Saat Insert dan Update

Kode hanya dibentuk saat insert, termasuk header dan detail pada composite insert. Bila request atau handler `onBefore*` sudah mengisi field target, nilai itu disimpan apa adanya tanpa diberi angka. Kode tetap sama walaupun field sumber diubah lewat update. Rincian response error ada di [constraint `abbreviation`](../rdf/field-validation.md#kode-singkatan-abbreviation).

Di frontend hasil `payload migrate`, field target tampil `readonly` dan tidak wajib diisi, sehingga form Add bisa disimpan dengan kode kosong. `defaultValue` bersumber IDGEN pada field itu dibuang saat migrate, karena nilai itu ikut terkirim dan menggantikan kode singkatan.

## Aturan Validasi

Pelanggaran aturan berikut dilaporkan saat file schema dimuat, dengan awalan `Failed to load schema file '<file>':`.

| No | Aturan | Pesan Error |
|----|--------|-------------|
| 1 | `field` ada di `fields` dan hanya muncul sekali di blok | `abbreviations field '<field>' in <table> is not a field` |
| 2 | Field target bertipe `string` | `abbreviations field '<field>' in <table> must be a string field, got '<type>'` |
| 3 | Field target unique | `abbreviations field '<field>' in <table> must be unique` |
| 4 | `from` ada di `fields`, bertipe `string` atau `text`, dan berbeda dari `field` | `abbreviations from '<field>' in <table> must be a string or text field, got '<type>'` |
| 5 | `maxLength` integer minimal 1 dan tidak melebihi panjang kolom | `abbreviations maxLength <n> for '<field>' in <table> exceeds the column length <length>` |
| 6 | `ignoreWords` berupa array kata yang hanya berisi huruf dan angka | `abbreviations ignoreWords for '<field>' in <table> must be an array of words containing only letters and digits` |

Key di luar `field`, `from`, `maxLength`, dan `ignoreWords` ditolak dengan pesan `abbreviations.<key> in <table> is not a recognized key`.

## Bantuan di CLI

Section `abbreviations` tersedia di katalog dbschema, termasuk algoritma dan aturan di atas.

```bat
npx restforge catalog dbschema --section=abbreviations
```

## Batasan

- Kode dari request tidak diperiksa terhadap algoritma. Nilai yang lebih panjang dari `maxLength` ditolak, sedangkan nilai yang sudah dipakai ditolak unique database dengan 409 `UNIQUE_VIOLATION`.
- Satu kode dasar menampung jumlah baris terbatas. Dengan `maxLength` 3, satu kode dasar menampung 99 baris sebelum ditolak.

---

**Lihat juga**: [`composite-unique.md`](./composite-unique.md) · [`constraints.md`](./constraints.md) · [`sdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
