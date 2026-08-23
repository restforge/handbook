# Sistem Tipe

Tipe bersifat logical, bukan physical. DDL generator akan memetakan ke tipe SQL native sesuai dialect target.

| Shorthand | Modifier | Deskripsi |
|-----------|----------|-----------|
| `string:N` | length wajib | String dengan panjang maksimum N karakter |
| `text` | tidak ada | String panjang tanpa batas spesifik. Diturunkan ke RDF `text` (unbounded) dan dirender sebagai UI `textarea`: `text` → RDF `text` → UDF `textarea` |
| `integer` | tidak ada | Bilangan bulat ukuran standar |
| `bigint` | tidak ada | Bilangan bulat ukuran besar |
| `decimal:M,N` | precision, scale wajib | Bilangan desimal presisi tetap |
| `boolean` | tidak ada | Nilai benar/salah |
| `date` | tidak ada | Tanggal tanpa waktu |
| `timestamp` | tidak ada | Tanggal dengan waktu |
| `uuid` | tidak ada | Universally unique identifier |
| `json` | tidak ada | Struktur data JSON |
| `geom:Type,SRID` | Type dan SRID sama-sama opsional (default `Point,4326`) | Tipe data spasial (PostGIS geometry). Hanya didukung dialect PostgreSQL, detail di bagian [Tipe `geom` (PostGIS)](#tipe-geom-postgis) |

**Contoh penggunaan:**

```javascript
fields: {
  product_code:   'string:20',
  description:    'text',
  stock_qty:      'integer',
  total_amount:   'decimal:15,2',
  is_active:      'boolean',
  register_date:  'date',
  created_at:     'timestamp',
  product_id:     'uuid',
  metadata:       'json',
  location:       'geom:Point,4326'
}
```

## Tipe `geom` (PostGIS)

Tipe spasial berbasis PostGIS. Berlaku hanya untuk dialect PostgreSQL. Schema
yang memuat field `geom` akan ditolak oleh dialect lain sejak tahap generate DDL
(lihat bagian [Penolakan Dialect Non-PostgreSQL](#penolakan-dialect-non-postgresql)
di bawah).

### Sintaks

```
geom               →  geomType='Point',  srid=4326  (default penuh)
geom:Polygon       →  geomType='Polygon', srid=4326  (SRID default)
geom:Polygon,3857  →  geomType='Polygon', srid=3857
```

Modifier mengikuti pola sama dengan `decimal:M,N`: dipisah titik dua, bagian
kedua dipisah koma. SRID wajib integer positif; modifier dengan lebih dari satu
koma atau SRID bukan integer positif akan gagal validasi saat schema dimuat.

Geometry type valid (case-insensitive saat parse, dinormalisasi ke bentuk
kanonik Pascal-case): `Point`, `LineString`, `Polygon`, `MultiPoint`,
`MultiLineString`, `MultiPolygon`, `GeometryCollection`, `Geometry`. `geom:point`
dan `geom:POINT` sama-sama dinormalisasi menjadi `Point`.

### Constraint yang Diizinkan

Hanya `notnull` dan `index` yang boleh dipasangkan pada field bertipe `geom`:

```javascript
fields: {
  geom: 'geom:Point,4326 notnull index'
}
```

Constraint `pk`, `unique`, `default`, dan `fk` tidak diizinkan pada field-level
maupun lewat jalur array-level (`options.primaryKey` composite,
`options.uniques`, entry composite di `options.indexes`). Entry single-column
pada `options.indexes` tetap diizinkan dan akan di-emit sebagai index GiST (lihat
bagian [Index GiST Otomatis](#index-gist-otomatis)).

### Pesan Error

| Kondisi | Pesan Error |
|---------|-------------|
| Modifier lebih dari satu koma | `Field '<field>': geom modifier must be 'Type' or 'Type,SRID', got '<modifier>'` |
| Geometry type tidak dikenal | `Field '<field>': unknown geometry type '<value>' (expected one of: Point, LineString, Polygon, MultiPoint, MultiLineString, MultiPolygon, GeometryCollection, Geometry)` |
| SRID bukan integer positif | `Field '<field>': geom SRID must be a positive integer, got '<value>'` |
| Constraint `pk`/`unique`/`default`/`fk` pada field-level | `Field '<field>': constraint '<name>' is not allowed on type 'geom' (only 'notnull' and 'index' are permitted)` |
| Field geom di `options.primaryKey` (composite) | `Table '<table>': primaryKey cannot include field '<field>': type 'geom' is not allowed in a primary key` |
| Field geom di `options.uniques` | `Table '<table>': unique constraint cannot include field '<field>': type 'geom' is not allowed in a unique constraint` |
| Field geom di entry composite `options.indexes` | `Table '<table>': composite index cannot include field '<field>': type 'geom' is not allowed in a composite index (requires 'btree_gist' extension, not supported)` |

### Pemetaan DDL PostgreSQL

Kolom dipetakan ke `geometry(Type,SRID)`:

```javascript
fields: {
  mck_id: 'string:36 pk',
  status: 'string:20 notnull index',
  geom:   'geom:Point,4326 notnull index'
}
```

menghasilkan:

```sql
CREATE EXTENSION IF NOT EXISTS postgis;

CREATE TABLE IF NOT EXISTS mck (
    mck_id VARCHAR(36) NOT NULL,
    status VARCHAR(20) NOT NULL,
    geom geometry(Point,4326) NOT NULL,
    CONSTRAINT pk_mck PRIMARY KEY (mck_id)
);

CREATE INDEX IF NOT EXISTS idx_mck_status ON mck (status);

CREATE INDEX IF NOT EXISTS idx_mck_geom ON mck USING GIST (geom);
```

Statement `CREATE EXTENSION IF NOT EXISTS postgis;` otomatis disisipkan sebagai
statement pertama begitu minimal satu field bertipe `geom` ada di schema yang
sedang di-generate, tanpa perlu dideklarasikan manual. Berlaku juga pada jalur
`schema apply` (ALTER, penambahan kolom geom pada tabel yang sudah ada): statement
tersebut tampil di bawah header `[(database)]` (bukan terikat ke satu tabel),
baik pada dry-run maupun eksekusi sungguhan.

### Index GiST Otomatis

Constraint `index` (field-level maupun single-column di `options.indexes`) pada
field bertipe `geom` di-emit sebagai `CREATE INDEX ... USING GIST (...)`, bukan
btree biasa seperti tipe lain. Berlaku konsisten pada jalur `schema migrate`
(CREATE TABLE) maupun `schema apply` (ALTER TABLE), sehingga tidak ada drift
antara keduanya. Lihat [`constraints.md`](./constraints.md#constraint-index)
untuk perilaku umum constraint `index` di luar tipe `geom`.

### Penolakan Dialect Non-PostgreSQL

Dialect mysql, oracle, dan sqlite menolak field bertipe `geom` dengan pesan:

```
Type 'geom' is only supported on PostgreSQL (PostGIS extension); <Dialect> does not support spatial geometry types.
```

`<Dialect>` tersubstitusi sesuai target (`MySQL`, `Oracle`, `SQLite`). Penolakan
terjadi pada tahap pemetaan tipe kolom saat `CREATE TABLE` di-generate, sehingga
berlaku untuk `schema migrate`, `schema apply`, maupun `schema generate-ddl`.

### Introspeksi dan Diff

`schema diff` memetakan kolom `geometry(Type,SRID)` di database kembali ke tipe
`geom` beserta `geomType` dan SRID-nya, sehingga field geom yang konsisten antara
SDF dan database tidak muncul sebagai drift. Bila metadata PostGIS di kolom tidak
lengkap (mis. kolom `geometry` polos tanpa Type/SRID declared), introspeksi TIDAK
diam-diam mengasumsikan `Point,4326`. Field dipetakan sebagai `Geometry` dengan
SRID `0`, disertai catatan bahwa metadata tidak lengkap, supaya perbedaan tersebut
tetap terlihat sebagai drift, bukan tersembunyi.

---

**Lihat juga**: [`sdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
