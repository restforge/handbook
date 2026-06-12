# Commands Reference

> Daftar lengkap perintah CLI di ekosistem RESTForge, dikelompokkan per binary.

## Dua Binary CLI

RESTForge ekosistem terdiri dari **dua binary CLI yang terpisah**, masing-masing dengan pattern command sendiri:

| Binary | Pattern Command | Tujuan | Dokumentasi |
|--------|----------------|--------|-------------|
| `restforge` | `npx restforge <resource> <verb> [options]` | Backend: runtime server + generator (SDF, RDF, processor, kafka, dll.) | [`restforge-backend/`](./restforge-backend/) |
| `restforge-designer` | `restforge-designer <verb> [options]` | Frontend: code generator (UDF → HTML/JS/CSS) | [`restforge-frontend/`](./restforge-frontend/) |

Kedua folder berisi dokumentasi binary terpisah. Pattern command berbeda di setiap binary — detail dibahas di bagian masing-masing.

## Perintah Operasi Backend

Pattern: `npx restforge <resource> <verb> [options]`. Total **43 public command** (4 runtime server + 2 verb global + 37 CLI generator) dan **2 internal binary**.

Detail dan daftar resource lengkap ada di [`restforge-backend/README.md`](./restforge-backend/README.md).

Quick-reference resource:

| Resource | Tujuan | Folder |
|----------|--------|--------|
| `runtime` | Server runtime, validasi config, license backend | [`restforge-backend/runtime/`](./restforge-backend/runtime/) |
| `schema` | SDF: init, validate, migrate, introspect, diff | [`restforge-backend/schema/`](./restforge-backend/schema/) |
| `endpoint` | Generate endpoint REST dari RDF | [`restforge-backend/endpoint/`](./restforge-backend/endpoint/) |
| `payload` | RDF: generate, validate, diff, sync | [`restforge-backend/payload/`](./restforge-backend/payload/) |
| `processor` | Generate processor (background job, custom logic) | [`restforge-backend/processor/`](./restforge-backend/processor/) |
| `dashboard` | Generate dashboard endpoint multi-widget | [`restforge-backend/dashboard/`](./restforge-backend/dashboard/) |
| `kafka` | Generate Kafka consumer | [`restforge-backend/kafka/`](./restforge-backend/kafka/) |
| `test` | Generate integration test | [`restforge-backend/test/`](./restforge-backend/test/) |
| `key` | Manajemen API key | [`restforge-backend/key/`](./restforge-backend/key/) |
| `query` | Validasi SQL terhadap database live | [`restforge-backend/query/`](./restforge-backend/query/) |
| `data` | Export/import baris data tabel (pull/push) berbasis SDF | [`restforge-backend/data/`](./restforge-backend/data/) |
| `config` | Mengelola config schema, template, default | [`restforge-backend/config/`](./restforge-backend/config/) |
| `catalog` | Introspeksi spesifikasi internal | [`restforge-backend/catalog/`](./restforge-backend/catalog/) |
| `project` | Operasi level project (list, delete) | [`restforge-backend/project/`](./restforge-backend/project/) |

File khusus:

| Halaman | Fungsi |
|---------|--------|
| [`restforge-backend/init.md`](./restforge-backend/init.md) | Verb global `npx restforge init` |
| [`restforge-backend/fast-track.md`](./restforge-backend/fast-track.md) | Verb global `npx restforge fast-track` (generate REST API + frontend dari SDF) |
| [`restforge-backend/internal-binary.md`](./restforge-backend/internal-binary.md) | Dokumentasi 2 binary internal (`restforge-consumer`, `restforge-consumer-deploy`) |
| [`restforge-backend/conventions.md`](./restforge-backend/conventions.md) | Konvensi flag-only, global options |

## Perintah Operasi Frontend

Pattern: `restforge-designer <verb> [options]`. Total **11 verb** (2 di antaranya parent dengan subcommand: `license`, `plugins`).

Detail dan daftar verb lengkap ada di [`restforge-frontend/README.md`](./restforge-frontend/README.md).

Quick-reference verb:

| Verb | Tujuan | Halaman |
|------|--------|---------|
| `init` | Scaffold project baru dari plugin | [`restforge-frontend/init.md`](./restforge-frontend/init.md) |
| `validate` | Validasi UDF | [`restforge-frontend/validate.md`](./restforge-frontend/validate.md) |
| `preview` | Preview daftar file output | [`restforge-frontend/preview.md`](./restforge-frontend/preview.md) |
| `generate` | Generate aplikasi frontend dari UDF | [`restforge-frontend/generate.md`](./restforge-frontend/generate.md) |
| `activate` | Setup license | [`restforge-frontend/activate.md`](./restforge-frontend/activate.md) |
| `deactivate` | Hapus license | [`restforge-frontend/deactivate.md`](./restforge-frontend/deactivate.md) |
| `license status` | Tampilkan status license | [`restforge-frontend/license-status.md`](./restforge-frontend/license-status.md) |
| `license migrate` | Pindah license dari `.env` ke encrypted storage | [`restforge-frontend/license-migrate.md`](./restforge-frontend/license-migrate.md) |
| `plugins list` | Daftar plugin ter-install | [`restforge-frontend/plugins-list.md`](./restforge-frontend/plugins-list.md) |
| `plugins inspect` | Metadata plugin tertentu | [`restforge-frontend/plugins-inspect.md`](./restforge-frontend/plugins-inspect.md) |
| `plugins scaffold` | Scaffold plugin custom baru | [`restforge-frontend/plugins-scaffold.md`](./restforge-frontend/plugins-scaffold.md) |

File khusus:

| Halaman | Fungsi |
|---------|--------|
| [`restforge-frontend/conventions.md`](./restforge-frontend/conventions.md) | Konvensi flag, format license, env var |

## Konvensi Antar Binary

Konvensi per binary ada di file `conventions.md` masing-masing:

- Backend: [`restforge-backend/conventions.md`](./restforge-backend/conventions.md)
- Frontend: [`restforge-frontend/conventions.md`](./restforge-frontend/conventions.md)

---

**Lihat juga**: [`catalogs/`](../catalogs/) · [`README`](../README.md)
