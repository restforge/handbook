# `key generate`

> Generate API key baru dan menulisnya ke file `.env`.

## Pattern

```
npx restforge key generate [options]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--output <FILE>` | Tidak | `.env` | File output untuk menulis API key |
| `--force` | Tidak | `false` | Timpa KEY existing tanpa konfirmasi |

## Contoh (Examples)

```bat
npx restforge key generate
npx restforge key generate --output=config/production.env --force
```

---

**Lihat juga**: [`key/`](./) · [`commands/`](../) · [`README`](../../../README.md)
