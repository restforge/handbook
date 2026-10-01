# Domain `health_*`

> 1 tool untuk memastikan transport MCP hidup dan responsif.

| Tool | Verb CLI | Parameter | Catatan perilaku |
|---|---|---|---|
| `health_ping` | Tidak memanggil CLI (berjalan in-process) | `message` | Satu-satunya tool tanpa parameter `cwd`. Tidak menyentuh filesystem, jaringan, maupun komponen RESTForge mana pun, sehingga tidak memerlukan `@restforgejs/platform` maupun license. Balasannya berupa `pong` disertai timestamp ISO 8601 dan versi server, ditambah `message` yang dikembalikan apa adanya bila diisi. |

Tool ini menjawab pertanyaan "apakah MCP server-nya nyala", bukan pertanyaan tentang keadaan
project. Untuk memeriksa license dan koneksi, jalurnya adalah `setup_validate_config`; untuk
membaca config aktif, `setup_read_env`.
