# Catatan Minggu 1 - Analisis Kualitas Air Smart Aqua

Dokumen ini rangkuman pengerjaan tugas Minggu 1 untuk studi kasus budidaya nila intensif
(tugas dua minggu, Analisis Kualitas Air pada Smart Aquaculture). Isinya mencakup audit data,
analisis deskriptif, dan analisis diagnostik. Analisis prediktifnya masuk Minggu 2, tapi arah
fiturnya sudah kami siapkan di bagian akhir.

Beberapa berkas terkait:

- Notebook kode: [`src/Tugas_Minggu_1.ipynb`](../src/Tugas_Minggu_1.ipynb)
- Data: [`data/smart_aqua_synthetic_water_quality.csv`](../data/smart_aqua_synthetic_water_quality.csv)
- Instruksi: [`data/tugas_2_minggu_water_quality (1).pdf`](<../data/tugas_2_minggu_water_quality (1).pdf>)

---

## 1. Gambaran umum

Unit budidaya ini mengelola 8 kolam dengan sensor yang merekam suhu, pH, DO, amonia, nitrit,
dan kekeruhan tiap 30 menit. Rata-rata kualitas air terlihat memadai, tetapi masih muncul
tanda stres ikan di waktu dan kolam tertentu. Manajer belum tahu apakah itu karena siklus
harian, biomassa, pakan, hujan, gangguan aerator, atau kesalahan sensor.

Di Minggu 1 kami menjawab dua tingkat pertanyaan:

- Deskriptif: apa yang sebenarnya terjadi pada kualitas air selama pengamatan.
- Diagnostik: kondisi apa yang berkaitan dengan munculnya risiko, dan kenapa polanya begitu.

---

## 2. Tentang dataset

- Bentuk: 23.040 baris x 26 kolom. Setara 8 kolam x 60 hari x 48 observasi/hari.
- Periode: 5 Januari 2026 00:00 sampai 5 Maret 2026 23:30.
- Satu baris = satu kolam pada satu waktu pengukuran.
- Spesiesnya cuma satu, yaitu nila.
- Pembagian data: TRAIN 16.128, VALIDATION 3.456, TEST 3.456.

Kelompok variabel:

| Kelompok | Variabel | Peran |
|---|---|---|
| Identitas dan waktu | `record_id`, `timestamp`, `pond_id`, `day_of_cycle`, `hour_decimal`, `time_period` | pengurutan dan pola temporal |
| Kondisi kolam | `pond_volume_m3`, `stocking_density_kg_m3`, `biomass_kg` | perbedaan antar kolam dan beban biologis |
| Kualitas air | `temperature_c`, `ph`, `dissolved_oxygen_mg_l`, `ammonia_mg_l`, `nitrite_mg_l`, `turbidity_ntu` | respons dan prediktor utama |
| Operasional | `feed_kg`, `aerator_command`, `aerator_status`, `aerator_fault`, `water_exchange_pct` | tindakan pengelolaan |
| Lingkungan | `rainfall_mm` | cuaca, juga sumber confounding waktu |
| Kualitas dan keluaran | `sensor_quality_flag`, `fish_stress_event`, `risk_next_3h`, `dataset_split` | audit, kejadian, target, split temporal |

Ambang kondisi berisiko yang dipakai (sesuai dokumen tugas): DO < 3,5 mg/L, amonia > 0,55
mg/L, atau nitrit > 0,35 mg/L. Target `risk_next_3h` bernilai 1 kalau kondisi berisiko terjadi
dalam 6 interval (3 jam) berikutnya. Enam observasi terakhir tiap kolam tidak punya target,
totalnya 48 baris kosong.

---

## 3. Audit kualitas data (bagian A)

### A.1 Struktur dan interval

| Item | Nilai |
|---|---|
| Jumlah baris | 23.040 |
| Jumlah kolom | 26 |
| Jumlah kolam | 8 (P01-P08) |
| Periode | 5 Jan 2026 sampai 5 Mar 2026 (60 hari) |
| Interval | 30 menit (2.880 observasi per kolam) |

Angka ini cocok dengan hitungan 8 x 60 x 48 = 23.040, jadi dari sisi dimensi datanya utuh.

### A.2 Duplikasi dan kelengkapan waktu

- `record_id` unik, tidak ada duplikat.
- Pasangan `pond_id` + `timestamp` juga unik, tidak ada duplikat.
- Setiap kolam punya 2.880 observasi dengan selisih waktu selalu 30 menit, tanpa lompatan.

