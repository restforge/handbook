# `schema validate`

> Memvalidasi seluruh file schema definition di sebuah folder. Cek mencakup single-model validation dan cross-model consistency.

## Pattern

```
npx restforge schema validate --path=<PATH>
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--path <PATH>` | Ya | - | Path file atau folder schema (mis. `./schema` atau `./schema/users.js`) |

## Contoh (Examples)

```bat
npx restforge schema validate --path=./schema
npx restforge schema validate --path=./my-schema
```

---

**Lihat juga**: [`schema/`](./) · [`commands/`](../) · [`README`](../../../README.md)
