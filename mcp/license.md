# Domain `license_*`

> 1 tool untuk membaca keadaan aktivasi license pada mesin saat ini.

| Tool | Verb CLI | Parameter | Catatan perilaku |
|---|---|---|---|
| `license_info` | `license info` | **`cwd`** | Read-only. Menampilkan license key, e-mail pemilik, tipe license, machine id, waktu pemeriksaan terakhir, dan masa berlaku. Verb ini terdiri dari dua token tanpa flag dan ditangani parser runtime, bukan CLI generator, sehingga flag tambahan diabaikan diam-diam alih-alih ditolak. Keluarannya hanya blok teks untuk dibaca manusia; tidak ada mode terstruktur, dan teks itu diteruskan apa adanya. Ketersediaan `@restforgejs/platform` diperiksa lebih dulu dan ketiadaannya dilaporkan sebagai precondition. |

Isi output memuat data sensitif. License key, e-mail, dan machine id sebaiknya hanya diulang
kepada pengguna bila memang diminta.

Pemakaian yang wajar adalah mendiagnosis kegagalan license yang dilaporkan
`setup_validate_config` sebelum aktivasi ulang disarankan. Untuk mengubah nilai license di file
config, jalurnya adalah `setup_update_env`, bukan domain ini.

## `license deactivate` Tidak Di-wrap

Pelepasan aktivasi tidak tersedia sebagai tool. Efeknya melampaui folder project karena seat
yang dipegang mesin ini dibebaskan secara terpusat di license server, dan tindakan itu tidak
dapat dibatalkan dari sisi MCP. Perintahnya diserahkan kepada pengguna untuk dijalankan sendiri
di terminal:

```bash
npx restforge license deactivate
```

Urutan yang disarankan adalah memanggil `license_info` lebih dulu agar terlihat apa yang sedang
aktif, baru kemudian pengguna menjalankan deaktivasi.
