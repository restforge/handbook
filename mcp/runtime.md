# Domain `runtime_*`

> 7 tool untuk mengenali project dan config, memeriksa kesiapan sebelum runtime dinyalakan,
> membaca status server, dan menulis file launcher.

Domain ini memegang batas paling tegas dalam surface MCP: **tidak ada tool yang menjalankan
runtime server maupun Kafka consumer**. Proses yang dinyalakan dari sesi AI menjadi anak proses
sesi tersebut dan ikut mati saat sesi ditutup, sehingga kepemilikan proses harus tetap di
terminal pengguna. Yang dikerjakan tool adalah menyiapkan file; eksekusinya milik pengguna.

Alur baku:

```
runtime_detect_project → runtime_detect_config → runtime_validate_preflight
  → runtime_check_launcher_exists → runtime_generate_launcher → pengguna menjalankan skrip
```

`runtime_check_status` berada di luar urutan itu dan boleh dipanggil kapan saja karena hanya
membaca keadaan. Menghentikan atau me-restart server yang sedang berjalan juga diserahkan
kepada pengguna dalam bentuk perintah konkret, bukan dijalankan lewat tool.

Kolom "parameter" menandai parameter wajib dengan **tebal**. Parameter `cwd` wajib pada seluruh
tool dan tidak diulang di kolom tersebut.

## Deteksi dan Kesiapan

| Tool | Verb CLI | Parameter | Catatan perilaku |
|---|---|---|---|
| `runtime_detect_project` | Operasi file langsung | (hanya `cwd`) | Membaca daftar file di `src/modules/`; nama project adalah nama file tanpa ekstensi `.js`. Folder yang belum ada dijawab sebagai precondition. Bila hanya satu project ditemukan, alur boleh lanjut tanpa bertanya; bila lebih dari satu, pilihan diserahkan kepada pengguna. |
| `runtime_detect_config` | Operasi file langsung | (hanya `cwd`) | Membaca isi folder `config/`. Nama file inilah yang kemudian diteruskan sebagai `--config=<filename>`, dan runtime me-resolve-nya relatif terhadap folder `config/`. |
| `runtime_validate_preflight` | `restforge validate --config=<config>`, ditambah pemeriksaan file | `config` (default `db-connection.env`), `port` | Menggabungkan validasi config (license, database, Redis dan Kafka bila diaktifkan) dengan pemeriksaan file PID serta ketersediaan port bila `port` diisi. Kegagalan preflight bersifat informatif, bukan penghalang: launcher tetap boleh dibuat dengan catatan peringatan. Pemeriksaan port bersifat best-effort, karena port yang bebas di mesin ini belum tentu bebas pada alamat bind server yang sebenarnya. |
| `runtime_check_status` | Operasi file langsung, `pm2 jlist`, dan probe HTTP | `mode` (default `auto`), `project`, `port`, `host_address` (default `127.0.0.1`), `health_path`, `timeout_ms` (default `3000`) | Memeriksa berurutan: file `.restforge/server.pid` (launcher lama), percobaan bind ke port yang dikonfigurasi (bind gagal berarti port terpakai dan server mode host dianggap berjalan), daftar proses PM2 lewat `pm2 jlist`, lalu probe endpoint health bila `health_path` diisi. Seluruhnya read-only dan tidak mengubah siklus hidup proses. |

## Penulisan Launcher

| Tool | Verb CLI | Parameter | Catatan perilaku |
|---|---|---|---|
| `runtime_check_launcher_exists` | Operasi file langsung | **`os`**, **`mode`** | Memeriksa apakah file launcher yang akan dihasilkan `runtime_generate_launcher` sudah ada, sehingga potensi penimpaan dapat dikonfirmasikan lebih dulu. Nama file bersifat tetap dan tidak dapat dikustomisasi: `server-start.bat` dan `server-stop.bat` untuk `os=windows`, `server-start.sh` dan `server-stop.sh` untuk `os=linux`, ditambah `ecosystem.config.js` pada `mode=pm2`. |
| `runtime_generate_launcher` | Operasi file langsung | **`os`**, **`mode`**, **`project`**, **`config`**, **`port`**, `overwrite` (default `false`), `cluster`, `workers`, `watch` | Menulis skrip launcher ke root project dan berhenti di situ; server tidak dijalankan. `mode=pm2` mensyaratkan PM2 sudah terpasang di mesin pengguna, dan tool tidak memasangkannya; bila PM2 tidak ada, kegagalan baru muncul saat skrip dijalankan. Tanpa `overwrite`, file yang sudah ada tidak ditimpa. |
| `runtime_generate_consumer_launcher` | `mode=host`: operasi file langsung. `mode=pm2`: `restforge-consumer-deploy --project=<p> --config=<c>` | **`os`**, **`mode`**, **`project`**, **`config`**, `consumer`, `port`, `output`, `overwrite` (default `false`) | Menyiapkan cara menjalankan Kafka consumer, dengan pola dua langkah yang sama. Consumer adalah binary terpisah dari API server, sehingga `runtime_generate_launcher` dan `runtime_check_status` tidak mencakupnya. `config` wajib di sini walau runtime server sendiri bisa mencari config-nya, karena binary consumer menolak jalan tanpa `--config` dan hanya menerima file berekstensi `.env`. Pada `mode=host` tool menulis pasangan `consumer-start` dan `consumer-stop` (`.bat` untuk windows, `.sh` untuk linux) dengan nama tetap, sehingga dua consumer di satu project akan saling menimpa nama file bila `overwrite` diizinkan. Pada `mode=pm2` penulisan didelegasikan ke `restforge-consumer-deploy`, yang menghasilkan `ecosystem.config.js` dan `consumer-manager.sh` di folder `deploy/`; parameter `os` diabaikan pada mode ini tetapi tetap diminta agar simetris dengan dua tool launcher lainnya. Flag `--license` dan `--license-server` sengaja tidak diekspos. |