Jadi nilai sensor yang hilang bukan berupa baris yang terhapus, tapi tetap ada sebagai `NaN`
sambil ditandai flag.

### A.3 Tipe data dan nilai hilang

| Variabel | Tipe | Missing | % | Unik | Min | Max |
|---|---|---:|---:|---:|---:|---:|
| `record_id` | str | 0 | 0 | 23.040 | - | - |
| `timestamp` | datetime64 | 0 | 0 | 2.880 | - | - |
| `pond_id` | str | 0 | 0 | 8 | - | - |
| `day_of_cycle` | int64 | 0 | 0 | 60 | 1 | 60 |
| `hour_decimal` | float64 | 0 | 0 | 48 | 0 | 23,5 |
| `time_period` | str | 0 | 0 | 4 | - | - |
| `species` | str | 0 | 0 | 1 | - | - |
| `pond_volume_m3` | int64 | 0 | 0 | 7 | 45 | 54 |
| `stocking_density_kg_m3` | float64 | 0 | 0 | 424 | 15,77 | 39,26 |
| `biomass_kg` | float64 | 0 | 0 | 438 | 820,0 | 1.845,4 |
| `temperature_c` | float64 | 0 | 0 | 556 | 25,84 | 31,65 |
| `ph` | float64 | 41 | 0,178 | 147 | 6,92 | 8,40 |
| `dissolved_oxygen_mg_l` | float64 | 38 | 0,165 | 539 | -0,99 | 15,88 |
| `ammonia_mg_l` | float64 | 32 | 0,139 | 462 | 0,110 | 0,593 |
| `nitrite_mg_l` | float64 | 0 | 0 | 410 | 0,103 | 0,527 |
| `turbidity_ntu` | float64 | 38 | 0,165 | 459 | 26,1 | 80,7 |
| `feed_kg` | float64 | 0 | 0 | 661 | 0 | 25,53 |
| `aerator_command` / `aerator_status` | str | 0 | 0 | 2 | - | - |
| `aerator_fault` | int64 | 0 | 0 | 2 | 0 | 1 |
| `rainfall_mm` | float64 | 0 | 0 | 146 | 0 | 10,59 |
| `water_exchange_pct` | int64 | 0 | 0 | 6 | 0 | 14 |
| `sensor_quality_flag` | str | 0 | 0 | 6 | - | - |
| `fish_stress_event` | int64 | 0 | 0 | 2 | 0 | 1 |
| `risk_next_3h` | float64 | 48 | 0,208 | 2 | 0 | 1 |
| `dataset_split` | str | 0 | 0 | 3 | - | - |

Nilai hilang hanya ada di kolom sensor dan target `risk_next_3h`, semuanya di bawah 0,25%.

### A.4 Flag sensor dan contoh anomali

| `sensor_quality_flag` | Jumlah | % |
|---|---:|---:|
| OK | 22.356 | 97,03 |
| PH_DRIFT | 327 | 1,42 |
| SENSOR_MISSING | 147 | 0,64 |
| DO_FLATLINE | 145 | 0,63 |
| DO_SPIKE | 61 | 0,26 |
| MULTIPLE | 4 | 0,02 |

Contoh tiap anomali yang kami temukan di grafik waktu:

- Spike: DO melonjak ke nilai tak wajar, rentangnya -0,99 sampai 15,88 mg/L, misalnya di
  P01 tanggal 31 Jan 2026 pukul 03:30. Munculnya cuma satu titik tapi bisa merusak statistik
  nilai ekstrem.
- Flatline: DO benar-benar datar, contohnya P01 konstan 5,53 pada 1 Feb 2026 pukul 00:00-06:00.
- Drift: pH P05 perlahan bergeser sampai 8,40, padahal rata-rata kolamnya sekitar 7,37.
- Missing: nilai kosong hanya ada di baris SENSOR_MISSING dan 2 baris MULTIPLE. Crosstab
  menunjukkan SENSOR_MISSING selalu bertemu nilai kosong, sedangkan MULTIPLE hanya 2 dari 4.

### A.5 Aturan penanganan nilai bermasalah

Kami bedakan perlakuannya per tahap supaya tidak asal buang:

- Deskriptif: semua baris dipakai, statistik dihitung `dropna()` per kolom. Versi tanpa baris
  ber-flag (`df_clean`) dipakai sebagai uji sensitivitas.
