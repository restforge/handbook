# `schema init`

> Membuat skeleton file schema definition dari template `dummy` di koleksi RestForge Schema Reference. Merupakan thin wrapper terhadap [`schema template`](./template.md) dengan parameter `--table=dummy --generate --lang=sdf` yang sudah preset.

## Pattern

```
npx restforge schema init --path=<PATH>
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--path <PATH>` | Ya | - | Path file schema yang akan dibuat (mis. `schema/users.js`) |

## Contoh (Examples)

```bat
npx restforge schema init --path=schema/users.js
npx restforge schema init --path=./schema/item_product.js
```

## Setara dengan (Equivalent To)

Perintah ini adalah shortcut yang setara dengan pemanggilan eksplisit verb `schema template`:

```bat
npx restforge schema template --table=dummy --generate --lang=sdf --path=<PATH>
```

Untuk memilih template dari catalog selain `dummy` (misalnya `customer`, `sales_order`, `employee`), gunakan langsung [`schema template`](./template.md) dengan flag `--table=<nama>`.

## Catatan Penting (Important Notes)

- File output berisi struktur template `dummy` (master-data, single-table, generic domain). Rename field dan tableName sesuai kebutuhan setelah generate
- Saat ini binary `sdf-tools.exe` hanya tersedia untuk Windows. Pemanggilan di Linux atau macOS akan return error eksplisit
- File destination yang sudah ada akan menyebabkan error (gunakan [`schema template --force`](./template.md) jika perlu overwrite)

---

**Lihat juga**: [`schema template`](./template.md) · [`schema/`](./) · [`commands/`](../) · [`README`](../../../README.md)
