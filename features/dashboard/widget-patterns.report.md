# Laporan Review — widget-patterns.md

Folder: `restforge-handbook/features/dashboard/`
Indikator: TEMUAN (6 temuan; dikelompokkan)

## Temuan

1. **Heading bilingual (Deteksi F) — 5 lokasi: baris 1, 7, 14, 24, 292**

   Baris 1   : `# Pola Widget (Widget Patterns)` → `# Pola Widget`

   Baris 7   : `## Aturan Pemetaan Payload ke Response (Payload-to-Response Mapping Rules)` → `## Aturan Pemetaan Payload ke Response`

   Baris 14  : `### Aturan Scalar Collapse (Scalar Collapse Rules)` → `### Aturan Scalar Collapse`

   Baris 24  : `### Contoh Lengkap (Complete Example)` → `### Contoh Lengkap`

   Baris 292 : `## Rekomendasi Nama Kolom SQL (Recommended SQL Column Names)` → `## Rekomendasi Nama Kolom SQL`

   Alasan : Bagian Inggris dalam kurung adalah terjemahan murni; bukan singkatan/istilah teknis bagian dari nama. Heading "Pola 1/2/3 (Scalar + Object + ...)" sengaja TIDAK dimasukkan karena bagian kurung mendeskripsikan shape, bukan terjemahan.

2. **Baris 197 — Kategori 3 (Bagian C glossary)**

   Asli   : > ****Good to know:**** `generate_series` menjamin jumlah baris konsisten meskipun ada hari tanpa transaksi.

   Usulan : > ****Perlu diketahui:**** `generate_series` menjamin jumlah baris konsisten meskipun ada hari tanpa transaksi.

   Alasan : Label "Good to know" terdaftar di Bagian C glossary dengan padanan "Perlu diketahui".

## Ringkasan
- Total temuan: 6 (dikelompokkan menjadi 2 nomor)
- Sudah direvisi: belum, menunggu approve
