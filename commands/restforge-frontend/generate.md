# `restforge-designer generate`

> Generate aplikasi frontend (HTML/JS/CSS) dari UDF.

## Sintaks

```bash
restforge-designer generate --payload <PAYLOAD> --output <OUTPUT> [OPTIONS]
```

## Flag

| Flag | Tipe | Default | Wajib | Keterangan |
|------|------|---------|:-----:|-----------|
| `-p`, `--payload <PAYLOAD>` | path | — | ✓ | Path ke file UDF JSON |
| `-o`, `--output <OUTPUT>` | path | — | ✓ | Folder output untuk file yang di-generate |
| `--plugin <PLUGIN>` | string | dari `appConfig.plugin` | ✗ | Override plugin ID dari UDF |
| `--overwrite` | flag | `false` | ✗ | Timpa file existing di folder output |
| `--scope <SCOPE>` | enum | `app` | ✗ | Scope generate: `app` (semua) atau `form` (satu halaman saja) |
| `--page <PAGE>` | string | — | ✓ jika `--scope=form` | Page ID target untuk scope `form` |
| `--skip-shared` | flag | `false` | ✗ | Skip generate shared files (`index.html`, `js/common.js`, CSS, dll.) |
| `--plugins-dir <PATH>` | string | auto-detect | ✗ | Override folder plugins (env: `RESTFORGE_DESIGNER_PLUGINS_DIR`) |
| `--license <KEY>` | string | dari storage/.env/env var | ✗ | License key override per-invocation |

## Contoh Penggunaan

### Generate Lengkap (Scope `app`)

```bash
restforge-designer generate --payload=payload/contact-mgmt.json --output=./apps/contact-mgmt-app
```

Generate semua page, shared files, dan plugin assets.

### Overwrite Folder Existing

```bash
restforge-designer generate --payload=payload/contact-mgmt.json --output=./apps/contact-mgmt-app --overwrite
```

Tanpa `--overwrite`, generate gagal jika folder output sudah berisi file.

### Generate Satu Page Saja (Scope `form`)

```bash
restforge-designer generate --payload=payload/contact-mgmt.json --output=./apps/contact-mgmt-app --scope=form --page=contact --overwrite
```

Hanya regenerate `contact.html` + `js/contact.js`. Shared files tetap di-generate kecuali ditambah `--skip-shared`.

### Skip Shared Files

```bash
restforge-designer generate --payload=payload/contact-mgmt.json --output=./apps/contact-mgmt-app --scope=form --page=contact --skip-shared --overwrite
```

Hanya page yang ditarget yang di-generate. `index.html`, `app-start.bat`, `js/common.js` tidak disentuh.

### Override Plugin dari UDF

```bash
restforge-designer generate --payload=payload/contact-mgmt.json --output=./apps/contact-mgmt-app --plugin=vanilla-js-auth --overwrite
```

Generate dengan plugin berbeda dari yang tertulis di `appConfig.plugin`. Berguna untuk membandingkan output antar plugin tanpa mengubah UDF.

### Override License Per-Invocation

```bash
restforge-designer generate --payload=payload/contact-mgmt.json --output=./apps/contact-mgmt-app --license=ABCD-1234-EFGH-5678
```

## Scope Generate

| Scope | Behavior |
|-------|----------|
| `app` (default) | Generate semua page + shared files + plugin assets |
| `form` | Generate satu page saja (`--page=<id>` wajib). Shared files tetap (kecuali `--skip-shared`) |

Scope `form` berguna untuk:

- Iterasi cepat saat develop satu page tanpa regenerate semua
- Update parsial di production tanpa overwrite shared files yang sudah custom

## File yang Di-generate

### Shared Files (Per Aplikasi)

| File | Fungsi |
|------|--------|
| `index.html` | Homepage atau redirect ke `homepage` page |
| `js/common.js` | Utility bersama (notifikasi, format, validasi) |
| `css/global-list.css` | Design system CSS |
| `app-start.bat` | Script Windows untuk run `npx serve -l <port>` di port dari `appConfig.port` |

### Per-Page Files

| File | Fungsi |
|------|--------|
| `{pageId}.html` | Halaman CRUD atau dashboard |
| `js/{pageId}.js` | Logic CRUD: DataTables init, save/load/delete, validasi |

### Plugin Assets

File vendor (jQuery, Select2, Flatpickr, Bootstrap, dll.) di-copy dari plugin ke folder output. Persisnya bervariasi per plugin.

## Behavior

Verb `generate`:

1. **Validasi license** (verb memerlukan license aktif)
2. **Membaca** UDF dari path `--payload`
3. **Muat plugin** dari `appConfig.plugin` atau override `--plugin`
4. **Jalankan validator** UDF — error membatalkan generate
5. **Hitung daftar file** yang akan dihasilkan
6. **Cek folder output**:
   - Jika kosong atau tidak ada → buat folder dan tulis file
   - Jika ada file → error kecuali ditambah `--overwrite`
7. **Tulis** file HTML/JS/CSS/aset ke folder output
8. **Tampilkan ringkasan**: jumlah file written, daftar warning

## Exit Code

| Exit Code | Kondisi |
|-----------|---------|
| `0` | Generate sukses |
| `1` | Validasi gagal, license invalid, folder existing tanpa `--overwrite`, plugin tidak ditemukan, atau IO error |

## Menjalankan Hasil Generate

Setelah generate selesai:

```bash
cd ./apps/contact-mgmt-app
app-start.bat
```

Script `app-start.bat` menjalankan `npx serve -l <port>` di port yang ditentukan oleh `appConfig.port` di UDF. Default port: `8080`.

URL aplikasi: `http://localhost:<port>/index.html`.

## Use Case Umum

| Skenario | Command |
|----------|---------|
| First generate setelah scaffold project | `generate --payload=... --output=./apps/contact-mgmt-app` |
| Re-generate setelah edit UDF | `generate --payload=... --output=./apps/contact-mgmt-app --overwrite` |
| Iterasi cepat 1 page | `generate --payload=... --output=./apps/contact-mgmt-app --scope=form --page=contact --overwrite` |
| Bandingkan output plugin | `generate ... --plugin=vanilla-js-auth --output=./apps/contact-mgmt-app-auth` |

---

**Lihat juga**: [`README`](./README.md) · [`init.md`](./init.md) · [`validate.md`](./validate.md) · [`preview.md`](./preview.md) · [`catalogs/udf/`](../../catalogs/udf/)
