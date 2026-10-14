# Resource `payload`

> Resource untuk mengelola file payload JSON yang menjadi sumber generate endpoint. Terdiri dari lima sub-verb yang berbagi parameter `--config`.

## Sub-command (5)

| Verb | Fungsi |
|------|--------|
| [`generate`](./generate.md) | Generate payload JSON dari schema database |
| [`validate`](./validate.md) | Validasi payload terhadap struktur tabel aktual |
| [`diff`](./diff.md) | Tampilkan perbedaan kolom antara payload dan database |
| [`sync`](./sync.md) | Apply perubahan schema dari database ke file payload |
| [`migrate`](./migrate.md) | Konversi payload backend (RDF) menjadi payload frontend (UDF) untuk Designer |

---

**Lihat juga**: [`commands/`](../) · [`README`](../../../README.md)
