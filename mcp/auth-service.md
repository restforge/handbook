# Domain `auth_service_*`

> 5 tool untuk memasang dan mengisi auth-service, yaitu layanan auth berbasis role dan permission (RBAC) untuk aplikasi frontend dengan plugin `vanilla-js-auth`.

Kelima tool membungkus verb `npx restforge auth-service <verb>` dan dijalankan dari root project aplikasi. Auth-service hanya mendukung PostgreSQL. Tabelnya berada di schema `auth` pada database aplikasi, dan layanannya berjalan sebagai project `auth-service` dengan file env sendiri (`config/auth.env`). Mekanisme ini berbeda dari `project_auth`, yang hanya menyediakan login tanpa role, dan keduanya tidak dipakai bersamaan pada satu aplikasi.

Seluruh tool mensyaratkan `@restforgejs/platform` terpasang lokal di `cwd`. Bila package belum ada, tool menjawabnya sebagai precondition, bukan kegagalan. Rincian flag tiap verb ada di [`commands/restforge-backend/auth-service/`](../commands/restforge-backend/auth-service/README.md).

Kolom "parameter" menandai parameter wajib dengan **tebal**. Parameter `cwd` wajib pada seluruh tool dan tidak diulang di kolom tersebut.

## Alur Baku

Urutan berikut mengikuti runbook Auth Service. Langkah di luar domain ini memakai tool dari domain lain, kecuali `npm install` yang dijalankan agent lewat shell.

```
auth_service_init → npm install → auth_service_bootstrap
  → runtime_generate_launcher (name "auth-service") → auth_service_link
  → codegen_generate_payload → codegen_create_endpoint
  → auth_service_provision → codegen_migrate_payload (authAppCode, authApiUrl)
  → designer_rbac_create → designer_generate
```

`npm install` diperlukan karena `init` mencatat paket `ejs` dan `nodemailer` ke `package.json` project. `link` dijalankan sebelum endpoint pertama dibuat. Setelah itu `codegen_generate_payload` memasang `authGuard` ke RDF baru, dan `codegen_create_endpoint` menambahkan resource ke `config/auth-manifest.json`, sehingga `auth_service_manifest` tidak termasuk alur baku. Tabel baru cukup melewati kedua tool itu, lalu `auth_service_provision` dijalankan sekali tanpa `ownerEmail`.

## Daftar Tool

| Tool | Verb CLI | Parameter | Catatan perilaku |
|---|---|---|---|
| `auth_service_init` | `auth-service init` | `appConfig` (default `db-connection.env`), `config` (default `auth.env`), `port` (default `3100`), `jwtAlgorithm` (`HS256` atau `RS256`), `force` | Menyalin file layanan auth ke project dan membuat `config/auth.env` beserta `JWT_SECRET` acak. Tool ini tidak menyentuh database. `RS256` membuat pasangan kunci di `config/keys/`, sedangkan `force` menimpa file lama setelah mengarsipkannya |
| `auth_service_bootstrap` | `auth-service bootstrap` | `config` (default `auth.env`), `reset` | Membuat schema `auth`, tabel, data awal, dan modul layanan auth. Password `superadmin` hanya ditampilkan sekali. Tanpa `reset`, perintah berhenti bila tabel `auth.app` sudah berisi. Dengan `reset: true` tool menjadi destruktif karena tabel di schema `auth` dihapus dan dibuat ulang beserta isinya, sehingga konfirmasi kepada pengguna harus dilakukan lebih dulu |
| `auth_service_link` | `auth-service link --project=<project> --app-code=<appCode>` | **`project`**, **`appCode`**, `config` (default `db-connection.env`), `authConfig` (default `auth.env`) | Menulis app code dan kunci verifikasi ke file env aplikasi, lalu memasang `authGuard` ke RDF aplikasi. `appCode` hanya berisi huruf kapital, angka, dan underscore. Perintah ini idempoten, bisa dijalankan sebelum project punya endpoint, mempertahankan `enabled: false` pada blok yang sudah ada, dan menolak project yang sudah memakai `project auth` |
| `auth_service_manifest` | `auth-service manifest --project=<project>` | **`project`**, `output` (default `config/auth-manifest.json`) | Menyusun ulang manifest permission secara manual dari RDF yang memakai `authGuard`, satu resource per endpoint. `codegen_create_endpoint` sudah memperbarui manifest untuk project yang terhubung. Isian yang ditambahkan pengguna, seperti `category`, `sortOrder`, `description`, dan action di luar CRUD, dipertahankan saat manifest disusun ulang |
| `auth_service_provision` | `auth-service provision --manifest=<manifest> [--owner-email=<ownerEmail>]` | **`manifest`**, `ownerEmail` (wajib selama user owner belum ada), `authConfig` (default `auth.env`), `ownerUsername`, `ownerPassword`, `dryRun`, `allowSelfRegister` | Mendaftarkan app, permission, role `OWNER`, dan user owner, lalu mengirim email notifikasi ke owner baru lewat SMTP di `auth.env`. `dryRun` hanya menampilkan rencana dan sebaiknya dijalankan lebih dulu. Tanpa `ownerPassword`, password acak dibuat, ditampilkan sekali, dan disimpan di `data-storage/credential/<app-code>-owner.txt`. `allowSelfRegister` membuka registrasi publik; `provision` tanpa parameter ini menutupnya kembali |

## Tool Terkait di Domain Lain

Beberapa langkah alur auth-service memakai tool dari domain lain dengan parameter khusus.

| Tool | Domain | Peran dalam alur auth-service |
|---|---|---|
| `runtime_generate_launcher` | [`runtime_*`](./runtime.md) | Launcher layanan auth ditulis dengan `name: "auth-service"`, project `auth-service`, config `auth.env`, dan port `3100`, sehingga tidak bertabrakan dengan launcher aplikasi di folder yang sama |
| `codegen_migrate_payload` | [`codegen_*`](./codegen.md) | `authAppCode` dan `authApiUrl` menulis blok `auth` ke UDF aggregator, dengan plugin `vanilla-js-auth` |
| `designer_rbac_create` | [`designer_*`](./designer.md) | Membuat halaman editor RBAC dan grup menu Administration setelah UDF dimigrasi, sebelum `designer_generate` |
| `project_tenant` | [`project_*`](./project.md) | Memisahkan data bisnis per tenant, dipakai bila registrasi publik dibuka dan setiap pendaftar memiliki tenant sendiri |