- Diagnostik: baris ber-flag tetap dipertahankan, `sensor_quality_flag` dijadikan kontrol,
  karena flag itu sendiri informatif.
- Prediktif (Minggu 2): flag jadi fitur indikator, imputasi dan standardisasi dilatih hanya
  dari TRAIN, baris target kosong tidak dipakai latih.

File CSV aslinya tidak diubah; semua pembersihan hanya pada salinan DataFrame.

---

## 4. Analisis deskriptif (bagian B)

### B.1 Statistik deskriptif

| Parameter | Mean | Median | Std | Min | Q1 | Q3 | Max | IQR | CV (%) |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| `temperature_c` | 28,601 | 28,600 | 1,240 | 25,84 | 27,53 | 29,67 | 31,65 | 2,14 | 4,34 |
| `ph` | 7,353 | 7,350 | 0,182 | 6,92 | 7,21 | 7,49 | 8,40 | 0,28 | 2,47 |
| `dissolved_oxygen_mg_l` | 5,206 | 5,160 | 0,886 | -0,99 | 4,63 | 5,76 | 15,88 | 1,13 | 17,02 |
| `ammonia_mg_l` | 0,336 | 0,333 | 0,088 | 0,110 | 0,273 | 0,398 | 0,593 | 0,125 | 26,01 |
| `nitrite_mg_l` | 0,281 | 0,274 | 0,079 | 0,103 | 0,217 | 0,337 | 0,527 | 0,120 | 28,28 |
| `turbidity_ntu` | 49,340 | 48,700 | 7,814 | 26,10 | 43,60 | 54,60 | 80,70 | 11,00 | 15,84 |

Soal CV: nilai CV = std/mean. Untuk amonia dan nitrit yang mean-nya kecil (0,336 dan 0,281),
penyebut yang kecil membuat CV terlihat besar walaupun variasi absolutnya tidak. Kalau mean
mendekati nol, CV bahkan bisa tak terdefinisi. Jadi untuk parameter sekecil ini kami lebih
memakai IQR atau simpangan baku absolut.

### B.2 Perbandingan antar kelompok

Rata-rata per kolam:

| pond | suhu | pH | DO | amonia | nitrit | turbidity |
|---|---:|---:|---:|---:|---:|---:|
| P01 | 28,14 | 7,35 | 5,45 | 0,248 | 0,202 | 42,1 |
| P02 | 28,44 | 7,36 | 5,31 | 0,315 | 0,258 | 47,4 |
| P03 | 28,64 | 7,34 | 5,04 | 0,388 | 0,327 | 54,4 |
| P04 | 28,54 | 7,36 | 5,15 | 0,342 | 0,285 | 49,5 |
| P05 | 29,14 | 7,37 | 5,22 | 0,337 | 0,278 | 49,2 |
| P06 | 28,74 | 7,35 | 4,91 | 0,425 | 0,362 | 56,9 |
| P07 | 28,34 | 7,35 | 5,45 | 0,267 | 0,221 | 43,8 |
| P08 | 28,84 | 7,35 | 5,12 | 0,368 | 0,311 | 51,4 |

Rata-rata per `time_period`:

| time_period | suhu | pH | DO | amonia | nitrit | turbidity |
|---|---:|---:|---:|---:|---:|---:|
| dawn | 27,14 | 7,16 | 4,46 | 0,335 | 0,282 | 47,4 |
| morning | 28,53 | 7,35 | 4,93 | 0,330 | 0,280 | 50,2 |
| afternoon | 30,06 | 7,55 | 6,09 | 0,335 | 0,278 | 49,5 |
| night | 28,67 | 7,36 | 5,34 | 0,346 | 0,282 | 50,3 |

Rata-rata per fase siklus:

| fase | suhu | pH | DO | amonia | nitrit | turbidity |
|---|---:|---:|---:|---:|---:|---:|
| Awal (hari 1-20) | 28,78 | 7,358 | 5,344 | 0,269 | 0,218 | 43,35 |
| Tengah (21-40) | 28,63 | 7,349 | 5,225 | 0,337 | 0,278 | 49,03 |
| Akhir (41-60) | 28,40 | 7,354 | 5,048 | 0,404 | 0,346 | 55,65 |

