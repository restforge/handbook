# `key revoke`

> Revoke (cabut) API key dari file `.env`.

## Pattern

```
npx restforge key revoke [options]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--file <PATH>` | Tidak | interactive | Path file `.env` yang berisi key untuk dicabut |
| `--yes` | Tidak | `false` | Lewati prompt konfirmasi |

## Contoh (Examples)

```bat
npx restforge key revoke --file=.env --yes
```

---

**Lihat juga**: [`key/`](./) · [`commands/`](../) · [`README`](../../../README.md)
