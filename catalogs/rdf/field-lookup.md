# Field Lookup (`fieldNameLookup`)

Mengonfigurasi kolom mana yang digunakan sebagai `id` dan `text` di response endpoint `/lookup`. Bersifat opsional.

```json
{
    "fieldNameLookup": {
        "id": "category_id",
        "text": "category_code||' - '||category_name as display_text"
    }
}
```

| Property | Tipe | Wajib | Keterangan |
|----------|------|-------|-----------|
| `id` | string | Ya (jika `fieldNameLookup` ditetapkan) | Kolom yang dipakai sebagai `id` di response |
| `text` | string | Ya (jika `fieldNameLookup` ditetapkan) | Kolom atau SQL expression untuk `text` di response. Mendukung concatenation native dialect (PostgreSQL pakai `||`, MySQL pakai `CONCAT()`) |

| Aturan | Penjelasan |
|--------|-----------|
| Tanpa konfigurasi | Auto-detect berdasarkan pola nama (`name`, `code`, `title`, `text`, `label`, `tag`) |
| Search behavior | Jika `fieldNameLookup` ada, search dilakukan pada kolom yang diekstrak dari `text`. Jika tidak, search lintas semua kolom teks auto-detected |
| Concatenation | Generator menerjemahkan operator `||` ke `CONCAT()` di MySQL secara otomatis |

Detail lengkap (auto-detect heuristic, format `{ id, text }`, integrasi dengan `viewQuery`) dibahas di dokumentasi fitur lookup terpisah.

---

**Lihat juga**: [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
