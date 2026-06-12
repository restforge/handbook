# `config set-default`

> Tetapkan file config sebagai default untuk working directory saat ini. Disimpan di `.restforge/defaults.json`.

## Pattern

```
npx restforge config set-default --config=<FILE>
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--config <FILE>` | Ya | - | File config yang dijadikan default |

## Contoh (Examples)

```bat
npx restforge config set-default --config=config/db-connection.env
```

---

**Lihat juga**: [`config/`](./) · [`commands/`](../) · [`README`](../../../README.md)
