# Resource `runtime`

> Command yang berhubungan dengan eksekusi server runtime serta manajemen license.

Berbeda dengan resource CLI generator lain, command pada `runtime` dipanggil langsung dengan verb tanpa prefix `runtime` (mis. `npx restforge serve`, bukan `npx restforge runtime serve`). Folder ini berfungsi sebagai pengelompokan dokumentasi semata.

## Sub-command (4)

| Verb | Fungsi |
|------|--------|
| [`serve`](./serve.md) | Menjalankan runtime HTTP server untuk project RESTForge yang sudah di-generate |
| [`validate`](./validate.md) | Memvalidasi file config database tanpa menjalankan server |
| [`license-info`](./license-info.md) | Menampilkan informasi license yang aktif pada mesin saat ini |
| [`license-deactivate`](./license-deactivate.md) | Melepas aktivasi license dari mesin saat ini |

---

**Lihat juga**: [`commands/`](../) · [`README`](../../../README.md)
