# `launcher create`

> Membuat satu script di root project untuk menjalankan dan menghentikan semua proses aplikasi lewat terminal atau pm2.

## Pattern

```
npx restforge launcher create --project=<NAME> [options]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--project <NAME>` | Ya | | Nama project backend yang terdaftar di `.restforge/projects.json` |
| `--runner <RUNNER>` | Tidak | `terminal` | Cara proses dijalankan, yaitu `terminal` atau `pm2` |
| `--os <OS>` | Tidak | OS tempat perintah dijalankan | `windows` menulis `.bat`, `linux` menulis `.sh`, dan `both` menulis keduanya |
| `--config <FILE>` | Tidak | Default config di `.restforge/defaults.json` | File env backend di folder `config/` |
| `--frontend-port <N>` | Tidak | `8000` | Port aplikasi frontend (range 1-65535) |
| `--force` | Tidak | `false` | Menimpa launcher dan `ecosystem.config.js` yang sudah ada |

## Proses yang Dijalankan

Launcher (script kendali) menjalankan proses yang ditemukan di project. Backend selalu masuk. Auth-service masuk bila `.restforge/projects.json` memuat project `auth-service` dan file `config/auth.env` ada. Frontend masuk bila folder `frontend/apps/<project>/` ada.

| Proses | Perintah | Port |
|--------|----------|------|
| auth-service | `serve --project=auth-service --config=auth.env` | `SERVER_PORT` di `config/auth.env` |
| backend | `serve --project=<project> --config=<file>` | `SERVER_PORT` di file `--config` |
| frontend | `frontend serve --dir=frontend/apps/<project>` | `--frontend-port` |

Perintah berhenti dengan error bila project belum terdaftar, file config tidak ditemukan, atau `SERVER_PORT` kosong.

## File yang Ditulis

Setiap runner menulis launcher dengan nama sendiri, sehingga launcher terminal dan launcher pm2 bisa ada bersamaan di root project.

| Runner | File |
|--------|------|
| `terminal` | `launch-terminal.bat` atau `launch-terminal.sh` |
| `pm2` | `launch-pm2.bat` atau `launch-pm2.sh`, ditambah `ecosystem.config.js` |

Perintah menolak menimpa file yang sudah ada dan menyebut nama file tersebut. Tambahkan `--force` untuk menulis ulang launcher, misalnya setelah port atau file config berubah.

## Aksi Launcher

Kedua launcher menerima aksi yang sama sebagai argumen pertama. Tanpa argumen, launcher menjalankan aksi `start`.

| Aksi | Runner `terminal` | Runner `pm2` |
|------|-------------------|--------------|
| `start` | Membuka satu window atau window tmux untuk setiap proses | Menjalankan `pm2 startOrRestart ecosystem.config.js`, lalu `pm2 save` |
| `stop` | Menutup window milik project | Menjalankan `pm2 stop ecosystem.config.js` |
| `restart` | Menjalankan `stop` lalu `start` | Sama dengan `start` |
| `status` | Menampilkan proses yang masih berjalan | Menjalankan `pm2 status` |
| `logs` | Masuk ke session tmux di Linux, tidak tersedia di Windows | Menjalankan `pm2 logs` |

Setelah `start`, launcher menampilkan URL setiap proses. Aksi yang tidak dikenal membuat launcher berhenti dan menampilkan daftar aksi yang tersedia. Sebelum `start`, launcher memeriksa bahwa folder `node_modules` ada, dan launcher pm2 juga memeriksa bahwa perintah `pm2` bisa dipanggil.

### Runner terminal di Windows

Setiap proses berjalan di window console sendiri dengan judul tetap, misalnya `myapp - backend`, `myapp - auth-service`, dan `myapp - frontend`. Aksi `stop` hanya menutup window dengan judul tersebut beserta proses di dalamnya, sehingga window terminal lain tetap terbuka. Menjalankan `start` saat proses masih berjalan tidak membuka window kedua.

### Runner terminal di Linux

Semua proses berjalan di satu session tmux bernama sesuai project, dengan satu window tmux untuk setiap proses. Aksi `logs` membuka session itu, dan session ditinggalkan tanpa menghentikan proses dengan Ctrl+B lalu D.

Runner ini hanya didukung di Ubuntu 22.04 ke atas. Di distro atau versi lain, launcher berhenti dengan pesan yang meminta launcher dibuat ulang dengan `--runner=pm2`. Bila tmux belum terpasang, launcher berhenti dengan pesan yang meminta pemasangan lewat `sudo apt install tmux`.

### Runner pm2

`ecosystem.config.js` mendaftarkan proses dengan nama `<project>-server`, `<project>-frontend`, dan `auth-service`. Setiap proses memanggil `node_modules/@restforgejs/platform/server.js` langsung, sehingga file yang sama berjalan di Windows dan Linux. Frontend dijalankan dengan `--server-address=0.0.0.0` agar bisa dibuka dari komputer lain di jaringan.

Watch pm2 tidak dipakai di semua proses, sehingga perubahan source code baru berlaku setelah `restart`. Aksi `stop` hanya menghentikan proses yang terdaftar di `ecosystem.config.js`, sedangkan proses pm2 lain tetap berjalan.

## Contoh

```bat
:: Launcher terminal untuk OS saat ini
npx restforge launcher create --project=myapp

:: Launcher pm2 untuk Windows
npx restforge launcher create --project=myapp --runner=pm2

:: Launcher pm2 untuk server Linux, dibuat dari komputer Windows
npx restforge launcher create --project=myapp --runner=pm2 --os=linux
```

Setelah file ditulis, perintah menampilkan output berikut.

```
Launcher created for project 'myapp' (runner: terminal)
  launch-terminal.bat

Processes:
  auth-service  http://localhost:3100/api/auth-service
  backend       http://localhost:3000/api/myapp
  frontend      http://localhost:8000/index.html

Usage: launch-terminal.bat [start|stop|restart|status|logs]
```

Launcher lalu dijalankan dari root project.

```bat
launch-terminal.bat start
launch-terminal.bat status
launch-terminal.bat stop
```

Di Linux, launcher dijalankan dengan `./launch-terminal.sh start` atau `./launch-pm2.sh start`.

---

**Lihat juga**: [`launcher/`](./) · [`frontend serve`](../frontend/serve.md) · [`fast-track`](../fast-track.md) · [`commands/`](../) · [`README`](../../../README.md)
