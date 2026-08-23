# `npx restforge-designer init`

> Scaffold project frontend baru dari plugin scaffold.

## Sintaks

```bash
npx restforge-designer init [OPTIONS]
```

Tanpa `--plugin` dan `--output`, command masuk mode **interaktif** dengan prompt.

## Flag

| Flag | Tipe | Default | Wajib | Keterangan |
|------|------|---------|:-----:|-----------|
| `--plugin <PLUGIN>` | string | — | ✓ (non-interactive) | Plugin ID, contoh `vanilla-js-basic`, `vanilla-js-auth` |
| `-o`, `--output <OUTPUT>` | path | — | ✓ (non-interactive) | Folder output project |
| `--app-name <APP_NAME>` | string | `"My Application"` | ✗ | Nama aplikasi |
| `--app-code <APP_CODE>` | string | derived dari `app-name` | ✗ | Kode aplikasi kebab-case |
| `--api-base-url <URL>` | string | `http://localhost:3032/api` | ✗ | Base URL backend API |
| `--auth-app-code <CODE>` | string | `XXXXXXXXX` | ✗ | Auth application code (untuk plugin `vanilla-js-auth`) |
| `--auth-api-url <URL>` | string | `https://restforge.dev/api/auth` | ✗ | Auth API URL (untuk plugin `vanilla-js-auth`) |
| `--port <PORT>` | integer | `8080` | ✗ | Port HTTP server untuk preview (range 1–65535) |
| `--idle-timeout <MINUTES>` | integer | `30` | ✗ | Idle timeout dalam menit (untuk plugin `vanilla-js-auth`) |
| `--no-auth` | flag | `false` | ✗ | Inisialisasi project tanpa autentikasi |
| `--overwrite` | flag | `false` | ✗ | Timpa folder output jika sudah ada |
| `--plugins-dir <PATH>` | string | auto-detect | ✗ | Override folder plugins (env: `RESTFORGE_DESIGNER_PLUGINS_DIR`) |

## Contoh Penggunaan

### Mode Interaktif

Tanpa `--plugin` dan `--output`, masuk mode prompt:

```bash
npx restforge-designer init
```

CLI akan meminta input plugin dan folder output secara interaktif.

### Mode Non-Interaktif (Plugin Basic)

```bash
npx restforge-designer init --plugin=vanilla-js-basic --output=./myapp --app-name="Contact Management" --app-code=contact-mgmt --api-base-url=http://localhost:3031/api/dbcontact --port=8000
```

### Project dengan Plugin Auth

```bash
npx restforge-designer init --plugin=vanilla-js-auth --output=./secure-app --app-name="Secure App" --auth-app-code=ABCD12345E --auth-api-url=https://auth.example.com/api/auth --idle-timeout=15
```

### Project Tanpa Auth (Override Plugin Default)

```bash
npx restforge-designer init --plugin=vanilla-js-auth --output=./simple-app --no-auth
```

### Overwrite Folder Existing

```bash
npx restforge-designer init --plugin=vanilla-js-basic --output=./myapp --overwrite
```

## Output yang Dihasilkan

`init` membuat folder project dengan:

| File/Folder | Fungsi |
|-------------|--------|
| `payload/<app-code>.json` | UDF sample sesuai plugin |
| `app-start.bat` | Script Windows untuk run `npx serve` di port yang ditentukan |
| `README.md` | Petunjuk dasar project |
| Folder plugin files | Aset dari plugin (CSS, vendor JS, dll. — bergantung plugin) |

Struktur persis bervariasi per plugin. Jalankan `npx restforge-designer plugins inspect --plugin=<plugin-id>` untuk melihat file yang di-scaffold per plugin.

## Validasi

| Aturan | Behavior |
|--------|----------|
| `--port` integer 1–65535 | Error jika di luar range |
| `--output` folder belum ada | Folder akan dibuat |
| `--output` folder sudah ada | Error kecuali ditambah `--overwrite` |
| `--plugin` ID tidak dikenali | Error: plugin tidak ditemukan |

## Langkah Berikutnya

Setelah `init`:

1. Edit `payload/<app-code>.json` sesuai kebutuhan (lihat [`catalogs/udf/`](../../catalogs/udf/))
2. Validasi: `npx restforge-designer validate --payload=payload/<app-code>.json`
3. Generate: `npx restforge-designer generate --payload=payload/<app-code>.json --output=./apps/<app-code>-app`

---

**Lihat juga**: [`README`](./README.md) · [`generate.md`](./generate.md) · [`plugins-list.md`](./plugins-list.md) · [`catalogs/udf/`](../../catalogs/udf/)
