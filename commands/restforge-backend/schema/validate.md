# `schema validate`

> Memvalidasi seluruh file schema definition di sebuah folder. Cek mencakup single-model validation dan cross-model consistency.

## Pattern

```
npx restforge schema validate --schema-path=<PATH>
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--schema-path <PATH>` | Ya | - | Path file atau folder schema (mis. `./schema` atau `./schema/users.js`) |

## Contoh

```bat
npx restforge schema validate --schema-path=./schema
npx restforge schema validate --schema-path=./my-schema
```

---

**Lihat juga**: [`schema/`](./) · [`commands/`](../) · [`README`](../../../README.md)
