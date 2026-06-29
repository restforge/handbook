# RESTForge — Quickstart

Dokumen ini menjelaskan bagaimana cara cepat dalam membuat project menggunakan RESTForge, inisialisasi project, membangun koneksi ke database serta contoh aplikasi sederhana yang bisa langsung di jalankan mulai dari backend sampai frontend.

---

## Kebutuhan Awal

Sebelum memulai siapkan beberapa tools yang diperlukan seperti dibawah ini.

| Item | Keterangan |
| --- | --- |
| Node.js LTS | Periksa ketersediaan NodeJS melalui terminal (`node -v`) |
| Database PostgreSQL | Pastikan database postgreSQL telah tersedia |
| Database Alternatif | Database SQLite bisa menjadi alternatif apabila database postgreSQL belum tersedia |



## Langkah 1 — Install

Sebelum mulai membuat project, pastikan bahwa nodejs sudah terinstal, jika belum ikuti cara instalasi di website resmi nodejs.

```bash
node --version
v22.21.1

```

Nama project yang digunakan dalam materi ini adalah  `myapp`, untuk membuat folder project baru gunakan `create-restforge-app`

```cmd
npx create-restforge-app@latest myapp --yes
```

Setelah proses instalasi selesai, masuk ke folder `myapp`

```bash
cd myapp
```

Periksa versi software yang terinstall
```bash
npx restforge --version
```

Hasil yang diharapkan seperti dibawah ini

```bash
<root-project>/myapp
> npx restforge --version
RESTForge Platform v5.2.13
Node.js v22.21.1
Platform: win32 (x64)
```



Designer kini tersedia lintas-platform via `npx restforge-designer` setelah project dibuat, karena binary di-bundle dalam package `@restforgejs/platform`. Tidak perlu download manual untuk sistem operasi apa pun. Lakukan verifikasi dengan perintah berikut.

```bash
npx restforge-designer --version

<root-project>/myapp
> npx restforge-designer --version
restforge-designer 1.2.7
```



## Langkah 2 — Inisialisasi project dan membuat koneksi ke database

Lakukan inisialisasi project menggunakan perintah `init`:

```cmd
npx restforge init
```

RESTForge akan menyediakan dialog interaktif dan harus di isi. Pastikan memiliki LICENSE KEY  serta kredensial yang valid untuk koneksi ke database. 
```bash
D:\projects\myapp
> npx restforge init

RESTForge - Init
====================

Enter configuration (press Enter to accept the default value):

  LICENSE (XXXX-XXXX-XXXX-XXXX): 8ECD-XXXX-XXXX-698Q

  Supported DB_TYPE: postgresql, mysql, oracle, sqlite
  DB_TYPE (postgresql): postgresql
  DB_HOST (127.0.0.1):
  DB_PORT (5432):
  DB_USER (postgres): postgres
  DB_PASSWORD (your_password_here): postgres1234
  DB_NAME (your_database_name): dbapp

  Config file name (db-connection.env): db-connection.env

====================================================
  Review configuration
====================================================
  License   : 8ECD-****-****-698Q
  Database  : postgresql @ 127.0.0.1:5432/dbapp
  Target    : config/db-connection.env

  Write this config? (Y/n): y

  Created: config\db-connection.env

Done! Sample files generated successfully.
```



Lakukan validasi untuk membuktikan bahwa RESTForge dapat berjalan dengan baik sesuai yang telah di persyaratkan.

```bash
npx restforge validate --config=db-connection.env
```

Hasil output yang diharapkan sebagai berikut.

```bash
D:\projects\myapp
> npx restforge validate --config=db-connection.env

==========================================
  RESTFORGE VALIDATE
==========================================

Config file:  db-connection.env

[1] Database Connection
    Type:     postgresql
    Host:     127.0.0.1
    Port:     5432
    User:     postgres
    Database: dbapp
    Status:   [FAIL] database "dbapp" does not exist

Database 'dbapp' was not found on the postgresql server.
Do you want to create a new database? (y/N):
```
> **Perlu diketahui:** Jika database belum tersedia, misal `dbapp`, selanjutnya pilih `Y` apabila ingin membuat database baru dan `N` jika ingin dilewati.

Jika database telah berhasil di buat, dan sudah tersedia maka lakukan validasi ulang.

