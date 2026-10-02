# `npx restforge-designer rbac`

> Membuat editor RBAC (halaman Users, Roles, dan matriks permission per role) untuk aplikasi frontend yang memakai layanan [`auth-service`](../restforge-backend/auth-service/README.md), lalu menambahkan grup Administration ke navigasi UDF.

## `--create`

### Sintaks

```bash
npx restforge-designer rbac --create --project=<NAME> [OPTIONS]
```

### Flag

| Flag | Tipe | Wajib | Default | Keterangan |
|------|------|:-----:|---------|-----------|
| `--create` | flag | ✓ | `false` | Trigger pembuatan editor RBAC |
| `--project <NAME>` | string | ✓ | - | Nama project frontend. Folder aplikasi adalah `<frontend-path>/<project>` dan file UDF adalah `<payload-path>/<project>.json` |
| `--frontend-path <PATH>` | string | ✗ | `frontend/apps` | Root folder frontend apps |
| `--payload-path <PATH>` | string | ✗ | `frontend/payload` | Folder UDF |
| `--overwrite` | flag | ✗ | `false` | Timpa halaman RBAC yang sudah ada. File lama diarsipkan ke `.restforge/archive/` |

### Prasyarat

- UDF `<payload-path>/<project>.json` sudah ada dan memakai plugin `vanilla-js-auth`. Plugin dibaca dari `appConfig.plugin`, termasuk bila nilainya berasal dari file yang dirujuk `extends`. Plugin lain ditolak dengan error dan tidak ada file yang ditulis.
- UDF dibuat dengan [`payload migrate`](../restforge-backend/payload/migrate.md) memakai `--plugin=vanilla-js-auth`, `--auth-app-code`, dan `--auth-api-url`.
- Layanan auth sudah disiapkan dengan perintah [`auth-service`](../restforge-backend/auth-service/README.md).

Bila UDF tidak memiliki blok `auth`, perintah tetap berjalan dan menampilkan peringatan. Halaman RBAC membutuhkan `appCode` dan alamat layanan auth dari blok itu, yang baru ditulis `generate` saat blok `auth` ada.

### Apa yang Dikerjakan

1. Memeriksa plugin dan bentuk `navigation` sebelum menulis file apa pun.
2. Menulis tiga halaman dan empat file JS ke `<frontend-path>/<project>/`.
3. Menambahkan grup Administration ke `navigation.items` di UDF.

File yang ditulis:

| File | Fungsi |
|------|--------|
| `rbac-users.html` | Daftar user, tambah dan ubah user beserta role-nya |
| `rbac-roles.html` | Daftar role dan detail role |
| `rbac-role-permissions.html` | Matriks permission per role |
| `js/rbac-users.js`, `js/rbac-roles.js`, `js/rbac-role-permissions.js` | Logika masing-masing halaman |
| `js/permission-grouping.js` | Pengelompokan permission untuk matriks |

File yang sudah ada dilewati. Dengan `--overwrite`, file lama diarsipkan lalu ditimpa.

### Grup Administration

Grup yang ditambahkan ke UDF memuat dua item `link` dengan key `roles`:

```json
{
  "type": "group",
  "label": "Administration",
  "icon": "shield-tick",
  "children": [
    { "type": "link", "label": "Users", "href": "rbac-users.html", "roles": ["OWNER", "ADMIN", "SUPER_ADMIN"] },
    { "type": "link", "label": "Roles", "href": "rbac-roles.html", "roles": ["OWNER", "ADMIN", "SUPER_ADMIN"] }
  ]
}
```

Menu Users dan Roles hanya tampil untuk user yang memegang salah satu role di `roles`. Grup yang seluruh isinya tersembunyi ikut tersembunyi. Detail key `roles` ada di [`catalogs/udf/navigation.md`](../../catalogs/udf/navigation.md#properti-roles-pada-item-link).

Bila navigasi sudah memuat item dengan `href` `rbac-users.html` di tingkat mana pun, grup tidak ditambahkan lagi dan UDF tidak berubah. Bagian lain UDF, termasuk indentasi dan akhir baris, tetap seperti semula.

### Posisi di Urutan

`rbac --create` dijalankan sesudah `payload migrate` dan sebelum `generate`. Halaman RBAC ikut tampil di sidebar karena grup Administration sudah ada di UDF saat `generate` berjalan.

`generate --overwrite` tidak menghapus halaman RBAC. Ketujuh file tetap ada dengan isi yang sama, sedangkan `sidebar.html` dibuat ulang dengan grup Administration.

### Contoh

```bat
cd /d D:\projects\myapp
npx restforge-designer rbac --create --project=myapp
npx restforge-designer rbac --create --project=myapp --frontend-path=frontend/apps
npx restforge-designer rbac --create --project=myapp --overwrite
```

### Output

```
RBAC editor created
Project    : myapp
Target dir : frontend/apps/myapp
UDF        : frontend/payload/myapp.json
Written    : rbac-users.html, rbac-roles.html, rbac-role-permissions.html, js/permission-grouping.js, js/rbac-users.js, js/rbac-roles.js, js/rbac-role-permissions.js
Skipped    : (none)
Archived   : (none)
Navigation : Administration group added

Next: npx restforge-designer generate --payload=frontend/payload/myapp.json --project=myapp --frontend-path=frontend/apps --overwrite
```

Menjalankan perintah yang sama untuk kedua kalinya melewati ketujuh file dan menampilkan `Navigation : unchanged`.

### Error Umum

| Pesan | Penyebab |
|-------|----------|
| `--create is required.` | Flag `--create` tidak diberikan |
| `UDF not found` | File `<payload-path>/<project>.json` tidak ada. Jalankan `payload migrate` lebih dulu |
| `the RBAC editor requires the 'vanilla-js-auth' plugin` | UDF memakai plugin lain. Jalankan `payload migrate` dengan `--plugin=vanilla-js-auth` |
| `cannot update navigation` | Bentuk `navigation` di UDF tidak valid |

---

## Catatan

Halaman RBAC memanggil layanan auth langsung dari browser memakai token login. Hak akses tetap ditegakkan layanan auth, sedangkan penyembunyian menu hanya mengatur tampilan.

---

**Lihat juga**: [`restforge-frontend/`](./) · [`auth-service`](../restforge-backend/auth-service/README.md) · [`generate.md`](./generate.md) · [`commands/`](../) · [`README`](../../README.md)
