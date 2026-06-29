# `project delete`

> Menghapus project dari registry termasuk seluruh endpoint, processor, dashboard, dan consumer yang berada di dalamnya.

## Pattern

```
npx restforge project delete --project=<NAME> [--yes]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--project <NAME>` | Ya | - | Nama project yang akan dihapus |
| `--yes` | Tidak | `false` | Lewati prompt konfirmasi |

## Contoh

```bat
npx restforge project delete --project=my-app
npx restforge project delete --project=my-app --yes
```

---

**Lihat juga**: [`project/`](./) · [`commands/`](../) · [`README`](../../../README.md)
