# Laporan Review — payload.md

Folder: `restforge-handbook/features/dashboard/`
Indikator: TEMUAN (13 temuan; dikelompokkan)

## Temuan

1. **Heading bilingual (Deteksi F) — 8 lokasi: baris 7, 42, 58, 88, 103, 125, 139, 153**

   Baris 7   : `## Struktur Dasar (Basic Structure)` → `## Struktur Dasar`

   Baris 42  : `## Referensi Field Payload (Payload Field Reference)` → `## Referensi Field Payload`

   Baris 58  : `## Mendefinisikan Parameter Request (Defining Request Parameters)` → `## Mendefinisikan Parameter Request`

   Baris 88  : `## Field Terlarang (Forbidden Fields)` → `## Field Terlarang`

   Baris 103 : `## Mengaktifkan Cache (Enabling Cache)` → `## Mengaktifkan Cache`

   Baris 125 : `### Cara Cache Invalidation Bekerja (How Cache Invalidation Works)` → `### Cara Cache Invalidation Bekerja`

   Baris 139 : `### Validasi Cross-Reference Cache (Cache Cross-Reference Validation)` → `### Validasi Cross-Reference Cache`

   Baris 153 : `## Validasi Payload (Payload Validation)` → `## Validasi Payload`

   Alasan : Bagian Inggris dalam kurung adalah terjemahan murni; bukan singkatan/istilah teknis bagian dari nama.

2. **Label "Good to know" (Bagian C glossary) — 2 lokasi: baris 82, 123**

   Asli (baris 82)  : > ****Good to know:**** Runtime mensubstitusi `:name` ke bind syntax dialect target secara otomatis — `$1`, `$2` untuk PostgreSQL; `?` untuk MySQL; `:1`, `:2` untuk Oracle.

   Usulan (baris 82): > ****Perlu diketahui:**** Runtime mensubstitusi `:name` ke bind syntax dialect target secara otomatis — `$1`, `$2` untuk PostgreSQL; `?` untuk MySQL; `:1`, `:2` untuk Oracle.

   Asli (baris 123) : > ****Good to know:**** Cache scope adalah full response envelope per kombinasi `params` dan subset `widgets[]`.

   Usulan (baris 123): > ****Perlu diketahui:**** Cache scope adalah full response envelope per kombinasi `params` dan subset `widgets[]`.

   Alasan : Label "Good to know" terdaftar di Bagian C glossary dengan padanan "Perlu diketahui".

3. **Baris 84 — Kategori 1 (terjemahan kaku)**

   Asli   : **Validator scan** setiap SQL widget dan throw error bila ada placeholder yang tidak terdaftar di `params`.

   Usulan : **Validator memindai** setiap SQL widget dan throw error bila ada placeholder yang tidak terdaftar di `params`.

   Alasan : "scan" sebagai verba tanpa konjugasi Indonesia menjadikan kalimat tidak gramatikal; padanan "memindai" memperbaiki tata bahasa.

4. **Baris 123 — Kategori 5 (Deteksi B + E gabungan)**

   Asli   : Cascade invalidation **otomatis fire** saat ada operasi WRITE (create/update/delete) pada tabel yang terdaftar di `invalidates`.

   Usulan : Cascade invalidation **otomatis terpicu** saat ada operasi WRITE (create/update/delete) pada tabel yang terdaftar di `invalidates`.

   Alasan : "fire" sebagai verba campur kode tanpa padanan glossary; "terpicu" adalah padanan natural untuk event yang aktif sendiri.

5. **Baris 155 — Kategori 5 (Deteksi E)**

   Asli   : Validator **dijalankan otomatis** sebelum generator menulis file.

   Usulan : Validator **dijalankan secara otomatis** sebelum generator menulis file.

   Alasan : "otomatis" menerangkan cara verba "dijalankan" sehingga harus didahului "secara".

## Ringkasan
- Total temuan: 13 (dikelompokkan menjadi 5 nomor)
- Sudah direvisi: belum, menunggu approve
