# Troubleshooting License Validation

> Pemecahan masalah untuk error yang muncul saat validasi license (`validate`, `serve`, `license info`).

Kembali ke [ikhtisar Runtime](README.md).

---

## "License server authenticity check failed"

Pesan ini muncul saat `validate` (atau `serve`) gagal memverifikasi keaslian response dari license server. Pesan sengaja dibuat umum, tidak menampilkan detail internal, sebagai bagian dari desain keamanan — sehingga penyebab pastinya perlu ditelusuri lewat tabel berikut, bukan dari teks error itu sendiri.

| Penyebab | Langkah |
|----------|---------|
| Jam sistem tidak sinkron (Windows Time service tidak berjalan, atau belum pernah sync ke NTP) | Jalankan `net start w32time`, lalu `w32tm /config /manualpeerlist:"time.windows.com" /syncfromflags:manual /reliable:YES /update`, lalu `w32tm /resync /rediscover`. Semua command butuh Command Prompt sebagai Administrator |
| Versi `@restforgejs/platform` sudah usang | Update ke versi terbaru: `npm install @restforgejs/platform@latest`, lalu ulangi validasi |
| Proxy atau antivirus melakukan SSL inspection yang mengubah response HTTPS | Whitelist domain `restforge.dev` pada proxy/antivirus, atau nonaktifkan sementara SSL inspection untuk domain tersebut |

Verifikasi jam sistem sudah benar setelah perbaikan:

```cmd
:: Command Prompt
echo %date% %time%

:: PowerShell
Get-Date
```

Lalu cek status sinkronisasi:

```cmd
w32tm /query /status
```

`Leap Indicator` harus menunjukkan `0 (no warning)` dan `Source` mengarah ke NTP server (mis. `time.windows.com`), bukan lagi `Local CMOS Clock`.

---

**Lihat juga**: [`runtime/`](./) · [`runtime/validate`](validate.md) · [`runtime/license-info`](license-info.md) · [`commands/`](../) · [`README`](../../../README.md)
