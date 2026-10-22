# `frontend serve`

> Melayani folder aplikasi frontend sebagai static file server sampai proses dihentikan dengan Ctrl+C.

## Pattern

```
npx restforge frontend serve [options]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--dir <PATH>` | Tidak | `.` | Folder aplikasi frontend yang dilayani |
| `--port <N>` | Tidak | `8080` | Port HTTP (range 1-65535) |
| `--server-address <HOST>` | Tidak | `127.0.0.1` | Alamat bind server. Isi `0.0.0.0` agar aplikasi bisa dibuka dari komputer lain di jaringan |

## Perilaku

Request ke `/` dilayani dengan `index.html`, sedangkan file lain dilayani sesuai path-nya. File `.html` dilayani apa adanya tanpa redirect ke URL tanpa ekstensi, sehingga `/login.html` tetap terbuka sebagai `login.html` dan `/login` menghasilkan 404. Dotfile seperti `.env` tidak dilayani.

Command berhenti dengan error bila folder tidak ditemukan atau port sudah dipakai proses lain.

## Contoh

```bat
:: Dari folder aplikasi frontend
cd frontend\apps\my-project
npx restforge frontend serve --port=8080

:: Dari root project
npx restforge frontend serve --dir=frontend/apps/my-project --port=8080
```

Setelah server siap, command menampilkan output berikut.

```
Serving D:\projects\my-app\frontend\apps\my-project
  Local: http://127.0.0.1:8080/index.html
  Press Ctrl+C to stop.
```

---

**Lihat juga**: [`frontend/`](./) · [`fast-track`](../fast-track.md) · [`commands/`](../) · [`README`](../../../README.md)
