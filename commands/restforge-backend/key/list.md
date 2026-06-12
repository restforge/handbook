# `key list`

> Menampilkan daftar API key yang terdaftar (default dengan tampilan masked).

## Pattern

```
npx restforge key list [options]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--show-full` | Tidak | `false` | Tampilkan key full tanpa masking |
| `--dir <PATH>` | Tidak | cwd | Directory pencarian file `.env` |

## Contoh (Examples)

```bat
npx restforge key list
npx restforge key list --show-full --dir=./config
```

---

**Lihat juga**: [`key/`](./) · [`commands/`](../) · [`README`](../../../README.md)