Yang kami baca dari sini: DO dan pH mengikuti siklus harian (puncak siang, terendah saat
fajar). Lalu dari fase Awal ke Akhir, amonia, nitrit, dan kekeruhan naik konsisten sementara DO
turun, yang sepertinya efek biomassa yang makin besar. Selain itu, perbedaan antar kolam
lumayan besar; P06 paling buruk, P01 dan P07 paling bersih.

### B.3 Stabil vs berfluktuasi

Paling stabil adalah ph (CV 2,5%) dan temperature_c (4,3%). Paling berfluktuasi adalah
nitrite_mg_l (28,3%), ammonia_mg_l (26,0%), dissolved_oxygen_mg_l (17,0%), dan turbidity_ntu
(15,8%). Tapi lagi-lagi, CV amonia/nitrit sebagian cuma artefak mean kecil, jadi dibaca
bareng IQR. Menurut kami yang paling penting diperhatikan tetap DO dan nitrit karena
keduanya langsung memicu kondisi berisiko.

### B.4 Frekuensi kondisi berisiko

Total 4.849 observasi atau 21,05%. Pemicu terbanyak adalah nitrit > 0,35 mg/L (4.795 baris);
amonia (105) dan DO rendah (181) jarang.

| pond | risk_count | risk_pct |
|---|---:|---:|
| P06 | 1.590 | 55,21 |
| P03 | 1.132 | 39,31 |
| P08 | 982 | 34,10 |
| P04 | 571 | 19,83 |
| P05 | 382 | 13,26 |
| P02 | 172 | 5,97 |
| P07 | 16 | 0,56 |
| P01 | 4 | 0,14 |

Proporsi risiko per jam relatif datar (sekitar 19-23%), cuma sedikit naik di dini hari.
Artinya perbedaan antar kolam jauh lebih besar daripada perbedaan antar jam. Heatmap
kolam-jam juga menunjukkan P03, P06, P08 tetap tinggi di hampir semua jam.

### B.5 Distribusi fish_stress_event

Total hanya 47 kejadian (0,20%) dari 23.040 observasi. Sebarannya hampir merata: P03 = 9,
P01 = 8, P06 = 7, P05 = 7, P04 = 6, P08 = 4, P02 = 3, P07 = 3. Jam tersering 22:00, 01:30,
dan 08:00 (masing-masing 0,63%). Menariknya P06 yang risikonya paling tinggi justru cuma 7
event, jadi event stres ini tidak bisa diandalkan sebagai prediktor dan hanya kami pakai untuk
deskriptif/diagnostik.

---

## 5. Analisis diagnostik (bagian C)

### C.1 Risiko antar kolam

Korelasi antar 8 kolam antara tingkat risiko dan karakter kolam:

| Faktor | Korelasi dengan risk_rate |
|---|---:|
| avg_density | 0,963 |
| avg_turbidity | 0,961 |
| avg_biomass | 0,941 |
| avg_temp | 0,500 |

P06 55,2% vs P01 0,14%, bedanya lebih dari 55 poin persen atau sekitar 400 kali secara rasio.
Korelasi dengan biomassa, kepadatan, dan kekeruhan sangat kuat; suhu cuma sedang dan P05 yang
paling panas malah berisiko rendah. Secara biologis masuk akal: kolam padat membuang lebih
banyak nitrogen. Tapi kolamnya cuma 8 dan faktor-faktornya saling berkaitan, jadi ini asosiasi,
bukan klaim kausal.

### C.2 Aerator dan DO

Agregatnya menyesatkan: DO OFF (5,47) lebih tinggi daripada ON (5,07). Setelah distratifikasi
hasilnya berubah, waktu dawn DO ON (4,49) lebih tinggi daripada OFF (3,76), dan pada kepadatan
tinggi hubungannya juga terbalik (OFF 4,39 vs ON 5,00). Jadi angka agregat muncul karena
periode siang ber-DO tinggi kebetulan aeratornya mati. Yang paling menjelaskan adalah
aerator_command: DO terendah (4,41) terjadi saat perintah ON tapi status OFF, yaitu kondisi
fault. Semua baris aerator_fault = 1 berstatus OFF dan DO-nya paling rendah. Jadi penurunan DO
lebih terkait gangguan aerator, bukan perintah ON/OFF biasa.

### C.3 Gangguan aerator

