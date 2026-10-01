# `project sdk`

> Menghasilkan SDK JavaScript untuk satu project — satu layer tipis di atas REST API hasil generate sehingga frontend cukup memanggil `client.<resource>.<verb>(payload)` tanpa menulis ulang boilerplate `fetch`/`$.ajax`.

## Pattern

```
npx restforge project sdk --generate --project=<NAME> [--sdk-path=<PATH>] [--base-url=<URL>] [--force]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--project <NAME>` | Ya | - | Nama project target (juga jadi nama package SDK) |
| `--generate` | Ya | - | Trigger eksekusi generate SDK source |
| `--sdk-path <PATH>` | Tidak | `<project-root>/sdk` | Folder output SDK source |
| `--base-url <URL>` | Tidak | derive dari config | Base URL API yang di-bake ke `sdk-client.js` |
| `--force` | Tidak | `false` | Timpa SDK source yang sudah ada |

`--base-url` default diturunkan dari default config project (`SERVER_ADDRESS` + `SERVER_PORT` + `/api/<project>`; `0.0.0.0`/kosong dipetakan ke `127.0.0.1`). Bila tidak ada config, fallback ke `http://127.0.0.1:3000/api/<project>`.

## Yang dihasilkan

Command hanya **menulis source** (tidak build). Resource diturunkan dari `metadata/<project>.json` (hanya endpoint `type: module`); `primaryKey` diambil dari `payload/<slug>.json`. Bila payload sebuah endpoint terdaftar tidak ditemukan, seluruh generate dibatalkan.

```
<sdk-path>/
├── package.json          # scripts: build = "tsup", deploy = "node deploy.mjs"
├── tsup.config.js        # build ESM + CJS + IIFE
├── deploy.mjs            # copy hasil build ke folder js/ app frontend
├── sdk-client.js         # bootstrap browser (baseUrl ter-bake)
├── README.md             # overview, resource+methods, field, konvensi request/response, snippet
└── src/
    ├── core/             # http-client.js, resource-client.js (generik, tidak berubah antar project)
    ├── resources/        # 1 file per resource
    └── index.js          # createClient()
```

## Prosedur Build & deploy

```bat
cd <project-root>\sdk
npm install
npm run build
npm run deploy
```

`npm run build` menghasilkan `dist/` (ESM `index.js`, CJS `index.cjs`, IIFE `index.global.js`). `npm run deploy` (atau `node deploy.mjs <app>/js`) menyalin hasil build ke `<app>/js/sdk/` dan `sdk-client.js` ke `<app>/js/sdk-client.js`. Untuk app berbasis bundler, pakai `npm install file:<sdk-path>` lalu `import { createClient } from '<NAME>'`.

## Contoh

```bat
npx restforge project sdk --generate --project=myapp
npx restforge project sdk --generate --project=myapp --force
npx restforge project sdk --generate --project=myapp --sdk-path=./client-sdk
npx restforge project sdk --generate --project=myapp --base-url=https://api.example.com/api/myapp
```

## Auth

Bila project sudah dipasang auth extension (`project auth`, terdeteksi dari `src/modules/<project>/rfx_auth.js`), generator otomatis menyertakan `client.auth` (login/register/refresh/logout/getMe) plus `core/auth-client.js` dan `core/storage.js`. Endpoint auth memakai baseUrl yang sama: `{baseUrl}/rfx_auth/*`. Token otomatis dilampirkan (Bearer) ke setiap pemanggilan resource dan di-refresh saat 401.

```js
await client.auth.login('username', 'password');
const data = await client.guestBook.datatables({ start: 0, length: 10 }); // token otomatis
client.auth.logout();
```

Tanpa auth extension, `createClient` hanya menerima `baseUrl` dan `client.auth` tidak ada.

## Catatan

- Adapter `react`/`vue` belum tersedia di versi awal.
- Jalankan ulang dengan `--force` setiap kali daftar endpoint backend atau status auth berubah, lalu build + deploy ulang.

---

**Lihat juga**: [`project/`](./) · [`endpoint create`](../endpoint/create.md) · [`commands/`](../) · [`README`](../../../README.md)
