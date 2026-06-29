# `restforge` Commands

> Daftar lengkap command CLI binary `restforge` (backend: runtime server + code generator). Pasangan binarynya untuk frontend adalah [`restforge-designer`](../restforge-frontend/README.md).

`restforge` adalah binary backend yang menjalankan runtime server sekaligus seluruh code generator (SDF, RDF, processor, kafka, dashboard, dll.). Pattern command bersifat **resource-first**: `<resource>` dan `<verb>` adalah positional wajib di posisi pertama dan kedua, sisanya flag bernama.

```
npx restforge <resource> <verb> [options]
```

Tiga pengecualian terhadap pattern resource-first:

- **Verb global** (`init`, `fast-track`) dipanggil tanpa prefix resource, mis. `npx restforge init`.
- **Command runtime server** dipanggil langsung tanpa prefix `runtime`, mis. `npx restforge serve`.
- **Internal binary** (`restforge-consumer`, `restforge-consumer-deploy`) adalah binary independen.

`npx restforge ...` ekuivalen dengan `restforge ...` bila binary sudah ter-install secara global (`npm install -g @restforgejs/platform`). Jalur instalasi utama adalah lokal via `npx create-restforge-app` (membuat folder project sekaligus install `@restforgejs/platform` lokal); global install tetap valid namun bukan jalur dominan.

## Verb Global

Verb yang tidak terikat resource, dipanggil langsung setelah binary:

| Verb | Tujuan | Halaman |
|------|--------|---------|
| `init` | Scaffold sample starter files (config, payload, query template) di working directory | [`init.md`](./init.md) |
| `fast-track` | Generate REST API + frontend app dari SDF dalam satu alur dialog interaktif | [`fast-track.md`](./fast-track.md) |

## Runtime Server

Command eksekusi server dan manajemen license. Dipanggil langsung tanpa prefix `runtime` (folder `runtime/` hanya pengelompokan dokumentasi):

| Verb | Tujuan | Halaman |
|------|--------|---------|
| `serve` | Menjalankan runtime HTTP server untuk project yang sudah di-generate | [`runtime/serve.md`](./runtime/serve.md) |
| `validate` | Memvalidasi file config database tanpa menjalankan server | [`runtime/validate.md`](./runtime/validate.md) |
| `license-info` | Menampilkan informasi license aktif pada mesin saat ini | [`runtime/license-info.md`](./runtime/license-info.md) |
| `license-deactivate` | Melepas aktivasi license dari mesin saat ini | [`runtime/license-deactivate.md`](./runtime/license-deactivate.md) |

## Resource Generator

Setiap resource memiliki satu atau lebih verb. Detail flag per verb ada di folder masing-masing:

| Resource | Verb | Tujuan | Folder |
|----------|------|--------|--------|
| `schema` | `list`, `describe`, `init`, `validate`, `models`, `generate-ddl`, `migrate`, `introspect`, `diff`, `apply`, `template` | SDF: definisi, validasi, migrasi, introspeksi, drift detection, reference template | [`schema/`](./schema/) |
| `payload` | `generate`, `validate`, `diff`, `sync`, `migrate` | RDF: kelola payload JSON sumber generate endpoint | [`payload/`](./payload/) |
| `endpoint` | `create` | Generate endpoint REST dari file payload JSON | [`endpoint/`](./endpoint/) |
| `processor` | `create` | Generate processor (background job atau business logic handler) | [`processor/`](./processor/) |
| `dashboard` | `create` | Generate dashboard endpoint multi-widget aggregator | [`dashboard/`](./dashboard/) |
| `kafka` | `consumer-create` | Generate Kafka consumer code dari payload JSON | [`kafka/`](./kafka/) |
| `test` | `generate` | Generate integration test (Jest + Supertest) untuk endpoint existing | [`test/`](./test/) |
| `key` | `generate`, `list`, `revoke` | Manajemen API key | [`key/`](./key/) |
| `query` | `validate` | Validasi statement SQL terhadap database live (EXPLAIN) | [`query/`](./query/) |
| `data` | `pull`, `push` | Export/import baris data tabel (format agnostic lintas dialect) berbasis SDF | [`data/`](./data/) |
| `config` | `schema`, `template`, `set-default`, `get-default`, `clear-default`, `list` | Kelola config schema, template, dan default config per working directory | [`config/`](./config/) |
| `catalog` | `field-validation`, `query-declarative`, `dashboard`, `dbschema` | Introspeksi spesifikasi internal (output JSON catalog) | [`catalog/`](./catalog/) |
| `project` | `list`, `delete`, `auth`, `sdk` | Operasi level project | [`project/`](./project/) |

## Internal Binary

Binary independen di luar pattern resource-first, ditujukan untuk maintainer atau advanced user:

| Binary | Fungsi | Halaman |
|--------|--------|---------|
| `restforge-consumer` | Menjalankan Kafka consumer runtime untuk consumer yang sudah di-generate dalam project | [`internal-binary.md`](./internal-binary.md) |
| `restforge-consumer-deploy` | Generate file PM2 ecosystem config untuk deploy multi-process consumer ke production | [`internal-binary.md`](./internal-binary.md) |

## Konvensi

Konvensi flag-only, global options (`--help`, `--version`), resolusi `--config`, semantik exit code, dan aturan penulisan single-line dijelaskan di [`conventions.md`](./conventions.md).

## Alur Workflow Umum

Alur tipikal dari inisialisasi project hingga runtime server berjalan:

```
┌────────┐   ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐   ┌────────┐
│  init  │ → │  schema migrate  │ → │ payload generate │ → │  endpoint create │ → │ serve  │
└────────┘   └──────────────────┘   └──────────────────┘   └──────────────────┘   └────────┘
  scaffold      SDF → database          RDF per tabel         endpoint REST         runtime
```

| Tahap | Command | Frekuensi |
|-------|---------|-----------|
| Inisialisasi | `npx restforge init` | Sekali per project |
| Migrate schema | `npx restforge schema migrate --schema-path=./schema --config=db.env` | Saat SDF berubah |
| Generate payload | `npx restforge payload generate --table=<NAME> --config=db.env` | Per tabel |
| Generate endpoint | `npx restforge endpoint create --project=<NAME> --payload=<FILE> --config=db.env` | Per tabel |
| Jalankan server | `npx restforge serve --project=<NAME> --config=db.env` | Saat menjalankan API |

Untuk men-generate seluruh alur di atas sekaligus secara interaktif (termasuk frontend app), tersedia verb global [`fast-track`](./fast-track.md).

---

**Lihat juga**: [`commands/`](../) · [`restforge-frontend/`](../restforge-frontend/README.md) · [`conventions`](./conventions.md) · [`catalogs/`](../../catalogs/) · [`README`](../../README.md)