Ada 479 kejadian awal gangguan. Rata-rata DO sebelum 5,059, sesudah 4,838, dan tanpa gangguan
5,226. Jadi DO turun di sekitar gangguan dan tetap tertekan sesudahnya. Catatannya, gangguan
tidak acak karena lebih banyak di kolam padat, jadi sebagian efeknya bisa confounding dan
perlu matching kolam/jam untuk klaim yang lebih kuat.

### C.4 Efek tertunda pakan

Korelasi pakan dengan parameter sangat lemah dan tidak ada puncak lag yang jelas:

| lag (interval) | 1 | 4 | 8 | 12 |
|---|---:|---:|---:|---:|
| ammonia_corr | 0,030 | 0,041 | 0,031 | 0,035 |
| nitrite_corr | 0,032 | 0,034 | 0,027 | 0,030 |
| turbidity_corr | 0,114 | 0,083 | 0,050 | 0,020 |
| DO_corr | -0,005 | 0,013 | 0,026 | 0,070 |

Kekeruhan sedikit lebih tinggi di lag terpendek tapi tetap kecil. Pada data ini pengaruh
langsung pakan tidak terlihat. Fitur lag tetap kami pertimbangkan untuk Minggu 2, dan harus
dibentuk per kolam supaya tidak menggeser data antar kolam.

### C.5 Hujan

Secara agregat perbedaan antar kategori hujan kecil, dan kategori hujan lebat isinya sangat
sedikit. Setelah dipecah per jam, pola DO tetap didominasi siklus harian. Karena hujan muncul
di jam tertentu, hasil agregat bisa berbeda dari per strata, itu sebabnya stratifikasi penting.
Belum ada bukti kuat hujan langsung mengubah kualitas air.

### C.6 Pertukaran air

| Waktu | amonia | nitrit |
|---|---:|---:|
| Sebelum (-30m) | 0,338 | 0,292 |
| Saat (t0) | 0,261 | 0,244 |
| Sesudah (+30m) | 0,261 | 0,242 |
| Sesudah (+1j) | 0,266 | 0,244 |

Amonia dan nitrit turun jelas setelah pertukaran dan bertahan sampai sekitar satu jam.
Penurunannya konsisten di kedelapan kolam (perubahan nitrit -0,034 sampai -0,064 mg/L). Tapi
efeknya kecil dan tidak mengubah status risiko secara drastis, karena sumber nitrogen tetap
berjalan dan P06 tetap tertinggi (0,32). Jumlah event per kolam juga sedikit, jadi perlu data
tambahan untuk menyimpulkan efektivitas.

---

## 6. Kesimpulan Minggu 1

Tiga temuan utama:

1. Kualitas data umumnya bagus. Tidak ada duplikasi dan interval waktunya konsisten. Anomali
   tetap ada, sekitar 2,97% baris ber-flag dan 0,65% nilai sensor hilang, semuanya di sensor.
2. Risiko kualitas air lebih ditentukan karakter kolam daripada jam. Risikonya 21,05%,
   didominasi nitrit, dan menumpuk di P06/P03/P08 yang padat dan berbiomassa tinggi.
3. DO mengikuti siklus harian yang jelas, dan penurunan DO lebih berkaitan dengan gangguan
   aerator (fault) daripada status ON/OFF.

Dua masalah kualitas data:

1. Nilai sensor hilang, 147 baris SENSOR_MISSING plus 2 baris MULTIPLE.
2. Anomali sensor berupa DO spike/flatline dan pH drift yang bisa menggeser statistik nilai
   ekstrem.

Hipotesis diagnostik yang masih perlu diuji: kegagalan aerator menyebabkan DO turun dan
memperbesar risiko dalam 3 jam berikutnya setelah mengontrol kolam, jam, kepadatan, dan
biomassa. Belum kami uji secara kausal.

Rencana fitur untuk Minggu 2: sin/cos jam; lag DO, suhu, pH, amonia, nitrit (1-12 interval per
kolam); statistik bergerak yang hanya memakai masa lalu; waktu sejak pakan dan akumulasi
pakan; interaksi aerator x fault dan suhu x biomassa; serta indikator sensor_quality_flag.
Aturannya, record_id, risk_next_3h, dataset_split, dan informasi masa depan tidak boleh jadi
fitur; species konstan; fish_stress_event hanya untuk deskriptif/diagnostik.

---

## 7. Peta notebook

