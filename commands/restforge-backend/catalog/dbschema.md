# `catalog dbschema`

> Output JSON catalog dbschema-kit defineModel API spec (field types, constraints, relations, shorthand syntax, naming rules). Mendukung filter section/name/kind.

## Pattern

```
npx restforge catalog dbschema [options]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--section <NAME>` | Tidak | semua | Filter berdasarkan section (`defineModelOptions`, `fieldTypes`, `constraints`, dll.) |
| `--name <VALUE>` | Tidak | semua | Filter berdasarkan nama exact |
| `--kind <VALUE>` | Tidak | semua | Filter constraint berdasarkan kind (`standalone`, `value`) |
| `--pretty <BOOL>` | Tidak | `true` | Pretty-print output JSON |

## Contoh

```bat
npx restforge catalog dbschema
npx restforge catalog dbschema --section=fieldTypes
npx restforge catalog dbschema --section=constraints --kind=standalone
```

---

**Lihat juga**: [`catalog/`](./) · [`commands/`](../) · [`README`](../../../README.md)
