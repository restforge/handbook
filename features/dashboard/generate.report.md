# Laporan Review — generate.md

Folder: `restforge-handbook/features/dashboard/`
Indikator: TEMUAN (13 temuan; dikelompokkan)

## Temuan

1. **Heading bilingual (Deteksi F) — 10 lokasi: baris 5, 44, 46, 52, 83, 102, 155, 223, 243, 272**

   Baris 5   : `## Membuat Endpoint Dashboard (Creating a Dashboard Endpoint)` → `## Membuat Endpoint Dashboard`

   Baris 44  : `## Menyiapkan Project (Project Setup)` → `## Menyiapkan Project`

   Baris 46  : `### Prasyarat (Prerequisites)` → `### Prasyarat`

   Baris 52  : `### Struktur Folder (Folder Structure)` → `### Struktur Folder`

   Baris 83  : `## Output File Generated (Generated File Output)` → `## Output File Generated`

   Baris 102 : `## Dukungan Multi-Dialect (Multi-Dialect Support)` → `## Dukungan Multi-Dialect`

   Baris 155 : `## Praktik Terbaik (Best Practices)` → `## Praktik Terbaik`

   Baris 223 : `### Mengupdate SQL Widget (Regenerate Workflow)` → `### Mengupdate SQL Widget`

   Baris 243 : `## Konvensi Tipe Response Cross-Dialect (Cross-Dialect Response Type Convention)` → `## Konvensi Tipe Response Cross-Dialect`

   Baris 272 : `## Keterbatasan (Limitations)` → `## Keterbatasan`

   Alasan : Bagian Inggris dalam kurung adalah terjemahan murni; bukan singkatan/istilah teknis bagian dari nama.

2. **Baris 26 — Kategori 3 (Bagian C glossary)**

   Asli   : > ****Good to know:**** Nilai `--name` menjadi filename `{name}.js` dan URL segment.

   Usulan : > ****Perlu diketahui:**** Nilai `--name` menjadi filename `{name}.js` dan URL segment.

   Alasan : Label "Good to know" terdaftar di Bagian C glossary dengan padanan "Perlu diketahui".

3. **Baris 48 — Kategori 5 (Deteksi B: campur kode)**

   Asli   : Project sudah **di-initialize** via `npx restforge init` sehingga mempunyai struktur folder `payload/`, `src/`, `metadata/`, `config/`.

   Usulan : Project sudah **diinisialisasi** via `npx restforge init` sehingga mempunyai struktur folder `payload/`, `src/`, `metadata/`, `config/`.

   Alasan : "initialize" tidak masuk glossary A/B; padanan baku Indonesia "diinisialisasi" sudah lazim di dokumentasi teknis.

4. **Baris 290 — Kategori 5 (Deteksi B: campur kode)**

   Asli   : File `{name}.js` gagal load saat startup sehingga route tidak **ter-register** dan request jatuh ke catch-all 403 handler.

   Usulan : File `{name}.js` gagal load saat startup sehingga route tidak **terdaftar** dan request jatuh ke catch-all 403 handler.

   Alasan : "register" tidak masuk glossary A/B dan memiliki padanan baku "terdaftar".

## Ringkasan
- Total temuan: 13 (dikelompokkan menjadi 4 nomor)
- Sudah direvisi: belum, menunggu approve
