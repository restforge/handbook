# Domain `key_*`

> 3 tool untuk mengelola API key project yang tersimpan di file `.env`.

Ketiganya membungkus resource `key` pada CLI backend dan mensyaratkan
`@restforgejs/platform` terpasang lokal di `cwd`. Domain ini berdiri sendiri: ia tidak menjadi
prasyarat maupun kelanjutan dari pipeline definition file.

Nilai API key bersifat rahasia. Nilai penuh hanya muncul bila diminta secara eksplisit, dan
sebaiknya tidak diulang kembali ke dalam percakapan.

| Tool | Verb CLI | Parameter | Catatan perilaku |
|---|---|---|---|
| `key_generate` | `key generate [--output] [--force]` | **`cwd`**, `output`, `force` | Membuat API key baru dan menuliskannya ke file `.env`. Bila file target sudah memuat key dan `force` tidak diisi, CLI meminta konfirmasi penimpaan; prompt itu tidak dapat dijawab dari MCP dan panggilan berpotensi menggantung sampai timeout. Karena itu `force` harus diisi saat key existing memang ingin ditimpa. Menulis ke file yang belum punya key berjalan normal tanpa `force`. Nilai key hasil generate ikut muncul di output tool. |
| `key_list` | `key list [--show-full] [--dir]` | **`cwd`**, `showFull`, `dir` | Mendaftar API key yang terdaftar di file `.env`. Nilai key ditutup secara default; `showFull` membuka nilai penuh dan hanya wajar dipakai atas permintaan eksplisit pengguna. `dir` mengarahkan pencarian ke folder lain. |
| `key_revoke` | `key revoke --file=<file> --yes` | **`cwd`**, **`file`** | Destruktif: key dihapus dari file `.env` yang disebut. `file` wajib karena tanpa argumen itu CLI membuka pemilih file interaktif yang tidak bisa dijawab dari MCP. Flag `--yes` di-hardcode sehingga pencabutan berlangsung seketika tanpa konfirmasi CLI, dan konfirmasi kepada pengguna harus dilakukan sebelum tool dipanggil. |
