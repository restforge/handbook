# `init` (global verb)

> Generate starter config file (`config/db-connection.env`) di working directory saat ini. Berjalan interaktif di terminal (prompt LICENSE, database, dan nama file config), atau non-interaktif dengan `--force`. Berguna untuk inisialisasi project baru.

Verb `init` adalah verb global yang tidak terikat pada resource tertentu, sehingga pemanggilannya cukup `npx restforge init` tanpa prefix resource.

## Pattern

```
npx restforge init [options]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--force` | Tidak | `false` | Lewati dialog interaktif dan timpa file existing tanpa konfirmasi (tulis template default) |

## Mode Operasi

| Invocation | Mode | Perilaku |
|------------|------|----------|
| `npx restforge init` (di terminal/TTY) | Interaktif | Dialog input LICENSE, database, dan nama file config, lalu konfirmasi sebelum menulis |
| `npx restforge init --force` | Non-interaktif | Tulis template default ke `config/db-connection.env` (timpa bila sudah ada) |
| Dipanggil dari pipe/CI/orchestrator (stdin bukan TTY) | Non-interaktif | Tulis template default; bila file sudah ada tanpa `--force`, command berhenti dengan error |

Pada lingkungan non-interaktif (CI/CD atau saat di-orkestrasi command lain seperti `fast-track`), dialog dilewati agar command tidak menggantung menunggu input. Nilai pada template tetap berupa placeholder yang perlu diedit setelahnya.

## Alur Dialog Interaktif

Pada mode interaktif, command meminta input berurutan lalu menampilkan review sebelum menulis:

1. **LICENSE** — license key runtime.
2. **Tipe database dan atributnya** — `DB_TYPE` menentukan atribut yang diminta berikutnya (lihat [Input Konfigurasi](#input-konfigurasi)).
3. **Nama file config** — default `db-connection.env`, dapat dikustomisasi. File ditulis ke folder `config/`.
4. **Review dan konfirmasi** — ringkasan license (disamarkan), database, dan target file ditampilkan, lalu konfirmasi `Y/n` sebelum penulisan.

Tekan Enter pada setiap prompt untuk memakai nilai default.

## Input Konfigurasi

| Prompt | Default | Keterangan |
|--------|---------|-----------|
| `LICENSE` | `XXXX-XXXX-XXXX-XXXX` | License key runtime. Placeholder default perlu diganti dengan license asli |
| `DB_TYPE` | `postgresql` | Pilihan: `postgresql`, `mysql`, `oracle`, `sqlite` |
| `DB_HOST` / `DB_PORT` / `DB_USER` / `DB_PASSWORD` / `DB_NAME` | menyesuaikan `DB_TYPE` | Kredensial database (non-sqlite). Default port/user mengikuti dialect: `postgresql` → `5432`/`postgres`, `mysql` → `3306`/`root`, `oracle` → `1521`/`oracle` (DB_NAME `ORCL`) |
| `DB_FILE` | `./data/myapp.db` | Path file database (hanya bila `DB_TYPE=sqlite`); nilai ditulis ke `DB_NAME` |
| Nama file config | `db-connection.env` | Nama file target di folder `config/` |

Pada mode `sqlite`, parameter `DB_HOST`, `DB_PORT`, `DB_USER`, dan `DB_PASSWORD` diabaikan. Seluruh nilai input diterapkan ke template `db-connection.env`; baris konfigurasi lain (Redis, Kafka, Logging, dll.) tetap memakai nilai default template.

## File yang Di-generate

| Path | Keterangan |
|------|-----------|
| `config/<nama-file>.env` | Template database connection config (default `config/db-connection.env`) |

Folder `config/` dibuat otomatis bila belum ada. Pada mode non-interaktif, nama file selalu `db-connection.env`.

## Contoh

```bat
:: Mode interaktif (prompt LICENSE, database, nama file config)
npx restforge init

:: Mode non-interaktif: tulis template default, timpa bila sudah ada
npx restforge init --force
```

## Catatan

- Command bersifat interaktif hanya pada terminal (TTY) dan tanpa `--force`. Pada lingkungan non-interaktif, gunakan `--force` atau edit file template hasil generate secara manual.
- Flag `--force` bersifat destruktif: file config yang sudah ada akan tertimpa tanpa konfirmasi.
- Pada mode interaktif, bila file target sudah ada, review menampilkan penanda overwrite dan konfirmasi `Y/n` berfungsi sebagai persetujuan penimpaan (tidak memerlukan `--force`).
- Orchestrator `fast-track` memanggil `init --force` secara internal untuk menyiapkan file config sebelum mengisi nilai dari dialognya sendiri.

---

**Lihat juga**: [`commands/`](./) · [`fast-track`](./fast-track.md) · [`conventions`](./conventions.md) · [`README`](../README.md)