```bash
D:\projects\myapp
> npx restforge validate --config=db-connection.env

==========================================
  RESTFORGE VALIDATE
==========================================

Config file:  db-connection.env

[1] Database Connection
    Type:     postgresql
    Host:     127.0.0.1
    Port:     5432
    User:     postgres
    Database: dbapp
    Status:   [OK] Connection successful

[2] Redis Connection
    Status:   [SKIP] No Redis-dependent feature enabled

[3] Kafka Connection
    Status:   [SKIP] KAFKA_ENABLED is not set to true

[4] Configuration Preflight
    [OK]      SERVER_ADDRESS: 127.0.0.1
    [OK]      SERVER_PORT: 3000

[5] License Validation
    Source:   env LICENSE
    Key:      8ECD-XXXX-XXXX-698Q
    Email:    dbxp.dev1@gmail.com
    Type:     trial
    Expires:  2026-07-19 03:31:27 (31 days remaining)
    Mode:     Online
    Status:   [OK] License is valid

==========================================
  VALIDATION PASSED
==========================================
```

**PASTIKAN** proses validasi menghasilkan nilai **PASSED**



## Langkah 3 — Membuat table Guest Book sederhana

Silahkan melakukan pengaturan config default mengarah kepada file `db-connection.env` agar setiap perintah yang membutuhkan parameter flag `--config` dapat mengambil nilai default yang telah ditetapkan.
```bash
npx restforge config set-default --config=db-connection.env
```

Selanjutnya secara cepat buat sebuah contoh table, gunakan template table yang telah tersedia, dan jalankan perintah `--generate` menjadi file SDF.
```bash
npx restforge schema template --table=guest_book --generate --schema-path=./schema
```

Perintah diatas menghasilkan struktur file SDF sesuai template seperti dibawah ini:
```bash
module.exports = ({ defineModel }) => defineModel('guest_book', {
  fields: {
    guest_book_id:    'string:36 pk',
    guest_book_name:  'string:100 notnull',
    email:            'string:255',
    phone:            'string:20',
    company_name:     'string:255',
    purpose_of_visit: 'string:255',
    notes:            'text',
    visit_date:       'timestamp notnull default:now()',
    status:           "string:20 notnull default:'waiting'",
    created_at:       'timestamp default:now()',
    created_by:       'string:100',
    updated_at:       'timestamp',
    updated_by:       'string:100'
  },
  indexes: [
    'guest_book_name'
  ],
  checks: [
    { field: 'status', in: ['waiting', 'called', 'done'] }
  ]
});
```



## Langkah 4 — Jalankan mode fast-track

Gunakan perintah `fast-track` untuk membangun REST API dan aplikasi frontend secara cepat, berdasarkan file SDF yang telah tersimpan dalam folder `schema`.

```bash
npx restforge fast-track --project=myapp --schema-path=./schema
```


Ketika file db-connection.env telah di setup menggunakan perintah `init` dan sudah di validasi, maka file tersebut langsung dapat digunakan. Edit file db-connection.env apabil diperlukan adanya perubahan.

```bash
================================================================
  RESTForge Fast-Track  —  Existing Configuration
================================================================

  Project    : myapp
  Schema     : ./schema
  Config     : db-connection.env
  License    : 8ECD-****-****-698Q
  REST API   : 127.0.0.1:3000
  Web server : 127.0.0.1:8000
  Mode       : sync

  Database Configuration
  Type       : postgresql
  Host       : 127.0.0.1
  Port       : 5432
  User       : postgres
  Password   : postgres1234
  DB Name    : dbapp



    Edit Configuration
  > Continue
```


Pilih **Contine** dan lanjutkan pilih REST API dan Frontend untuk membuat aplikasi lengkap backend dan frontend.

```bash
  Select what to generate:

    Generate REST API Only
  > Generate REST API with Frontend Application
```

Pilih plugin Vanilla JS Basic.
```bash
  Select a designer plugin:

    vanilla-js-custom  (auth: No)
  > vanilla-js-basic  (auth: No)
    vanilla-js-auth  (auth: No)

```

