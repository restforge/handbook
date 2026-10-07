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
| `rbac-users.html` | Daftar user, tampilan lihat user, serta tambah dan ubah user beserta role-nya |
| `rbac-roles.html` | Daftar role dan detail role |
| `rbac-role-permissions.html` | Matriks permission per role |
| `js/rbac-users.js`, `js/rbac-roles.js`, `js/rbac-role-permissions.js` | Logika masing-masing halaman |
| `js/permission-grouping.js` | Pengelompokan permission untuk matriks |

File yang sudah ada dilewati. Dengan `--overwrite`, file lama diarsipkan lalu ditimpa.

### Halaman Users

Halaman `rbac-users.html` menampilkan daftar user beserta role yang dipegang setiap user. Kolom Roles berisi nama role, bukan kode role, dengan satu badge per role yang diurutkan menurut nama. User tanpa role menampilkan `-`.

Isi satu user bisa dilihat tanpa membuka form Edit. Klik nama user di kolom User, atau pilih `View` di menu Actions. Keduanya membuka modal View User yang menampilkan setiap field sebagai pasangan label dan nilai teks, mulai dari Username sampai Last Login. Field yang kosong bertuliskan `Not provided`, sedangkan password tidak pernah ditampilkan. Modal ditutup lewat tombol `Close`, ikon silang, atau tombol Escape.

Checkbox Roles pada modal Add User dan Edit User juga hanya memuat nama role, misalnya `Staff Gudang`.

Kolom Roles membaca nama role dari endpoint `user` layanan auth. Layanan auth yang dipasang sebelum fitur ini menampilkan `-` di kolom tersebut sampai definisi endpoint `user` diperbarui. Jalankan [`init --force`](../restforge-backend/auth-service/init.md#menjalankan-ulang-dengan---force) dengan flag yang sama seperti pemasangan awal, lalu buat ulang endpoint tersebut:

```bat
npx restforge endpoint create --project=auth-service --name=user --payload=auth_user.json --config=auth.env --database=postgres --force
```

Halaman Users yang dibuat sebelum fitur ini belum punya modal View User. Jalankan `rbac --create --overwrite` agar halaman memakai versi baru.

### Role Sistem dan Role Tenant

Setiap owner memegang satu tenant, yaitu kelompok user dan role yang terpisah dari kelompok lain di aplikasi yang sama. Pendaftar dari [registrasi publik](../restforge-backend/auth-service/README.md#registrasi-publik) otomatis menjadi owner tenant barunya, dan editor RBAC menyesuaikan tampilan dengan tenant itu.

- Role bawaan sistem seperti `OWNER` tampil berbadge System di halaman Roles. Bagi owner, role ini read-only: tidak bisa diubah, tidak bisa dihapus, dan permission-nya tidak bisa diubah. Menu di barisnya hanya berisi View dan Assign Permissions, dan matriks permission tampil tanpa checkbox yang bisa diubah.
- Role yang dibuat owner, misalnya `STAFF`, berbadge Custom dan hanya terlihat di tenant pembuatnya.
- Halaman Users hanya menampilkan user dari tenant owner yang login.
- Pilihan role di modal Add User dan Edit User memuat `OWNER` dan role milik tenant itu.

`SUPER_ADMIN` tetap bisa mengubah role sistem.

Halaman RBAC yang dibuat sebelum fitur ini perlu dibuat ulang dengan `rbac --create --overwrite`, lalu `generate`, agar memakai versi baru.

### Matriks Assign Permissions

Halaman `rbac-role-permissions.html` menampilkan satu baris per resource dan satu kolom per aksi. Description resource tampil di samping kode resource dengan warna dan ukuran huruf yang sama. Resource tanpa description membiarkan area itu kosong.

Description aksi tampil sebagai tooltip checkbox aksi itu. Aksi tanpa description memakai nama permission, misalnya `Create Item`.

Kedua description ditulis di manifest permission dan disimpan lewat [`auth-service provision`](../restforge-backend/auth-service/provision.md#description-resource-dan-aksi). Cara mengisinya ada di [`auth-service manifest`](../restforge-backend/auth-service/manifest.md#description-resource).

Halaman yang dibuat sebelum fitur ini tidak menampilkan description resource. Jalankan `rbac --create --overwrite` agar halaman memakai versi baru.

### Grup Administration

Grup yang ditambahkan ke UDF memuat dua item `link` dengan key `roles`:

```json
{
  "type": "group",
  "label": "Administration",
  "icon": "shield-tick",
  "children": [
    { "type": "link", "label": "Users", "href": "rbac-users.html", "roles": ["OWNER", "SUPER_ADMIN"] },
    { "type": "link", "label": "Roles", "href": "rbac-roles.html", "roles": ["OWNER", "SUPER_ADMIN"] }
  ]
}
```

Menu Users dan Roles hanya tampil untuk user yang memegang salah satu role di `roles`. Kedua role itu adalah role bawaan sistem. Role yang dibuat sendiri di halaman Roles tidak membuka menu ini, termasuk role yang diberi kode `ADMIN`. Hal yang sama berlaku di backend, sehingga hanya OWNER dan SUPER_ADMIN yang bisa mengelola user, role, dan permission. Grup yang seluruh isinya tersembunyi ikut tersembunyi. Detail key `roles` ada di [`catalogs/udf/navigation.md`](../../catalogs/udf/navigation.md#properti-roles-pada-item-link).

Bila navigasi sudah memuat item dengan `href` `rbac-users.html` di tingkat mana pun, grup tidak ditambahkan lagi. Item Users dan Roles yang masih memakai `roles` bawaan versi lama, yaitu `["OWNER", "ADMIN", "SUPER_ADMIN"]`, diganti menjadi `["OWNER", "SUPER_ADMIN"]`. Nilai `roles` lain dianggap pilihan pengguna dan tidak diubah. Selain itu, UDF tidak berubah. Bagian lain UDF, termasuk indentasi dan akhir baris, tetap seperti semula.

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
╭── RestForge Designer RBAC ───────────────────╮
│ RBAC editor created                          │
│ Project    : myapp                           │
│ Target dir : frontend/apps/myapp             │
│ UDF        : frontend/payload/myapp.json     │
│ Written    : rbac-users.html                 │
│              rbac-roles.html                 │
│              rbac-role-permissions.html      │
│              js/permission-grouping.js       │
│              js/rbac-users.js                │
│              js/rbac-roles.js                │
│              js/rbac-role-permissions.js     │
│ Skipped    : (none)                          │
│ Archived   : (none)                          │
│ Navigation : Administration group added      │
╰──────────────────────────────────────────────╯

Next: npx restforge-designer generate --payload=frontend/payload/myapp.json --project=myapp --frontend-path=frontend/apps --overwrite
```

Setiap file ditulis satu per baris. Perintah lanjutan dicetak di bawah kotak sebagai satu baris utuh, sehingga dapat langsung di-copy.

Menjalankan perintah yang sama untuk kedua kalinya memindahkan ketujuh file ke `Skipped` dan menampilkan `Navigation : unchanged ('rbac-users.html' is already in the menu)`. Dengan `--overwrite`, baris pertama `Archived` berisi folder arsip, lalu diikuti nama file yang diarsipkan.

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
