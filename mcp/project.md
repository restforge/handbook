# Domain `project_*`

> 4 tool untuk operasi tingkat project: melihat daftar, menghapus, memasang ekstensi auth
> backend, dan menghasilkan SDK client JavaScript.

Keempatnya membungkus resource `project` pada CLI backend dan mensyaratkan
`@restforgejs/platform` terpasang lokal di `cwd`.

Kolom "parameter" menandai parameter wajib dengan **tebal**. Parameter `cwd` wajib pada seluruh
tool dan tidak diulang di kolom tersebut.

| Tool | Verb CLI | Parameter | Catatan perilaku |
|---|---|---|---|
| `project_list` | `project list` | (hanya `cwd`) | Read-only. Verb ini tidak menerima flag lain. Berguna untuk orientasi, tetapi bukan prasyarat langkah mana pun. |
| `project_delete` | `project delete --project=<project> --yes` | **`project`** | Destruktif. Flag `--yes` di-hardcode agar prompt konfirmasi CLI dilewati, karena prompt itu tidak bisa dijawab dari MCP, sehingga penghapusan berlangsung seketika. Konfirmasi kepada pengguna harus dilakukan sebelum tool dipanggil. |
| `project_auth` | `project auth --create --project=<project>` | **`project`**, `schemaPath`, `config`, `force` | Memasang ekstensi auth backend pada project yang sudah ada. Flag `--create` selalu dikirim karena CLI menjadikannya pemicu wajib; flag opsional hanya diteruskan saat parameternya diisi. Rangkaian efeknya: file SDF auth berprefix `rfx` ditulis ke `schemaPath`, tabel auth dibuat di database secara idempoten, middleware dan router auth ditulis, enam processor (register, login, refresh, logout, me, reset-password) dihasilkan, variabel env auth beserta `JWT_SECRET` acak disuntikkan ke file config, lalu `bcrypt` dan `jsonwebtoken` dicatat sebagai dependency runtime. Ini adalah auth sisi backend; UI login frontend ditangani `designer_auth_create`. |
| `project_sdk_generate` | `project sdk --generate --project=<project>` | **`project`**, `sdkPath`, `baseUrl`, `force` | Menghasilkan source SDK JavaScript untuk satu project, yaitu lapisan client tipis di atas REST API hasil generate sehingga frontend memanggil `client.<resource>.<verb>(payload)`. Flag `--generate` selalu dikirim karena tanpa flag itu CLI keluar dengan exit code 2. SDK diturunkan dari `metadata/<project>.json` dan file payload tiap resource; bila ada endpoint terdaftar yang tidak punya file payload, generate dibatalkan sebelum satu file pun ditulis. Ketika ekstensi auth backend terdeteksi, SDK juga memperoleh `client.auth` dan token bearer otomatis terpasang pada setiap panggilan resource, sehingga `project_auth` sebaiknya dijalankan lebih dulu bila project memang butuh auth. Default `sdkPath` adalah `<root project>/sdk`. Tool hanya menulis source: `npm install`, build, dan deploy tetap dijalankan pengguna. Regenerate setelah daftar endpoint atau status auth berubah membutuhkan `force=true`, yang menimpa folder SDK di tempat tanpa backup, jadi hal itu perlu disampaikan lebih dulu. |