| Bagian | Isi | Sel |
|---|---|---|
| Setup | import dan konstanta | 2 |
| A.1 | baca CSV, dimensi, periode, interval | 5 |
| A.2 | duplikasi dan kelengkapan waktu | 6-7 |
| A.3 | tabel audit | 10-11 |
| A.4 | flag dan grafik spike/flatline/drift/missing | 14-18 |
| A.5 | aturan penanganan nilai bermasalah | 19-20 |
| B.1 | statistik deskriptif dan sensitivitas | 24-26 |
| B.2 | per kolam, time_period, jam, fase | 28-32 |
| B.3 | stabilitas (CV) | 34-35 |
| B.4 | frekuensi risiko dan heatmap | 37-41 |
| B.5 | distribusi stres ikan | 43-45 |
| C.1 | risiko antar kolam | 48-50 |
| C.2 | aerator vs DO | 52-55 |
| C.3 | gangguan aerator | 57-60 |
| C.4 | lag pakan | 61-63 |
| C.5 | hujan | 65-67 |
| C.6 | pertukaran air | 68-71 |
| Ringkasan | temuan dan rencana fitur | 72 |

Cara menjalankan:

```bash
jupyter notebook src/Tugas_Minggu_1.ipynb
```

Dependensi yang dipakai: pandas, numpy, matplotlib, dan seaborn. Notebook sudah kami coba
jalankan dari awal sampai akhir tanpa error, total 34 sel kode dan 12 gambar.

---

## 8. Perbaikan pada notebook

Versi awal hasil ekspor dari Colab masih ada beberapa masalah, ini yang kami rapikan:

| # | Masalah | Perbaikan |
|---|---|---|
| 1 | `np.nan` dipakai di B1 sebelum `import numpy` | menambah `import numpy as np` di Setup |
| 2 | import `scipy.stats.ttest_ind` gagal dan tidak dipakai | import dihapus |
| 3 | sel pertukaran air rusak, ada teks `pip install opencv-python` menyelip di kode | ditulis ulang jadi analisis sebelum/sesudah |
| 4 | sel `do_aerator_pond` terduplikasi | diganti stratifikasi lengkap |
| 5 | butir A.5 (aturan nilai bermasalah) belum ada | ditambahkan beserta `df_clean` dan uji sensitivitas |
| 6 | belum ada grafik missing dan spike | ditambah panel spike/flatline/drift/missing |
| 7 | hujan hanya agregat, pertukaran air hanya rata-rata grup | ditambah stratifikasi jam dan analisis sebelum/sesudah |
| 8 | banyak sel bagian C belum dijalankan | semua dijalankan ulang dan outputnya disimpan |
| 9 | markdown campur bahasa dan belum konsisten | dirapikan dan tiap butir diberi pembacaan hasil |

---

## 9. Keterbatasan dan jawaban pertanyaan refleksi

Keterbatasan umum: data ini sintetis, jumlah kolam hanya 8 sehingga korelasi antar kolam
rapuh, analisisnya observasional sehingga klaim kausal harus hati-hati, dan ambang risiko
adalah asumsi pembelajaran.

Jawaban singkat pertanyaan di dokumen tugas:

1. Yang tidak terlihat kalau hanya memakai rata-rata seluruh data: siklus harian DO/pH dan
   perbedaan risiko yang sangat besar antar kolam (P06 vs P01), keduanya tertutup oleh
   rata-rata global.
2. Contoh korelasi agregat yang berubah setelah stratifikasi: aerator dengan DO. Agregatnya
   menunjukkan OFF > ON, tapi setelah distratifikasi per time_period dan kepadatan arahnya
   berbalik (dawn: ON > OFF). Ini contoh Simpson's paradox.
3. Keputusan pembersihan yang paling memengaruhi hasil: penanganan spike dan flatline DO.
   Membuang atau membiarkannya mengubah statistik ekstrem dan analisis kejadian aerator,
   karena itu kami lakukan uji sensitivitas.
4. Model bagus di VALIDATION belum tentu bagus di TEST karena distribusinya bisa bergeser
   antar waktu. Ini terlihat dari prevalensi risk_next_3h: TRAIN sekitar 8,5%, VALIDATION
   sekitar 46%, dan TEST sekitar 64%.
5. Bukti tambahan yang diperlukan untuk klaim kausal operasi ke perbaikan kualitas air:
   eksperimen atau desain before-after-control dengan pencocokan kolam dan jam, kontrol
   confounding, dan replikasi pada kolam nyata.