Apabila sudah sesuai lanjutkan `Continue = Y`
```bash
================================================================
  RESTForge Fast-Track
================================================================

  Project    : myapp
  Generate   : REST API + Frontend app
  Schema     : ./schema   (1 SDF files)
  Config     : db-connection.env   (will be written)
  License    : 8ECD-****-****-698Q
  REST API   : 127.0.0.1:3000   (SERVER_PORT)
  Web server : 127.0.0.1:8000
  Database   : postgresql @ 127.0.0.1:5432/dbapp
  Plugin     : vanilla-js-basic
  Mode       : sync   (use --overwrite to drop & regenerate)

  Continue? (Y/n): y
```

Tunggu proses berjalan sampai pada dialog konfirmasi berikut ini, dan pilih Y
```bash
================================================================
  DONE  —  REST API generated + frontend app generated   [scope: REST API + Frontend app]
================================================================
  Backend  : payload/ , examples/myapp/  (1 endpoint)
  Start    : server-start.bat  (start REST API manually)
  Frontend : frontend/apps/myapp/  (start: npx serve . -l 8000)
================================================================

  Run Runtime Server and frontend application now in a new window? (Y/n):
```


Proses berakhir sampai pada dialog informasi seperti dibawah ini
```bash
  Opening new window: "RESTForge Server - myapp"
#npx restforge serve --project=myapp --config=db-connection.env --watch
  ✓ Server window opened. Keep it open. Stop with Ctrl+C.

  Waiting for server health: http://127.0.0.1:3000/api/myapp/health
  ✓ Server healthy (3.1s).

  Opening new window: "RESTForge Frontend - myapp"
#npx serve . -l 8000
  ✓ Frontend window opened (WEB_SERVER_PORT 8000).
    Open: http://localhost:8000/index.html
```

Terminal akan menampilkan dua Service :

```
✓ Backend  API  : http://localhost:3000/api/myapp/guest-book
✓ Frontend App  : http://localhost:8000
```

Buka browser dan akses ke **http://localhost:8000**

Hasilnya adalah aplikasi CRUD **Guest Book** yang berfungsi penuh: tabel data dengan sorting dan pagination, form tambah/edit, tombol hapus, serta pencarian, seluruhnya tersambung ke Database melalui REST API. 

Tambahkan satu-dua data sebagai uji coba, dan data tersebut langsung tersimpan di tabel `guest_book`.

**PASTIKAN Bahwa data tersimpan dengan benar**



## Langkah 5 — Memahami Proses yang Terjadi

Setelah hasilnya terlihat, model mentalnya menjadi mudah dipahami. Ketiga file definisi sebelumnya memetakan ke tiga fase:

| File | Fase | Menghasilkan |
| --- | --- | --- |
| `schema/guest_book.js` (SDF) | Database | Tabel di Database |
| `payload/guest-book.json` (RDF) | Backend | Endpoint REST API |
| `frontend/payload/pages/guest-book.json` (UDF) | Frontend | Halaman aplikasi |

Perhatikan bahwa ada tiga file dan tiga fase dalam satu alur, inilah metode yang diperkenalkan sebagai **definition-first** yaitu pengguna mendeklarasikan *apa* yang diinginkan, dan RESTForge yang akan menuliskan kode program sesuai kebutuhan.

---

## Selanjutnya

- **Tutorial membangun sendiri aplikasi dari nol** → [Playbook Skenario](https://github.com/restforge/playbook) memandu penulisan SDF/RDF/UDF secara mandiri, penambahan relasi JOIN, dan pembuatan aplikasi multi-page.
- **Referensi lengkap** → [Handbook](https://github.com/restforge/handbook) untuk spec SDF, RDF, UDF, dan CLI.

---

## Bila Terjadi Kegagalan

Hanya ada satu titik yang bergantung pada lingkungan setempat, yaitu koneksi database. Hampir semua masalah berasal dari sini.

| Pesan error | Artinya | Solusi |
| --- | --- | --- |
| `ECONNREFUSED` | Database tidak terjangkau | Pastikan Database berjalan; periksa `DB_HOST` dan `DB_PORT` |
| `password authentication failed` | Kredensial salah | Periksa `DB_USER` dan `DB_PASSWORD` di `db-connection.env` |
| `port 3000/8000 already in use` | Port terpakai | Hentikan proses lain, atau ubah menjadi port  yang lain |
