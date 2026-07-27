# MLOps Hands-On 1 | Risk Score Prediction (Chicago Crimes) 

**Group 5**
**Anggota:** 
- Nadia Aisyah Fazila
- Sabbia Meilandri Putri Delarosya

## 1. Penjelasan Singkat Dataset

- Sumber: [Chicago Crimes](https://drive.google.com/file/d/12K13Az7ynV3ZVimSARImgggnbDbVqjk_/view?usp=sharing)
- Subset yang digunakan: 

    **3 tahun terakhir yang tersedia di dataset (2024-2026)**.

    **Justifikasi:**
    1. **Rentang tahun dipilih, bukan area/district**, supaya variasi spasial (hotspot vs area aman) tetap utuh untuk dianalisis. Ini krusial karena Risk Score sangat bergantung pada sinyal spasial yang kaya. Membatasi ke satu area/district justru menghilangkan variasi itu sebelum sempat dianalisis.
    2. **3 tahun** dipilih supaya pola musiman (`month`, `day-of-week`) tidak bias ke satu musim/periode tertentu, sekaligus menggunakan data paling terkini yang tersedia supaya relevan dengan kondisi kejahatan Chicago saat ini.

- Jumlah baris setelah subset 3 tahun (2024-2026): **553.919 baris**.
- Jumlah baris setelah cleaning: **550.896 baris** (3.023 baris atau ~0,55% dibuang karena koordinat/tanggal tidak valid, seperti koordinat (0,0) atau di luar bounding box Chicago). Persentase yang dibuang tergolong kecil, menandakan kualitas data cukup baik untuk subset ini.

## 2. Insight EDA

**Temuan penting: artefak pencatatan waktu:**
- Saya menemukan **5,80% baris** dengan waktu kejadian yang kemungkinan tidak presisi, yaitu pada jam 0 dan 12 yang menunjukkan lonjakan tidak wajar pada beberapa grafik awal (jauh di atas jam-jam sekitarnya).
- Verifikasi lewat distribusi menit mengonfirmasi bahwa pada jam 0, menit `:00` muncul 10.571 kali (~6x lebih sering dari menit tersering berikutnya), dan pada jam 12, menit `:00` muncul 6.424 kali (~3,6x lebih sering), jauh melebihi menit lain yang tersebar relatif merata.
- Ini mengindikasikan waktu kejadian **dibulatkan/default** saat tidak diketahui pasti oleh pencatat, bukan sinyal kejahatan asli. Perbandingan **pola jam sebelum vs sesudah baris ini dikeluarkan** membuktikan dampaknya signifikan:
    - **Sebelum** exclude: jam 0 melonjak ke ~20.000 kejadian (vs ~9.000-11.000 di jam sekitarnya) dan jam 12 menonjol tidak wajar (~16.700).
    - **Sesudah** exclude: kurva jadi mulus dan masuk akal secara domain (aktivitas rendah dini hari, naik bertahap, puncak sore-malam jam 15-16).
- Karena dampaknya terbukti besar, baris-baris ini **saya keluarkan dari seluruh analisis selanjutnya** (feature engineering, severity scoring, pseudo-labeling).

**Pola jam: hari kerja vs akhir pekan:**
Hari kerja menunjukkan pola "gundukan siang-sore" yang jelas (naik bertahap dari pagi, puncak sekitar jam 12 dan 15-16), sedangkan akhir pekan jauh lebih flat pada rentang jam yang sama.

**Pola waktu berbeda per jenis kejahatan:**
Lima jenis kejahatan terbanyak menunjukkan pola jam yang cukup berbeda satu sama lain:
- **THEFT** sangat terkonsentrasi siang-sore (puncak di jam 12-16), sejalan dengan pencurian oportunistik yang butuh keramaian/aktivitas publik.
- **BATTERY** dan **ASSAULT** tetap tinggi di dini hari (jam 0-2), kemungkinan terkait pengaruh alkohol di malam hari, lalu naik lagi di sore-malam.
- **MOTOR VEHICLE THEFT** meningkat di jam malam (18-23), sejalan dengan kendaraan yang diparkir tanpa pengawasan serta faktor visibilitas yang rendah akibat jalanan yang sudah gelap dan sepi, sehingga menurunkan risiko pelaku untuk teridentifikasi.

Perbedaan pola ini menunjukkan bahwa "jam rawan" **tidak seragam** antar jenis kejahatan. Ini menjustifikasi kenapa fitur temporal idealnya cukup detail untuk menangkap perbedaan ini.

**Interaksi hari x jam:**
Heatmap kombinasi hari-jam menunjukkan aktivitas yang lebih tinggi di jam sore-malam (15-19), khususnya mendekati akhir pekan (Jumat-Sabtu), pola yang tidak terlihat sejelas ini bila `dow` dan `hour` dianalisis terpisah satu-satu.

**Kesimpulan:**
Insight-insight di atas menjustifikasi kenapa fitur temporal (cyclical encoding) *dan* spasial (grid aggregation) sama-sama diperlukan untuk memodelkan Risk Score: pola kejahatan jelas tidak seragam lintas waktu maupun lokasi, bahkan berinteraksi satu sama lain (hari × jam, jenis kejahatan x jam), bukan sekadar fungsi linear sederhana dari salah satu dimensi saja. EDA ini juga menunjukkan pentingnya memeriksa kualitas data sebelum feature engineering, karena artefak pencatatan waktu (jam 0 / 12) dapat secara signifikan mendistorsi kesimpulan bila tidak ditangani.

## 3. Justifikasi Keputusan Desain

### 3.1 Feature Engineering

**Ukuran grid (sel spasial):** Saya menguji dua opsi secara kuantitatif sebelum memutuskan:

| Ukuran | Jumlah sel unik | Rata-rata kejadian/sel |
|---|---|---|
| 2 desimal (~1.110 m/sel) | 726 | 714,8 |
| 3 desimal (~111 m/sel) | 37.996 | 13,7 |

Saya memilih **3 desimal (~111 m/sel)**, meskipun rata-rata kejadian per sel jauh lebih kecil. Pertimbangannya:
- 2 desimal terlalu kasar (~1,1 km/sel, hanya 726 sel untuk seluruh Chicago). Risk Score pada resolusi ini terlalu tergeneralisasi untuk actionable secara praktis (area rawan dan aman yang berdekatan bisa tercampur dalam satu sel).
- 3 desimal memberi resolusi setara skala blok jalan, jauh lebih berguna untuk prediksi risiko lokasi spesifik.
- Sparsity per sel (rata-rata 13,7 kejadian) yang menjadi trade-off-nya saya tangani lewat spatial smoothing berbasis BallTree pada tahap Pseudo-Labeling (lihat 3.2), sel dengan data tipis "meminjam" sinyal dari sel-sel tetangga, sehingga estimasi risiko tetap stabil tanpa kehilangan resolusi spasial.

**Fitur agregat tambahan:**
- **`crime_diversity`** 

    Jumlah jenis kejahatan (`Primary Type`) berbeda per unit (sel × hari × jam). Area dengan banyak jenis kejahatan berbeda mengindikasikan risiko yang lebih beragam/tidak terprediksi, berbeda dari area yang hanya didominasi satu jenis kejahatan ringan berulang.
- **`arrest_rate`**

    Proporsi kejadian yang berujung penangkapan per unit. Area dengan arrest rate rendah bisa mengindikasikan penegakan hukum yang kurang menjangkau, yang secara tidak langsung berkorelasi dengan risiko berkelanjutan di lokasi itu.

Keduanya secara konsep adalah Feature Engineering (fitur input), namun secara teknis baru bisa dihitung setelah tabel `unit` terbentuk (butuh hasil groupby sel x hari x jam), bukan berarti keduanya bagian dari pembentukan label `risk_score`.

**Fitur temporal tambahan:** `is_weekend` (flag biner dari `dow`) saya tambahkan karena EDA menunjukkan pola jam hari kerja vs akhir pekan cukup berbeda bentuknya. `month` sengaja tidak saya sertakan sebagai fitur di level unit, karena akan memecah unit analisis (sel x hari x jam) menjadi (sel x bulan x hari x jam), menambah 12 kemungkinan dimensi baru yang akan membuat data jauh lebih sparse tanpa data pendukung yang cukup, mengingat sel pada resolusi 3 desimal sudah cukup tipis.

### 3.2 Pseudo-Labeling

**Severity Scoring (2 level: kategori + modifier).** Saya memakai skor dasar per `Primary Type` (berdasarkan tingkat bahaya obyektif terhadap keselamatan manusia, kejahatan terhadap nyawa/tubuh > kejahatan bersenjata > properti > pelanggaran administratif) ditambah modifier dari kata kunci di `Description` (mis. `AGGRAVATED`, `ARMED`, `SIMPLE`, `ATTEMPT`). Pendekatan ini menghindari masalah baseline di mana satu default dipakai untuk seluruh kombinasi yang tidak tercakup.

Hasilnya: cakupan **100%** (0% fallback), naik drastis dari baseline yang **80,46%** baris kena default seragam. Ranking severity per Primary Type konsisten dengan intuisi keamanan publik: `HOMICIDE` (99,99), `CRIMINAL SEXUAL ASSAULT` (99,45), `KIDNAPPING` (92,28) berada di puncak; `BATTERY` (55,2), `ASSAULT` (51,7) di tengah; pelanggaran administratif di dasar, tidak ada anomali urutan yang tidak masuk akal.

**Temporal Decay (Exponential Half-Life).** Saya mengganti peluruhan linear (yang membuat kejadian tertua berbobot persis 0) dengan bentuk eksponensial $w(t) = 0.5^{age\_days/H}$, dengan $H$ = 180 hari (6 bulan). Pola kejahatan cenderung bergeser dalam hitungan bulan (musim, penegakan hukum), bukan hari (terlalu sensitif) atau tahun (terlalu lambat/stale). Hasil: `w_time_exp` berkisar 0,0408-1,0, kejadian tertua di subset tetap berkontribusi ~4%, tidak pernah nol sepenuhnya seperti pada peluruhan linear.

**Spatial Aggregation (BallTree + Haversine + Gaussian Kernel).** Saya mengganti rata-rata kasar grid 3×3 (baseline) dengan jarak haversine sesungguhnya (`sklearn.neighbors.BallTree`) dan bobot kernel Gaussian $w = \exp(-d^2/(2\sigma^2))$, radius 1,5 km, sigma 0,6 km. Rata-rata terdapat 494,4 sel bertetangga per sel dalam radius ini (dari total 37.996 sel unik). Untuk efisiensi komputasi, saya vectorize perhitungannya memakai sparse matrix (`scipy.sparse`, 18.784.700 pasangan tetangga non-zero) alih-alih loop per baris, proses selesai dalam hitungan detik, bukan belasan menit. Hasil `risk_raw_v2` berkisar 0,11-194,37, dengan smoothing yang membuat sel-sel dengan data tipis (rata-rata `crime_count` ~1-2 per unit) menjadi lebih stabil karena "meminjam" sinyal dari tetangganya.

**Normalisasi (log1p + Min-Max).** Distribusi `risk_raw_v2` sangat skewed, sehingga saya mengompresnya dengan `log1p` sebelum melakukan min-max scaling ke rentang 0-100. Dibanding peringkat persentil, `log1p` tetap mempertahankan informasi magnitude asli (seberapa jauh perbedaan risiko antar sel); dibanding clipping kuantil, `log1p` tidak membuang informasi apa pun dari sel ekstrem. Hasil akhir `risk_score`: mean 54,58, std 4,17, rentang 0-100, distribusi bell-shaped yang jauh lebih sehat dibanding min-max langsung (yang menumpukkan >80% sel di rentang 0-20 dari skala).

## 4. Refleksi

**Kendala:**
- Sebagian kecil data (~0,55%) memiliki koordinat atau tanggal yang tidak valid, saya tangani dengan langkah cleaning standar dari tutorial.
- Saya menemukan artefak pencatatan waktu (5,80% baris pada jam 00:00/12:00 dengan menit persis `:00`) yang jika dibiarkan akan mendistorsi kesimpulan EDA dan Risk Score, saya verifikasi lewat distribusi menit sebelum memutuskan mengeluarkannya dari analisis.
- Perhitungan spatial smoothing awal (loop per baris) terlalu lambat untuk 433.002 unit, saya optimasi ulang memakai sparse matrix vectorization agar tetap efisien dijalankan di Colab.

**Solusi/pembelajaran:** Saya belajar bahwa kualitas data (termasuk artefak pencatatan yang tidak terlihat sekilas) sama pentingnya dengan pemilihan metode feature engineering, keduanya sama-sama menentukan validitas Risk Score akhir. Saya juga belajar pentingnya mempertimbangkan efisiensi komputasi (vectorization) saat resolusi grid diperhalus, bukan hanya akurasi konsep.

## 5. Struktur Repository

```
├── Hands_On_1.ipynb                 # notebook lengkap 
├── features_labels.csv              # dataset akhir 
├── features_labels.parquet          # dataset akhir 
├── README.md                        # laporan .md
├── HO1_MLOps_NadiaAisyahFazila.pdf  # laporan .pdf     
```