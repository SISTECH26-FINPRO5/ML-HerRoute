# Risk Score Feature Pipeline — Safe Route, Visual Risk Indicator, Safe Place Locator - Final Project

Notebook ini membangun **dataset fitur + label (risk score)** untuk tiga fitur produk:

- **Safe Route Recommendation & Risk Prediction**
- **Visual Risk Indicator** (🟢/🟡/🔴)
- **Safe Place Locator**

Karena data kejahatan & transit Jakarta (MRT) tidak tersedia untuk kebutuhan ini, notebook memakai **Chicago Crimes + stasiun CTA ('L' train)** sebagai *analog struktural*: satu baris kejahatan = satu insiden, satu stasiun CTA = satu stasiun MRT. 

Fitur di luar scope ini (SOS/emergency, anonymous reporting, integrasi kepolisian/CCTV, moda transportasi selain MRT) **tidak** direpresentasikan di notebook ini.

Link features_labels.csv (Tidak cukup untuk di push ke github karena size >= 100 MB): [https://drive.google.com/file/d/1MHamOo-2fhbgLtBjjNWZz4dCHH0FAcEV/view?usp=sharing]
---

## 1. Alur Pipeline

| # | Tahap | Ringkasan |
|---|---|---|
| 1 | Setup & Import | Load Chicago Crimes (3 tahun terakhir) + stasiun CTA |
| 2 | Pembersihan Data | Audit missing/duplikat, parsing tanggal, filter koordinat, buang artefak pencatatan waktu |
| 3 | Subset Spasial | Persempit ke radius 600 m jalan kaki dari stasiun CTA terdekat |
| 4 | EDA | Pola temporal (jam/hari/bulan) dan spasial (hotspot) |
| 5 | Feature Engineering | Cyclical encoding waktu, grid spasial 3 desimal (~110 m/sel), fitur Safe Place |
| 6 | Pseudo-Labeling | Severity scoring → temporal decay → spatial smoothing → normalisasi → **risk_score** |
| 7 | Audit & Penyimpanan | Validasi akhir, simpan `features_labels.csv/.parquet`, `model_config.csv`, `safe_places.csv` |

---

## 2. Keputusan & Alasan (per tahap)

### 2.1 Subset Data: 3 tahun, radius 600 m dari stasiun
- **Rentang tahun dipilih, bukan area/district**, supaya variasi spasial (hotspot vs area aman) tetap utuh untuk dianalisis. Ini krusial karena Risk Score sangat bergantung pada sinyal spasial yang kaya. Membatasi ke satu area/district justru menghilangkan variasi itu sebelum sempat dianalisis.
- **3 tahun** dipilih supaya pola musiman (`month`, `day-of-week`) tidak bias ke satu musim / periode tertentu, sekaligus menggunakan data paling terkini yang tersedia supaya relevan dengan kondisi kejahatan Chicago saat ini.
- **Radius 600 m** dipakai untuk mempersempit scope dari skala kota ke skala area stasiun, dihitung dengan *great circle distance* (haversine) karena data jaringan jalan pejalan kaki Chicago tidak tersedia. Untuk radius seratusan meter pendekatan ini dianggap cukup wajar, tapi tetap dicatat sebagai limitasi (bukan jarak jalan sesungguhnya).

### 2.2 Pembersihan Artefak Pencatatan Waktu
Ditemukan lonjakan tidak wajar di jam 00:00 dan 12:00 — hasil investigasi menunjukkan ini artefak pembulatan waktu oleh petugas pencatat (menit `:00` dominan), bukan pola kejahatan sungguhan. Baris dengan indikasi waktu dibulatkan dikeluarkan dari seluruh analisis selanjutnya, supaya risk_score merefleksikan pola waktu asli.

### 2.3 Grid Spasial: 3 Desimal (~110 m/sel)
Diuji kuantitatif 2 vs 3 desimal.
- 2 desimal (~1,1 km/sel) → satu sel nyaris menutupi seluruh area studi (radius 600 m), sehingga area rawan dan aman yang berdekatan tercampur — tidak *actionable* untuk Safe Route.
- 3 desimal (~110 m/sel) dipilih meski lebih sparse, karena granularitasnya setara skala blok jalan — yang memang dibutuhkan supaya Safe Route bisa membedakan satu ruas jalan dari sebelahnya. Konsekuensi sparsity ditangani eksplisit di dua tahap terpisah (lihat 2.6 & 2.7).

### 2.4 Cyclical Encoding (jam, hari, bulan)
Jam 23 dan jam 0 secara temporal berdekatan tapi secara numerik jauh (23 vs 0). Encoding sin/cos menjaga jarak antar-waktu tetap konsisten secara siklis, dibuktikan dengan mengecek jarak Euclidean pada bidang (sin, cos) antar jam yang berdekatan.

### 2.5 Safe Place Features — Fitur Terpisah, Bukan Bagian Formula Risk Score
- **Kriteria titik aman**: pos polisi, damkar, community centre, apotek, rumah sakit, minimarket / supermarket, dan seluruh stasiun CTA — dipilih karena selalu ada orang berjaga/berstaf yang bisa
  dimintai bantuan.
- **Radius 500 m** (beda dari radius subset 600 m) — 600 m mendefinisikan *area studi*, sedangkan 500 m mendekati jarak jalan kaki cepat untuk mencari bantuan dari titik pengguna berada.
- **Kenapa tidak masuk formula risk_score**: risk_score dimaksudkan merepresentasikan satu konstruk — seberapa berbahaya suatu lokasi-waktu berdasar pola kejahatan historis. Kedekatan ke safe place adalah konstruk berbeda (mitigasi/akses bantuan). Mencampur keduanya membuat risk_score sulit diinterpretasi (skor bisa turun karena daerah aman, atau kebetulan dekat kantor polisi padahal tetap rawan). Karena itu disimpan sebagai dua kolom fitur terpisah: `dist_to_nearest_safe_place`, `n_safe_places_within_radius` — supaya modul Safe Route bisa menggabungkan dua sinyal ini dengan logikanya sendiri.

### 2.6 Severity Scoring — 2 Level (Kategori + Modifier)
Baseline (5 kombinasi manual + 1 default) tidak scalable. Diganti pendekatan 2-level:
- **Level 1** — skor dasar per `Primary Type` (~30 kategori), ordinal berdasar keparahan (nyawa/tubuh > bersenjata/seksual > properti berat > properti ringan > administratif).
- **Level 2** — modifier dari kata kunci `Description`:
    - **Kelompok senjata** (eksklusif, ambil yang terberat): HANDGUN/FIREARM/GUN/KNIFE/DANGEROUS WEAPON.
    - **Kelompok konteks** (akumulatif, boleh lebih dari satu aktif sekaligus): AGGRAVATED, CHILD, FIRST DEGREE, DOMESTIC, dst — termasuk modifier negatif untuk SIMPLE, ATTEMPT, nilai kecil.
- Setiap modifier **diverifikasi aktif** di data (dihitung jumlah kemunculannya), bukan diasumsikan bekerja — termasuk perbaikan bug pencocokan tanda baca (`ARMED: HANDGUN` vs `ARMED - HANDGUN`) yang sempat membuat satu modifier tidak pernah aktif.

### 2.7 Temporal Decay — Half-Life 90 Hari, Label Window 180 Hari
- **Pemisahan jendela fitur vs label**: fitur prediktor (`hist_crime_count`, `crime_diversity`, dst) dihitung **hanya** dari kejadian sebelum titik potong (`df_hist`), sedangkan risk_score dibentuk **hanya** dari kejadian dalam 180 hari terakhir (`df_label`). Ini mencegah kebocoran target, tanpa pemisahan ini, fitur dan label dihitung dari himpunan kejadian yang sama sehingga model hanya menghafal tautologi, bukan belajar memprediksi.
- **Half-life 90 hari** (bukan 180 atau 45) dipilih dan **dipertahankan** dengan alasan:
    1. Dibaca berpasangan dengan label window 180 hari: kejadian di ujung jendela (hari ke-180) masih menyumbang bobot 25% (`0.5^(180/90)`) — proporsional. Half-life 45 hari akan menyusutkannya jadi ~6%, membuat 180 hari data yang dikumpulkan efektif tidak terpakai.
    2. Data per unit (sel × jam) sangat sparse (rata-rata 1–2 kejadian/unit); decay yang lebih agresif membuat risk_score makin rentan berubah drastis hanya karena satu-dua kejadian baru, bertentangan dengan tujuan stabilitas tier (lihat 2.9).
    3. Risk_score berperan sebagai peta risiko dasar untuk Safe Route & Visual Risk Indicator — butuh stabil, bukan sensitif real-time (sensitivitas kejadian terbaru itu domainnya Anonymous Reporting, di luar scope notebook ini).
    4. Seluruh validasi distribusi & tier yang sudah diperiksa dihasilkan dari nilai ini.

### 2.8 Penyusutan Empiris (Empirical Bayes Shrinkage)
Grid penuh (sel × 7 hari × 24 jam) dibentuk lewat perkalian kartesian — bukan `groupby` langsung — supaya kombinasi tanpa kejadian tetap punya baris bernilai nol (contoh negatif yang sah, bukan data hilang). Konsekuensinya, mayoritas unit sangat sparse. Ditangani dengan shrinkage: `base_value = alpha * obs_value + (1 - alpha) * cell_total * global_share`, di mana `alpha = cell_n / (cell_n + K_SHRINK)`, `K_SHRINK = 50` (jumlah kejadian saat sebuah sel mulai dianggap separuh bisa dipercaya sendiri). Sel dengan sedikit histori "meminjam" bentuk pola waktu dari profil kota, alih-alih mengikuti satu-dua kejadian kebetulan.

### 2.9 Spatial Smoothing — Gaussian Kernel via BallTree Haversine
- Diganti dari baseline (rata-rata 8 tetangga grid 3×3, jarak diestimasi dari indeks grid) ke kernel Gaussian berbasis jarak haversine sesungguhnya: `w = exp(-d² / (2σ²))`.
- Radius & sigma dipilih dari perbandingan kuantitatif beberapa kandidat, dinilai dari rasio simpangan baku sesudah/sebelum smoothing (semakin turun → semakin halus, tapi juga semakin kehilangan granularitas hotspot). **Kandidat terpilih: radius 0.4 km, sigma 0.15 km** (rata-rata 36 tetangga per sel).
- Bentuk agregasi memakai **rata-rata berbobot**, bukan jumlah berbobot — supaya sel yang kebetulan dikelilingi banyak tetangga tidak mendapat risk_raw yang lebih besar semata karena densitas tetangganya, bukan karena risikonya sungguh lebih tinggi.

### 2.10 Normalisasi — `log1p` + Min-Max, Dua Skala Berbeda Peran
`risk_raw` sangat *right-skewed*. Dibanding alternatif (peringkat persentil, clipping kuantil): `log1p` dipilih karena mempertahankan informasi *magnitude* (seberapa jauh jarak risiko antar sel) tanpa membuang informasi dari sel ekstrem. Karena kompresi log membuat skor tinggi jadi langka (median akan "terbaca" sangat aman kalau ditampilkan mentah), disimpan dua kolom dengan peran berbeda:
- `risk_score` — hasil `log1p` + min-max, dipakai sebagai target/label model.
- `risk_score_display` — peringkat persentil, tersebar merata 0–100, dipakai untuk ditampilkan ke pengguna (Visual Risk Indicator).

### 2.11 Tiga Tier Risk Indicator — Threshold Persentil 65 & 90
- Dibagi rata sepertiga-sepertiga (persentil 33/66) ditolak: buruk secara desain produk karena tier merah yang menyala di sepertiga peta akan cepat diabaikan pengguna (*alert fatigue*).
- Dipakai **persentil 65 & 90**, target proporsi ~65% Aman, 25% Waspada, 10% Rawan — sejalan dengan konvensi sistem peringatan publik yang membuat level tertinggi memang jarang menyala, dan mencerminkan kenyataan bahwa mayoritas area dekat stasiun transit relatif aman.
- Ambang batas **dibekukan** (disimpan sebagai konstanta `T_LOW`/`T_HIGH` di `model_config.csv`), bukan dihitung ulang tiap kali ada data baru — supaya warna sebuah lokasi tidak berubah semata karena lokasi lain membaik/memburuk.
- Divalidasi tiga hal: pemisahan skor antar tier tidak overlap, satu sel tidak "berkedip" antar tier terlalu sering sepanjang minggu, dan tier Rawan memang mengelompok di jam-jam yang masuk akal secara EDA (bukan tersebar acak).

### 2.12 `arrest_rate` Sengaja Tidak Disertakan
Bukan karena sistem tidak terhubung ke data kepolisian (seluruh dataset memang berasal dari kepolisian), tapi karena arah hubungannya ambigu — tingkat penangkapan tinggi bisa berarti penegakan hukum aktif (lebih aman) atau justru banyak kejahatan serius (lebih rawan). Karena tandanya tidak bisa ditentukan dari data ini, fitur ini dihilangkan supaya interpretasi modelm tidak menjadi lebih rumit.

---

## 3. Dataset Akhir

`features_labels.csv` / `.parquet` — **1.089.480 baris × 24 kolom** (grid unit lengkap: setiap sel × 7 hari × 24 jam), tanpa nilai kosong, tanpa duplikat kunci unit.

| Kolom | Kategori | Keterangan |
|---|---|---|
| `cell_id`, `lat_r`, `lon_r` | Identitas spasial | Grid 3 desimal (~110 m/sel) |
| `dow`, `hour`, `hour_sin/cos`, `dow_sin/cos`, `is_weekend` | Temporal | Cyclical encoding |
| `hist_crime_count`, `crime_diversity`, `violent_share`, `night_share`, `street_share` | Historis per sel | Dihitung dari `df_hist` (jendela fitur, terpisah dari label) |
| `seasonal_cv`, `peak_month_sin/cos` | Musiman | Dikoreksi bias jumlah tahun per bulan |
| `dominant_crime_type` | Interpretasi | Jenis kejahatan paling sering per sel |
| `dist_to_nearest_safe_place`, `n_safe_places_within_radius` | Safe Place | Fitur terpisah dari risk_score |
| `risk_score`, `risk_score_display`, `risk_tier` | Label | Target model / tampilan pengguna / tier 🟢🟡🔴 |

File pendukung lain: `safe_places.csv` (titik-titik aman untuk Safe Place Locator),
`model_config.csv` (konstanta pipeline: `T_LOW`, `T_HIGH`, `half_life_days`,
`label_window_days`, `cutoff`, `reference_date`, `radius_km`, `sigma_km`, `k_shrink`,
`grid_decimals`, `station_radius_m`, `safe_place_radius_m`) — bukan model terlatih, melainkan
manifest parameter supaya pipeline bisa direproduksi konsisten pada data baru.

---

## 4. Struktur Repo

```
.
├── README.md                  # dokumen ini
├── final_project.ipynb        # notebook utama: HO-1 (cleaning → EDA → FE → pseudo-labeling)
├── features_labels.csv        # dataset fitur + label (risk_score, risk_tier)
├── features_labels.parquet    # versi parquet, sama isinya, lebih ringkas
├── safe_places.csv            # titik-titik aman untuk Safe Place Locator
└── model_config.csv           # konstanta pipeline (T_LOW/T_HIGH, half-life, radius, dst.)
```