# Laporan Review — frontend-mapping.md

Folder: `restforge-handbook/features/dashboard/`
Indikator: TEMUAN (8 temuan; 2 dikelompokkan)

## Temuan

1. **Heading bilingual (Deteksi F) — 2 lokasi: baris 1, 537**

   Asli (baris 1)   : # Pemetaan ke Widget Frontend **(Frontend Widget Mapping)**

   Usulan (baris 1) : # Pemetaan ke Widget Frontend

   Asli (baris 537) : ## Pemisahan Concern **(Separation of Concerns)**

   Usulan (baris 537): ## Pemisahan Concern

   Alasan : Bagian Inggris dalam kurung adalah terjemahan murni dari judul Indonesia; bukan singkatan/istilah teknis bagian dari nama.

2. **Label "Good to know" (Bagian C glossary) — 6 lokasi: baris 55, 262, 294, 411, 465, 561**

   Asli (baris 55)  : > ****Good to know:**** Bila subtitle bersifat statis dan tidak berubah berdasarkan data, dashboard tidak diperlukan.

   Usulan (baris 55): > ****Perlu diketahui:**** Bila subtitle bersifat statis dan tidak berubah berdasarkan data, dashboard tidak diperlukan.

   Asli (baris 262) : > ****Good to know:**** Warna per arc, label legend, dan teks deskriptif card ditentukan frontend.

   Usulan (baris 262): > ****Perlu diketahui:**** Warna per arc, label legend, dan teks deskriptif card ditentukan frontend.

   Asli (baris 294) : > ****Good to know:**** `to_char(..., 'Mon')` menghasilkan label manusiawi (`Jan`, `Feb`, ...) yang langsung bisa dipakai sebagai label axis.

   Usulan (baris 294): > ****Perlu diketahui:**** `to_char(..., 'Mon')` menghasilkan label manusiawi (`Jan`, `Feb`, ...) yang langsung bisa dipakai sebagai label axis.

   Asli (baris 411) : > ****Good to know:**** Key snake_case dari SQL (misalnya `north_america`) menjadi key JSON yang stabil.

   Usulan (baris 411): > ****Perlu diketahui:**** Key snake_case dari SQL (misalnya `north_america`) menjadi key JSON yang stabil.

   Asli (baris 465) : > ****Good to know:**** Library seperti ECharts dan AntV/G2 dapat mengonsumsi long format secara native melalui konfigurasi `seriesField`, sehingga pre-process tidak selalu diperlukan.

   Usulan (baris 465): > ****Perlu diketahui:**** Library seperti ECharts dan AntV/G2 dapat mengonsumsi long format secara native melalui konfigurasi `seriesField`, sehingga pre-process tidak selalu diperlukan.

   Asli (baris 561) : > ****Good to know:**** Semua nilai numeric dikembalikan sebagai string di response (misalnya `"value": "500000000"`).

   Usulan (baris 561): > ****Perlu diketahui:**** Semua nilai numeric dikembalikan sebagai string di response (misalnya `"value": "500000000"`).

   Alasan : Label "Good to know" terdaftar di Bagian C glossary dengan padanan "Perlu diketahui".

## Ringkasan
- Total temuan: 8 (dikelompokkan menjadi 2 nomor)
- Sudah direvisi: belum, menunggu approve
