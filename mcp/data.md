# Domain `data_*`

> 2 tool untuk mengekspor dan mengimpor baris data tabel dalam bentuk file envelope JSON,
> sepenuhnya dikendalikan metadata SDF sehingga hasilnya tidak terikat dialect database.

Pasangan ini dipakai berurutan saat data dipindahkan antar database: `data_pull` di sisi sumber,
`data_push` di sisi tujuan. Nama file di kedua sisi identik, sehingga hasil pull dapat langsung
di-push tanpa penyesuaian.

Kedua tool wajib menerima `cwd` dan mensyaratkan `@restforgejs/platform` terpasang lokal serta
license yang valid. Keduanya juga selalu mengirim flag `--json` agar ringkasan hasilnya
terstruktur; tidak ada mode output lain.

**Scope wajib tepat satu.** Isi salah satu dari `table`, `schema`, atau `allSchemas`.
Mengosongkan ketiganya, atau mengisi lebih dari satu, terhitung kesalahan pemakaian.

| Tool | Verb CLI | Parameter | Catatan perilaku |
|---|---|---|---|
| `data_pull` | `data pull (--table \| --schema \| --all-schemas) --json` | **`cwd`**, `table`, `schema`, `allSchemas`, `config`, `schemaPath`, `limit`, `batchSize`, `storagePath`, `force` | Mengekspor baris ke file envelope JSON di folder `data-storage/` relatif terhadap `cwd`, kecuali `storagePath` diisi. Tabel yang punya schema ditulis bersarang sebagai `data-storage/<schema>/<table>.json`, tabel tanpa schema ditulis rata sebagai `data-storage/<table>.json`. Hanya tabel yang terdaftar di SDF yang bisa di-pull. `limit` membatasi jumlah baris per tabel, `batchSize` mengatur ukuran batch pembacaan, dan `force` menimpa file envelope yang sudah ada. Tool tidak melakukan pre-check terhadap SDF maupun config; kegagalan dilaporkan langsung dari CLI. |
| `data_push` | `data push (--table \| --schema \| --all-schemas) --json` | **`cwd`**, `table`, `schema`, `allSchemas`, `config`, `schemaPath`, `storagePath`, `batchSize` | Memuat baris dari file envelope ke database tujuan lewat batch INSERT. Sifatnya **append-only**: tidak ada upsert maupun replace, dan tidak ada opsi force atau overwrite. Menjalankannya dua kali menyisipkan data dua kali, jadi konfirmasi kepada pengguna diperlukan sebelum push ke database yang mungkin sudah berisi baris tersebut. Untuk scope `schema` dan `allSchemas`, tabel dimuat mengikuti urutan topologi foreign key sehingga tabel induk lebih dulu. |
