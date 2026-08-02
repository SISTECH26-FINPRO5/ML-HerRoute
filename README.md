# HerRoute: Safety-Tech App untuk Rute Aman di Jakarta

## Deskripsi Proyek

Proyek ini membangun pipeline end-to-end untuk memprediksi tingkat risiko keamanan (Risk Score) di sekitar stasiun transit, menggunakan dataset Chicago Crimes dan stasiun CTA (Chicago Transit Authority) sebagai analog struktural untuk stasiun MRT Jakarta, karena Chicago tidak memiliki MRT tetapi memiliki sistem transit rel sejenis (CTA 'L', elevated train). Proyek terdiri dari dua checkpoint yaitu pembentukan dataset dan label (Checkpoint 1), serta pembangunan model prediksi dan sistem continual learning (Checkpoint 2).

## Fitur Aplikasi

1. **Safe Route Recommendation & Risk Prediction**: rute teraman vs tercepat antar dua titik, dengan trade off jarak vs risiko.
2. **Visual Risk Indicator** (🟢/🟡/🔴): skor risiko suatu lokasi pada waktu tertentu.
3. **Safe Place Locator**: tempat aman terdekat (minimarket, apotek, pos polisi, rumah sakit, stasiun).
4. **Anonymous Reporting**: laporan insiden tanpa identitas pelapor.
5. **Emergency Feature**: Tombol SOS.
> Emergency Feature tidak masuk cakupan pipeline ML ini karena tidak memerlukan model.

---

## Ringkasan Proses End-to-End

### Checkpoint 1: Feature Engineering & Pseudo-Labeling

| Tahap | Deskripsi |
|-------|-----------|
| Setup dan pengumpulan data | Menggunakan dua sumber data, yaitu Chicago Crimes (data historis kejahatan) dan stasiun CTA dari City of Chicago Data Portal (data snapshot lokasi stasiun terkini). Hanya kolom yang relevan yang diambil dari masing-masing sumber untuk menjaga dataset tetap ringkas. |
| Subsetting data | Data dipersempit ke 3 tahun terakhir yang tersedia (2024 sampai 2026), dipilih berdasarkan rentang waktu, bukan area atau district, agar variasi spasial tetap utuh untuk dianalisis. |
| Audit dan pembersihan data | Dilakukan audit menyeluruh terhadap missing value, duplikasi, dan tipe data sebelum baris apa pun dibuang, supaya keputusan cleaning berbasis bukti. Baris tanpa koordinat atau tanggal valid dibuang, koordinat di luar bounding box Chicago disingkirkan, dan ditemukan artefak pencatatan waktu di jam 0 dan jam 12 yang kemudian dikeluarkan dari analisis. |
| Subset spasial berbasis radius stasiun | Scope data dipersempit dari skala kota menjadi radius jalan kaki 600 meter dari stasiun CTA terdekat, dihitung menggunakan great circle distance (haversine) melalui BallTree. |
| Exploratory Data Analysis (EDA) | Menganalisis pola temporal (jam, hari, bulan), pola spasial (hotspot kejahatan), interaksi hari x jam, serta perbandingan hari kerja vs akhir pekan, untuk membangun intuisi yang memandu feature engineering dan pseudo-labeling. |
| Feature engineering temporal | Jam dan hari dienkode secara siklikal menggunakan sin dan cos, supaya jarak antar waktu (misalnya jam 23:00 dan 00:00) direpresentasikan secara benar oleh model. |
| Feature engineering spasial | Koordinat dibulatkan menjadi grid sel berukuran sekitar 110 meter (3 desimal), dipilih setelah diuji secara kuantitatif terhadap alternatif 2 desimal (sekitar 1,1 kilometer per sel). |
| Safe Place Features | Titik-titik tempat aman (kantor polisi, pemadam kebakaran, rumah sakit, apotek, minimarket, dan lain-lain) diambil dari OpenStreetMap melalui Overpass API, lalu dihitung jarak ke tempat aman terdekat dan jumlah tempat aman dalam radius 500 meter per sel. Fitur ini sengaja dipisah dari formula risk_score karena merepresentasikan konstruk yang berbeda. |
| Pseudo-labeling untuk membentuk Risk Score | Dilakukan melalui empat tahap berurutan, severity scoring, temporal decay, agregasi spasial (spatial smoothing), dan normalisasi ke rentang 0 sampai 100. |
| Severity scoring dua level | Skor dasar per Primary Type dikombinasikan dengan modifier berbasis kata kunci di kolom Description (misalnya "ARMED", "AGGRAVATED"), dan setiap modifier diverifikasi benar-benar aktif di data, bukan sekadar diasumsikan bekerja. |
| Pemisahan jendela fitur dan label | Data dipecah menjadi jendela historis (df_hist, untuk membentuk fitur) dan jendela label (df_label, untuk membentuk risk_score), untuk mencegah kebocoran target akibat fitur dan label dihitung dari kejadian yang sama. |
| Pembentukan grid unit penuh dengan penyusutan empiris | Seluruh kombinasi sel, hari, dan jam dibentuk secara eksplisit (bukan hanya groupby), termasuk kombinasi yang tidak punya kejadian sama sekali, supaya model punya contoh negatif (area aman) untuk dipelajari. |
| Spatial smoothing dengan kernel Gaussian | Risiko satu sel dipengaruhi oleh sel-sel tetangga menggunakan bobot berbasis jarak haversine sesungguhnya (BallTree), bukan estimasi jarak grid seperti pada versi baseline. |
| Normalisasi menjadi Risk Score | Nilai risk_raw dikompresi dengan log1p sebelum di-min-max-kan ke rentang 0 sampai 100, untuk mengatasi distribusi yang sangat skewed. |
| Kategorisasi Risk Tier | Risk Score dikategorikan menjadi tiga tier (Aman, Waspada, Rawan) menggunakan ambang persentil 65 dan 90, bukan pembagian rata tiga bagian, untuk menghindari kelelahan peringatan pada indikator visual. |
| Audit akhir dan penyimpanan dataset | Dataset final divalidasi (grid lengkap, tidak ada nilai kosong, tidak ada kolom pembentuk label yang bocor sebagai fitur, risk_score berada pada rentang 0 sampai 100) sebelum disimpan. |
| Snapshot periodik untuk continual learning | Sebagai persiapan Checkpoint 2, dibangun dataset time series tambahan berisi rangkaian snapshot berurutan waktu (6 periode, masing-masing sekitar 30 hari), dengan parameter normalisasi dibekukan dari periode pertama agar perubahan risk_score antar waktu tetap terlihat, bukan direntangkan ulang tiap periode. |

### Checkpoint 2: Model Prediksi & Continual Learning

| Tahap | Deskripsi |
|-------|-----------|
| Load dataset dari Checkpoint 1 | Empat berkas dimuat, yaitu features_labels.csv (dataset utama), features_labels_timeseries.csv (dataset periodik), model_config.csv (ambang tier), dan safe_places.csv (titik tempat aman). |
| Baseline non-ML | Dibangun beberapa baseline sederhana sebagai patokan minimal, yaitu global mean, rata-rata per sel, rata-rata per hari-jam, serta kombinasi Sel+Jam, Sel+Hari, dan Sel+Weekend, lengkap dengan strategi fallback tiga tingkat. Kombinasi Sel+Jam terbukti menjadi baseline terkuat secara empiris. |
| Feature Vector Assembly | Ditambahkan frequency encoding dan target encoding untuk cell_id, keduanya dihitung hanya dari data training untuk mencegah data leakage. Fitur final dipilih berdasarkan kekuatan korelasinya terhadap risk_score, dan seluruh proses dibungkus dalam satu fungsi (assemble_features_v2) agar representasi fitur konsisten antara training dan serving. |
| Analisis linearitas | Perbedaan korelasi Pearson dan Spearman dicek untuk tiap fitur, untuk mengetahui indikasi hubungan non-linear sebelum memilih model. |
| Training model regresi | Tiga model dilatih dan dibandingkan, yaitu Linear Regression, Random Forest, dan Gradient Boosting. Ketiganya mengalahkan seluruh baseline, dengan Gradient Boosting sebagai model terbaik pada MAE, RMSE, dan R2. |
| Verifikasi asumsi model | Plot residual dan distribusi residual dicek untuk Linear Regression, menunjukkan model tidak bias secara sistematis meskipun ada sedikit ekor di sisi kanan. |
| Evaluasi model vs baseline | Perbandingan eksplisit dilakukan pada data test yang tidak pernah dilihat model selama training, termasuk analisis MAE per rentang risk_score aktual, yang mengungkap model paling lemah justru pada area risiko tinggi akibat ketimpangan jumlah data (data imbalance). |
| Simulasi continual learning | Data dibagi menjadi beberapa batch berdasarkan kolom period asli di dataset time series, bukan drift buatan, sehingga drift yang muncul benar-benar berasal dari pola alami data, dengan satu periode terakhir disisihkan sebagai holdout evaluasi. |
| Drift detection | Dikembangkan dua lapis, yaitu uji Kolmogorov-Smirnov (KS) dan Population Stability Index (PSI) untuk drift fitur, ditambah pengecekan concept drift melalui perubahan korelasi antara crime_count dan risk_score. |
| Checkpoint retraining dan model versioning | Tiga strategi retraining dibandingkan, yaitu cumulative, sliding window, dan fine-tuning (warm_start), dengan mekanisme champion vs challenger yang mempromosikan model baru hanya jika memenuhi margin perbaikan MAE dan tidak menurunkan R2. |
| Model registry | Setiap keputusan retraining, baik yang dipromosikan maupun tidak, dicatat lengkap dengan dataset fingerprint, hyperparameter, dan metrik evaluasi, disimpan dalam format tabular (CSV) agar mudah difilter dan dibandingkan lintas strategi. |
| Integrasi ke fitur aplikasi | Model dan data yang sudah dibangun dihubungkan ke empat fitur aplikasi, yaitu Visual Risk Indicator, Safe Place Locator, Safe Route Recommendation, dan Risk Prediction, termasuk mekanisme remap koordinat dari Jakarta ke rentang koordinat Chicago yang dikenali model, dengan menjaga posisi relatif antar titik. |
| Export artefak final | Model, konteks encoding, lookup histori sel, dan bounding box remap koordinat disimpan sebagai file joblib untuk keperluan serving. |

---

## Justifikasi Keputusan-Keputusan Penting

### CHECKPOINT 1: Feature Engineering & Pseudo-Labeling

#### 1. Setup & Pengumpulan Data

##### 1.1 Dataset Chicago Crimes
- Kolom yang diambil dibatasi pada yang relevan saja, yaitu `Date`, `Primary Type`, `Description`, `Latitude`, `Longitude`, `Arrest`, `Location Description`, supaya dataset tetap ringkas.

##### 1.2 Dataset Stasiun CTA
- Chicago tidak punya MRT, sehingga CTA 'L' (elevated train) dipakai sebagai analog struktural untuk area sekitar stasiun MRT.
- Dataset ini bersifat snapshot kondisi terkini, bukan data historis per tahun seperti Chicago Crimes, karena satu baris merepresentasikan satu titik stop atau entrance stasiun, bukan satu kejadian.
- Kolom dibatasi ke `stop_id`, `station_name`, dan `location`. Kolom lain seperti flag jalur (`red`, `blue`, `g`, dst) dan metadata region internal Socrata tidak relevan untuk kebutuhan perhitungan jarak spasial.

##### 1.3 Subset 3 Tahun Terakhir (2024 sampai 2026)
- **Rentang tahun dipilih, bukan area/district**, supaya variasi spasial (hotspot vs area aman) tetap utuh untuk dianalisis. Ini krusial karena Risk Score sangat bergantung pada sinyal spasial yang kaya. Membatasi ke satu area/district justru menghilangkan variasi itu sebelum sempat dianalisis.
- **3 tahun** dipilih supaya pola musiman (`month`, `day-of-week`) tidak bias ke satu musim/periode tertentu, sekaligus menggunakan data paling terkini yang tersedia supaya relevan dengan kondisi kejahatan Chicago saat ini.

#### 2. Pembersihan Data

##### 2.1 Audit Kualitas Data Mentah
- Sebelum membuang baris apa pun, kondisi data mentah diperiksa lebih dulu supaya keputusan cleaning didasarkan pada bukti, bukan asumsi. Tiga hal yang dicek:
  1. **Missing value per kolom.** Koordinat yang hilang membuat baris tidak bisa dipetakan ke grid spasial sama sekali, sehingga baris tersebut memang harus dibuang. Sebaliknya, kolom deskriptif yang bolong tidak selalu harus mematikan barisnya.
  2. **Duplikasi.** Chicago Crimes adalah data operasional yang di-update berkala, sehingga satu `ID` bisa muncul lebih dari sekali ketika ada koreksi laporan. Duplikat semacam ini akan membuat satu insiden terhitung ganda dan menggelembungkan Risk Score di lokasi tersebut.
  3. **Konsistensi tipe data.** Memastikan kolom numerik benar-benar terbaca sebagai numerik dan tidak jatuh menjadi object akibat karakter asing.

##### 2.2 Pembersihan Data
- Baris tanpa `Latitude`/`Longitude` atau tanggal dibuang, karena tanpa koordinat sebuah kejadian tidak bisa dipetakan ke grid spasial.
- Koordinat di luar bounding box Chicago disingkirkan untuk menyingkirkan koordinat rusak (misalnya 0,0).

##### 2.3 Subset Spasial: Radius 600 m dari Stasiun CTA
- Jarak dihitung memakai great circle distance (haversine) melalui BallTree, bukan jarak jaringan jalan sebenarnya, karena data jaringan pejalan kaki Chicago tidak tersedia. Untuk radius sekecil 600 meter, pendekatan ini dianggap cukup wajar, namun tetap dicatat sebagai limitasi.
- **Interpretasi hasil subset:**
  1. **Penyusutan data** menegaskan scope analisis sudah berpindah dari skala kota ke skala area stasiun.
  2. **Sebaran jarak ke stasiun terdekat** yang cukup merata antara batas bawah dan batas radius menandakan tidak ada bias sistematis. Jika kejadian menumpuk hanya di depan stasiun atau di tepi radius, sinyal spasialnya akan timpang.
  3. **Konsistensi dengan radius safe place.** Radius subset 600 meter sengaja dibedakan dari radius safe place 500 meter. Yang pertama menentukan area studi, sedangkan yang kedua merepresentasikan jarak jalan kaki cepat untuk mencari bantuan dari titik pengguna berada.
- **Limitasi.** Karena memakai great circle distance dan bukan jarak jaringan jalan sebenarnya, jarak tempuh sesungguhnya akan lebih panjang, terutama pada blok yang terpotong rel, sungai, atau jalan besar.

#### 3. Exploratory Data Analysis (EDA)
- Tujuan EDA adalah membangun intuisi tentang kapan dan di mana kejahatan paling sering terjadi, yang nanti memandu keputusan feature engineering dan pseudo-labeling.
- **Pola jam akhir pekan vs hari kerja** dipisah untuk melihat apakah jam sibuk kejahatan bergeser tergantung jenis harinya.
- **Pola waktu per jenis kejahatan** dibandingkan untuk melihat apakah pola waktunya cukup berbeda antar 5 jenis kejahatan terbanyak.
- **Heatmap interaksi hari x jam** dipakai untuk melihat jumlah kejadian tiap kombinasi hari dan jam.

##### 3.7 Artefak Pencatatan Waktu di Jam 0 dan 12
- **Kecurigaan awal:** lonjakan tajam di jam 0 dan 12 mencurigakan karena jika waktu kejadian benar-benar natural, seharusnya tidak ada satu jam yang jauh lebih tinggi dibanding jam-jam di sekitarnya secara konsisten.
- **Dugaan:** ini adalah artefak pencatatan, saat waktu kejadian tidak diketahui pasti oleh petugas atau pelapor, sistem kemungkinan mencatat waktu yang dibulatkan ke jam penuh (`00:00` atau `12:00`) sebagai default.
- **Cara pembuktian:** distribusi menit pada baris berjam tepat 0 dan 12 dicek. Jika waktu kejadian presisi dan acak, distribusi menit seharusnya tersebar relatif merata. Hasilnya, menit `:00` muncul jauh lebih dominan dibanding menit lain, mengonfirmasi dugaan bahwa waktu tersebut memang dibulatkan, bukan dicatat secara presisi.
- **Verifikasi dampak:** baris dengan flag `time_likely_imprecise` (jam 0 atau 12, dengan menit dan detik persis 0, supaya tidak salah membuang kejadian yang memang betul terjadi tepat di jam tersebut dengan presisi lain) dibandingkan pola jamnya sebelum vs sesudah dikeluarkan. Sebelum exclude, jam 0 dan 12 melonjak tidak wajar; sesudah exclude, kurva jam kembali mulus.
- **Keputusan akhir:** baris-baris tersebut dikeluarkan dari seluruh analisis selanjutnya, karena jika dibiarkan, Risk Score bisa salah menganggap jam 0 dan 12 sebagai jam yang jauh lebih berisiko, padahal itu hanya kebiasaan pencatatan petugas.

#### 4. Feature Engineering

##### 4.1 Fitur Temporal: Cyclical Encoding
- Waktu bersifat siklikal, jam `23:00` sebenarnya berdekatan dengan `00:00`, tetapi jika dipakai angka mentah model akan menganggapnya berjauhan (selisih 23). Solusinya, tiap nilai siklikal dipetakan ke lingkaran dengan `sin` dan `cos`, sehingga jarak antar waktu direpresentasikan secara benar oleh model.

##### 4.2 Fitur Spasial: Grid Aggregation
**1. Ukuran sel: 3 desimal (sekitar 110 meter per sel).**
- Diuji secara kuantitatif 2 versus 3 desimal. Resolusi 2 desimal setara sekitar 1,1 kilometer per sel, dan pada konteks radius 600 meter dari stasiun, satu sel sebesar itu hampir menutupi seluruh area studi, sehingga Risk Score tidak akan actionable untuk membedakan blok jalan yang lebih aman dari yang lain. Resolusi 3 desimal memberi granularitas setara skala blok jalan yang dibutuhkan agar Safe Route Recommendation bisa membedakan satu ruas jalan dari ruas di sebelahnya.

**2. Penanganan sparsity pada dua sumbu sekaligus.**
- Konsekuensi grid halus adalah mayoritas kombinasi sel, hari, dan jam tidak punya kejadian tercatat. Ini ditangani lewat dua mekanisme:
  - Sumbu temporal: penyusutan empiris terhadap profil jam global, supaya sel dengan sedikit kejadian meminjam bentuk pola waktu dari data kota.
  - Sumbu spasial: penghalusan berbasis jarak dengan kernel Gaussian, supaya sel meminjam sinyal dari tetangga terdekatnya secara proporsional terhadap jarak.

**3. Fitur turunan dihitung di level sel, bukan level unit (sel x hari x jam).**
- Percobaan menghitungnya di level sel x hari x jam menghasilkan fitur yang praktis konstan, karena rata-rata hanya ada satu sampai dua kejadian per unit. Secara domain pun besaran ini memang merupakan karakteristik tempat, bukan karakteristik yang berubah tiap jam.

**4. Fitur temporal tambahan.**
- `is_weekend` ditambahkan sebagai flag biner karena EDA menunjukkan bentuk pola jam hari kerja dan akhir pekan cukup berbeda. Model perlu sinyal eksplisit ini karena `dow_sin` dan `dow_cos` memperlakukan seluruh hari sebagai kontinum siklikal tanpa membedakan hari kerja dari hari libur.
- `month` tidak dijadikan dimensi baru pada unit analisis, karena akan memecah data menjadi dua belas kali lebih sparse tanpa data pendukung yang memadai. Informasi musiman tetap dipertahankan lewat fitur `seasonal_cv`, `peak_month_sin`, dan `peak_month_cos` di level sel.

##### 4.2b Perbandingan Ukuran Grid (Keputusan: 3 Desimal)
- Resolusi 2 desimal setara sekitar 1,1 kilometer per sel. Pada area studi yang hanya berjarak 600 meter dari stasiun, satu sel sebesar itu praktis menelan seluruh area, sehingga area rawan dan area aman yang berdekatan tercampur dalam satu nilai.
- Resolusi 3 desimal memberi granularitas setara skala blok jalan, yang dibutuhkan agar Safe Route Recommendation bisa membedakan satu ruas jalan dari ruas di sebelahnya.
- Sparsity yang menjadi konsekuensinya ditangani secara eksplisit pada dua tahap berikutnya, yaitu penyusutan empiris pada sumbu temporal dan penghalusan kernel pada sumbu spasial.

##### 4.2c Safe Place Features
- Data titik tempat aman diambil dari OpenStreetMap melalui Overpass API, dibatasi pada bounding box area studi, karena fitur Safe Place Locator membutuhkan titik-titik tempat pengguna dapat mencari bantuan ketika merasa terancam.
- **Catatan teknis kategori OSM di dua tag berbeda.** Pos polisi, pemadam kebakaran, community centre, dan apotek memakai tag `amenity`, sedangkan minimarket dan supermarket memakai tag `shop`. Membaca `amenity` saja membuat seluruh node toko masuk dengan kategori kosong, dan karena `value_counts()` mengabaikan nilai kosong secara diam-diam, node tersebut lenyap dari ringkasan tanpa menimbulkan error apa pun. Karena itu ekstraksi kategori membaca kedua tag secara berurutan (amenity lalu shop), dan hasilnya diverifikasi lewat `assert` bahwa tidak ada node tanpa kategori dan jumlah `value_counts()` sama dengan jumlah baris, supaya kegagalan semacam ini langsung menghentikan notebook alih-alih lolos tanpa terdeteksi.
- **Kriteria titik aman:** tempat yang secara wajar selalu ada orang yang bisa dimintai bantuan ketika terjadi insiden. Kategori yang masuk beserta alasannya:
  - `amenity=police`: pos keamanan, tujuan bantuan paling langsung.
  - `amenity=fire_station`: berstaf 24 jam dan terlatih menangani situasi darurat.
  - `amenity=community_centre`: ruang publik berpenghuni dengan pengelola.
  - `amenity=pharmacy` dan `amenity=hospital`: jam operasional panjang, terang, dan berstaf.
  - `shop=convenience`, `shop=supermarket`: banyak yang buka hingga larut, ramai, dan ada kasir.
  - Stasiun CTA: setiap stasiun transit punya loket dan petugas.
- **Interpretasi hasil safe place:**
  1. Median `dist_to_nearest_safe_place` yang rendah berarti akses bantuan di area sekitar stasiun relatif merata, bukan terpusat di beberapa titik saja.
  2. Sebaran `n_safe_places_within_radius` dengan kuartil yang berjauhan menandakan fitur ini memang membedakan sel yang dekat infrastruktur bantuan dari yang jauh. Kalau mayoritas sel bernilai sama, fitur ini tidak akan berguna bagi model.
  3. **Uji redundansi.** Karena seluruh data sudah dibatasi radius 600 meter dari stasiun, dan stasiun ikut dihitung sebagai safe place, ada risiko fitur ini hanya mengulang informasi jarak ke stasiun. Persentase sel yang safe place terdekatnya adalah stasiun dipakai sebagai pemeriksanya.
- **Limitasi tagging OSM.** Kelengkapan OpenStreetMap bergantung pada kontribusi sukarelawan dan tidak seragam antar wilayah, sehingga jumlah titik di sini adalah batas bawah, bukan hitungan lengkap.

##### Kenapa Safe Place Tidak Masuk Formula risk_score
- Kedekatan ke tempat aman tidak dimasukkan ke formula `risk_score`, melainkan jadi dua kolom fitur terpisah. `risk_score` merepresentasikan satu konstruk, yaitu seberapa berbahaya suatu lokasi-waktu berdasarkan pola kejahatan historis, sedangkan kedekatan ke tempat aman adalah konstruk berbeda soal mitigasi dan akses bantuan. Mencampur keduanya ke satu angka membuat `risk_score` sulit diinterpretasi, karena skor bisa turun karena daerah memang lebih aman, atau karena kebetulan dekat kantor polisi padahal tetap rawan.
- Dengan dipisah, modul Safe Route Recommendation bisa menggabungkan dua sinyal ini dengan logikanya sendiri, misalnya meminimalkan `risk_score` sepanjang rute sambil memberi bobot tambahan ke rute yang melewati banyak safe place, jauh lebih fleksibel dibanding kalau sudah dicampur duluan di level label.

#### 5. Pseudo-Labeling: Membentuk Risk Score

##### 5.1 Severity Scoring
**1. Cakupan kombinasi: pendekatan dua level.**
- Alih-alih memperluas daftar kombinasi Primary Type + Description satu per satu (yang tidak akan pernah mencakup semua variasi Description), digunakan pendekatan dua level yang otomatis mencakup seluruh data: Level 1 skor dasar per `Primary Type` (sekitar 30 kategori), Level 2 modifier dari kata kunci di `Description` (misalnya "AGGRAVATED", "ARMED", "SIMPLE"). Pendekatan ini menghindari perlunya menghafal ratusan kombinasi secara manual, sambil tetap sensitif terhadap detail dalam Description.

**2. Strategi fallback per Primary Type.**
- Fallback ini hanya berlaku jika Primary Type-nya sendiri tidak dikenal (kasus langka), jauh lebih jarang terjadi dibanding baseline yang defaultnya berlaku untuk 95%+ kombinasi yang tidak tercakup.

**3. Justifikasi skala nilai severity.**
- Skor dasar disusun berdasarkan tingkat bahaya obyektif terhadap keselamatan manusia:
  - Kejahatan terhadap nyawa/tubuh (Homicide, Sexual Assault) diberi skor tertinggi (90 sampai 98), karena berdampak langsung dan permanen terhadap keselamatan/nyawa korban.
  - Kejahatan bersenjata/kekerasan (Robbery, Weapons Violation) diberi skor tinggi (60 sampai 82), karena melibatkan ancaman senjata atau potensi cedera fisik serius meski tidak selalu berakibat fatal.
  - Kejahatan properti (Theft, Burglary, Criminal Damage) diberi skor menengah-rendah (12 sampai 45), karena umumnya tidak melibatkan kontak fisik langsung dengan korban.
  - Pelanggaran administratif (Liquor Law, Gambling) diberi skor terendah (10 sampai 12), karena dampaknya minim terhadap keselamatan publik secara langsung.
  - Urutan ordinal ini konsisten dengan logika penilaian keparahan pada kerangka kerja keamanan publik (misalnya FBI UCR yang membedakan Part I/kejahatan serius vs Part II/pelanggaran ringan), meski nilai numerik pastinya tetap keputusan sendiri yang bersifat ilustratif.

**4. Verifikasi aturan, bukan asumsi.**
- Skema berbasis keyword punya satu mode kegagalan yang berbahaya, yaitu gagal tanpa menimbulkan error. Kalau format teks di data berbeda sedikit saja dari kunci yang ditulis, aturan tersebut tidak pernah aktif, skor tetap terbentuk, dan tidak ada tanda apa pun bahwa ada yang salah.
- Perbaikannya adalah mencocokkan nama senjata secara langsung, sehingga tidak lagi bergantung pada variasi tanda baca, dan ditambahkan cell verifikasi yang menghitung berapa baris yang benar-benar tersentuh oleh setiap modifier. Modifier dengan nol kemunculan langsung ditandai sebagai tidak aktif.
- Modifier `THREAT` dengan nilai negatif dihapus, karena tidak ada dasar yang bisa dipertanggungjawabkan untuk menyatakan bahwa adanya unsur ancaman justru menurunkan tingkat keparahan sebuah insiden.
- Modifier dipisah menjadi dua kelompok dengan aturan berbeda. Modifier senjata bersifat eksklusif karena satu insiden hanya melibatkan satu jenis senjata terberat, sedangkan modifier konteks bersifat akumulatif karena beberapa keadaan bisa berlaku bersamaan. Pemisahan ini menghilangkan ambiguitas `break` pada versi sebelumnya, yang perilakunya bergantung pada urutan penulisan daftar.

**Hasil akhir Severity Table:**
- Cakupan penuh: persentase baris yang jatuh ke fallback turun drastis dibanding baseline yang memberikan satu nilai default seragam untuk mayoritas kombinasi.
- Ranking konsisten dengan intuisi keamanan publik: kejahatan terhadap nyawa dan tubuh berada di puncak, kekerasan sedang serta properti berat di tengah, dan pelanggaran administratif di dasar. Urutan ini diperiksa untuk memastikan tidak ada anomali atau pembalikan yang tidak masuk akal.
- Seluruh modifier terbukti aktif, dipastikan lewat cell verifikasi bahwa tidak ada aturan yang gagal diam-diam akibat ketidakcocokan format teks.

##### 5.2 Temporal Decay: Recency

**Masalah pertama: label dan fitur dihitung dari kejadian yang sama.**
- Pada versi sebelumnya, `crime_count` dan seluruh fitur agregat dihitung dari himpunan kejadian yang persis sama dengan yang membentuk `risk_score`. Model yang dilatih di atas dataset seperti itu akan mencapai akurasi sangat tinggi, tetapi bukan karena berhasil belajar apa pun, melainkan karena input dan targetnya adalah dua cara menghitung angka yang sama. Ini tautologi, bukan prediksi, dan merupakan bentuk kebocoran target yang paling umum.
- **Solusi:** sumbu waktu dipisah menjadi jendela historis (dari awal subset sampai titik potong, dipakai untuk seluruh fitur prediktor) dan jendela label (180 hari terakhir, dipakai hanya untuk perhitungan `risk_score`). Dengan pembagian ini, pertanyaan yang dijawab dataset berubah menjadi pertanyaan yang benar secara operasional, yaitu berdasarkan pola historis suatu lokasi dan waktu, seberapa berisiko lokasi itu sekarang. Tidak ada satu kejadian pun yang muncul di kedua sisi.

**Masalah kedua: bentuk dan parameter peluruhan.**
- Peluruhan linear pada baseline membuat kejadian tertua berbobot persis nol, seolah-olah sama sekali tidak relevan, dan titik nol itu bergeser mengikuti ukuran subset, bukan alasan substantif. Karena itu dipakai peluruhan eksponensial berbasis half-life, yang turun mulus, tidak pernah benar-benar nol, dan parameternya punya arti yang bisa langsung dijelaskan, yaitu berapa lama sebuah kejadian kehilangan separuh relevansinya.
- **Parameter yang dipilih: jendela label 180 hari, half-life 90 hari.** Ini revisi dari versi sebelumnya yang memakai half-life 45 hari di atas subset tiga tahun, yang tidak koheren karena total bobot seluruh data hanya setara sekitar 65 hari data penuh (lebih dari 90 persen kejadian yang sudah susah payah dikumpulkan praktis tidak berkontribusi apa pun), dan justifikasi memilih subset tiga tahun agar pola musiman tidak bias pun menjadi batal dengan sendirinya karena pola musiman sudah terhapus oleh peluruhan sebelum sempat masuk ke label.
  - Jendela label 180 hari cukup panjang untuk mengumpulkan kejadian yang memadai bagi sinyal spasial yang stabil, dan cukup pendek untuk tetap menggambarkan kondisi terkini.
  - Half-life 90 hari di dalam jendela tersebut membuat kejadian bulan terakhir jelas lebih berbobot dibanding kejadian enam bulan lalu, tanpa membuat label bergantung pada segelintir kejadian minggu terakhir saja.
  - Sisa data historis tetap dipakai penuh tanpa peluruhan sebagai basis fitur, karena karakter jangka panjang sebuah lokasi (termasuk pola musimannya) memang butuh rentang panjang untuk diestimasi dengan stabil.
  - Dengan kata lain, peluruhan cepat dipakai untuk menjawab seberapa berisiko sekarang, sedangkan rentang panjang dipakai untuk menjawab lokasi seperti apa ini, karena kedua pertanyaan itu menuntut horizon waktu yang berbeda.

**Koreksi bias musiman pada peak_month.**
- Subset mencakup dua tahun penuh ditambah tahun berjalan yang baru terisi sebagian, sehingga bulan-bulan awal tahun memiliki lebih banyak tahun data dibanding bulan-bulan akhir tahun. Menghitung bulan puncak langsung dari jumlah mentah akan menghasilkan bias sistematis ke bulan awal, sehingga `peak_month` justru merekam artefak pengumpulan data alih-alih pola musiman sesungguhnya. Karena itu jumlah kejadian per bulan dinormalisasi dulu terhadap banyaknya tahun yang tersedia untuk bulan tersebut, sebelum bulan puncak dan koefisien variasi dihitung.

**`arrest_rate` tidak disertakan.**
- Bukan karena sistem tidak terhubung ke data kepolisian, sebab seluruh dataset ini justru berasal dari kepolisian. Alasan sebenarnya adalah arah hubungannya ambigu: tingkat penangkapan yang tinggi bisa berarti penegakan hukum aktif sehingga lokasi lebih aman, tetapi bisa juga berarti banyak kejahatan serius memang terjadi di sana. Karena tandanya tidak dapat ditentukan dari data ini, memasukkannya hanya akan mempersulit interpretasi model.

##### 5.2b Tabel Unit Analisis Lengkap

**Kenapa `groupby` saja tidak cukup.**
- `groupby` hanya mengembalikan kombinasi yang benar-benar muncul di data, sehingga tabel yang dihasilkan hanya berisi lokasi dan waktu yang ada kejahatannya. Ruang unit yang sesungguhnya adalah seluruh kombinasi sel dikali 7 hari dikali 24 jam, dan kombinasi yang teramati hanya sebagian kecil darinya. Akibatnya:
  - Kolom jumlah kejadian tidak pernah bernilai nol, karena baris nol memang tidak pernah dibuat.
  - Tier "aman" berubah makna menjadi sepertiga paling ringan di antara lokasi dan waktu yang tetap ada kejahatannya, bukan benar-benar aman.
  - Model tidak punya satu pun contoh negatif untuk dipelajari, sehingga tidak akan pernah bisa menyimpulkan bahwa suatu lokasi pada jam tertentu aman.
  - Aplikasi harus melayani pertanyaan untuk sembarang lokasi dan jam, sehingga mayoritas pertanyaan pengguna jatuh ke unit yang tidak pernah ada di data latih.
- **Perbaikan:** grid penuh dibangun lebih dulu lewat perkalian kartesian, lalu hasil agregasi ditempelkan ke atasnya, dan kombinasi yang tidak teramati diisi nol. Nol di sini adalah informasi yang sah (tidak ada kejadian tercatat), bukan data yang hilang.

**Penanganan sparsity dengan penyusutan empiris (empirical Bayes shrinkage).**
- Grid penuh memunculkan konsekuensi baru: sebagian besar unit hanya berisi nol, dan unit yang terisi biasanya hanya memuat satu atau dua kejadian, sehingga menghitung profil per jam langsung untuk tiap sel akan menghasilkan angka yang lebih banyak mencerminkan kebetulan daripada pola nyata.
- Solusinya menggabungkan dua sumber informasi: nilai yang benar-benar teramati di sel tersebut (`observed`), dan profil temporal global (`share`) yang punya basis data sangat besar sehingga bentuknya halus dan dapat dipercaya, dikombinasikan lewat bobot kepercayaan $\alpha_c = n_c / (n_c + K)$ yang bergerak mengikuti jumlah kejadian di sel.
- Perilakunya persis seperti yang diinginkan: sel dengan banyak kejadian punya alpha mendekati satu sehingga pola temporal khasnya dipertahankan, sedangkan sel dengan sedikit kejadian punya alpha mendekati nol sehingga profilnya meminjam bentuk dari pola global. Tidak ada ambang batas keras yang perlu ditentukan, karena peralihannya berlangsung mulus.
- Pendekatan ini sekaligus menjawab kelemahan yang sebelumnya hanya diserahkan pada penghalusan spasial, sehingga sparsity kini ditangani pada dua sumbu sekaligus: temporal lewat penyusutan di sini, spasial lewat kernel pada bagian berikutnya.

**Tabel unit versi baseline (pembanding).**
- Dibentuk dengan `groupby` langsung tanpa grid penuh dan tanpa penyusutan empiris, hanya disimpan sebagai pembanding untuk memperlihatkan perbedaan hasil. `crime_count` di sini minimalnya selalu satu, karena `groupby` tidak pernah menghasilkan baris untuk kombinasi yang tidak ada kejadiannya, tepatnya masalah yang diperbaiki oleh grid penuh.

##### 5.2c Fitur Prediktor dari Jendela Historis
- Seluruh fitur dihitung hanya dari `df_hist` (kejadian sebelum titik potong), tidak ada satu pun kejadian di jendela label yang menyentuh sisi fitur. Inilah yang membuat dataset ini benar-benar bisa disebut dataset prediksi, bukan sekadar dataset perhitungan ulang.
- **Level agregasi: per sel, bukan per sel x hari x jam.** Percobaan menghitung keragaman kejahatan dan proporsi penangkapan di level sel x hari x jam menghasilkan fitur yang praktis konstan (keragaman jenis kejahatan hampir selalu bernilai satu, proporsi penangkapan runtuh menjadi biner nol atau satu), karena rata-rata hanya ada satu sampai dua kejadian per unit. Secara domain pun kedua besaran ini memang merupakan karakteristik tempat, bukan sifat yang berbeda antara pukul sepuluh dan pukul sebelas.
- **Fitur yang dibentuk dan alasan relevansinya:**
  - `hist_crime_count`: basis intensitas jangka panjang lokasi.
  - `crime_diversity`: lokasi dengan banyak ragam kejahatan lebih sulit diprediksi dibanding lokasi yang didominasi satu jenis ringan berulang.
  - `violent_share`: membedakan lokasi rawan pencurian properti dari lokasi rawan kekerasan terhadap orang, dan yang kedua jauh lebih relevan untuk keselamatan pejalan kaki.
  - `night_share`: menandai lokasi yang risikonya terkonsentrasi di malam hari (22.00 sampai 04.00).
  - `street_share`: memakai `Location Description`, membedakan risiko yang dialami pejalan kaki dari risiko di dalam bangunan privat.
  - `seasonal_cv`: menandai lokasi yang polanya musiman tajam versus rata sepanjang tahun.
  - `peak_month_sin`, `peak_month_cos`: menyimpan informasi kapan lokasi biasanya paling rawan tanpa memecah unit analisis.
  - `dominant_crime_type`: menjawab pertanyaan pengguna tentang alasan sebuah titik ditandai merah.

##### 5.3 Spatial Aggregation: Proximity

**1. Bobot berdasarkan jarak: kernel Gaussian.**
- Diganti dari bobot rata-rata sama (baseline) menjadi kernel Gaussian berdasarkan jarak haversine sesungguhnya, dihitung pakai `sklearn.neighbors.BallTree` dengan metric `haversine`, bukan estimasi jarak dari indeks grid seperti baseline. Sel yang lebih dekat otomatis mendapat bobot lebih besar, dan bobot turun mulus (bukan seragam lalu terpotong tiba-tiba di batas radius).

**2. Radius pengaruh: `RADIUS_KM = 0.4` km, `SIGMA_KM = 0.15`.**
- Diganti dari kaku 1 sel (grid 3x3) karena sejak grid diubah ke 3 desimal (sekitar 110 m per sel), radius 3x3 hanya menjangkau sekitar 330 meter, terlalu sempit untuk merepresentasikan "lingkungan" yang bermakna secara risiko kejahatan. Radius baru dipilih sebagai skala neighborhood yang wajar, dengan sigma diatur supaya pengaruh signifikan (w > 0.05) terkonsentrasi di sekitar 1 sampai 2 blok kota, sekaligus mengatasi sparsity data pada grid halus karena sel dengan sedikit kejadian dapat meminjam sinyal dari lebih banyak sel tetangga dibanding radius 3x3 yang sempit.

**3. Bentuk agregasi: rata-rata berbobot, bukan jumlah berbobot.**
- `base_value` sudah merupakan hasil penjumlahan severity x decay dari seluruh kejadian di satu sel, sehingga kalau antar-tetangga dijumlahkan lagi, sel yang kebetulan dikelilingi banyak tetangga akan mendapat `risk_raw` yang jauh lebih besar semata karena densitas tetangganya tinggi, bukan karena risikonya sungguh lebih tinggi. Rata-rata berbobot menjaga skala `risk_raw` tetap sebanding antar sel terlepas dari berapa banyak tetangga yang ada di sekitarnya.

**4. Koreksi penting pada penyebut rata-rata berbobot.**
- Versi sebelumnya menyusun vektor penanda hanya untuk sel yang punya kejadian pada kombinasi hari dan jam yang sedang diproses, sehingga penyebut rata-rata berbobot hanya menjumlahkan tetangga yang kebetulan ada kejadiannya, sementara tetangga yang sepi diperlakukan seolah tidak ada. Efeknya adalah pelebihan risiko yang sistematis, paling parah justru pada jam-jam sepi (dini hari), yaitu jam paling penting bagi aplikasi keselamatan.
- **Perbaikan:** penyebut dihitung dari seluruh sel tetangga tanpa kecuali, sehingga tetangga yang tidak ada kejadiannya ikut menekan nilai rata-rata ke bawah sebagaimana mestinya. Karena tabel unit kini berbentuk grid penuh, penyebut ini bahkan konstan dan cukup dihitung sekali di luar perulangan.

**5. Radius dipilih lewat perbandingan, bukan ditebak.**
- Radius diuji pada beberapa nilai dan dinilai dari rasio simpangan baku sesudah terhadap sebelum penghalusan. Rasio yang terlalu kecil menandakan variasi lokal ikut terhapus, sehingga peta risiko menjadi rata dan kehilangan gunanya untuk membedakan rute. Rasio yang terlalu besar menandakan penghalusan nyaris tidak bekerja, sehingga sparsity spasial tidak tertangani.

##### 5.4 Normalisasi menjadi Risk Score 0 sampai 100

**Masalah min-max langsung.**
- Distribusi `risk_raw` sangat timpang ke kanan (mayoritas sel bernilai kecil, segelintir sel bernilai jauh lebih besar). Kalau nilai itu langsung di-min-max-kan, sel-sel ekstrem akan menekan hampir seluruh sel lain ke rentang skor yang sangat sempit di ujung bawah, sehingga risiko rendah dan risiko sedang menjadi sulit dibedakan.

**Solusi: `log1p` sebelum min-max.**
- Nilai dikompresi dulu dengan `log1p`, yang menstabilkan varians tanpa mengubah urutan risiko, baru kemudian di-min-max-kan ke rentang 0 sampai 100. `log1p` dipilih dibanding dua alternatif lain:
  - Dibanding peringkat persentil, `log1p` mempertahankan informasi magnitude (seberapa jauh jarak risiko antar sel), bukan sekadar urutannya. Ini penting karena Risk Score dimaksudkan merepresentasikan besaran risiko.
  - Dibanding clipping kuantil atas, `log1p` tidak membuang informasi apa pun dari sel ekstrem dan hanya mengompresnya secara proporsional. Clipping akan menyamakan seluruh sel di atas kuantil tertentu, sehingga menghilangkan granularitas di area paling berisiko, padahal area itulah yang paling penting untuk dibedakan.

**Dua skala untuk dua peran yang berbeda.**
- Kompresi logaritmik punya konsekuensi yang perlu diantisipasi: karena ekor atas dipadatkan, sebagian besar unit akan menempati separuh bawah rentang 0 sampai 100, dan skor tinggi menjadi sangat jarang muncul. Untuk model hal ini tidak masalah, tetapi bagi pengguna aplikasi skor seperti ini menyesatkan, karena nilai yang sebenarnya median akan terbaca sangat aman.
- Karena itu disimpan dua kolom dengan peran berbeda: `risk_score` (hasil `log1p` dan min-max, mempertahankan magnitude, dipakai sebagai target model) dan `risk_score_display` (peringkat persentil, tersebar merata sepanjang 0 sampai 100, lebih intuitif untuk ditampilkan kepada pengguna).

**Membaca bentuk distribusi risk_score yang timpang ke kanan.**
- Bentuk ini bukan cacat, melainkan gambaran yang memang diharapkan, karena dua sebab:
  1. Grid unit kini lengkap, sehingga mayoritas kombinasi lokasi dan jam di area sekitar stasiun memang tidak punya kejadian tercatat sama sekali. Distribusi yang simetris justru akan mencurigakan, karena berarti unit sepi ikut terangkat oleh penghalusan yang terlalu agresif.
  2. Radius penghalusan sengaja dipilih kecil. Radius yang besar akan meratakan hotspot bersama area sepi menjadi tingkat risiko sedang di mana-mana, yang justru menghapus informasi paling berguna bagi Safe Route Recommendation.
- Bentuk timpang inilah yang menjadi alasan kuat mengapa kategorisasi tiga tier memakai ambang batas berbasis persentil, bukan angka absolut yang dibagi rata.

#### 6. Audit EDA Dataset Akhir, Kategorisasi Tier, dan Penyimpanan
- Audit menyeluruh (nilai kosong, duplikasi, outlier, korelasi antar fitur, keseimbangan kelas) sengaja ditempatkan di akhir agar yang diperiksa adalah dataset yang benar-benar akan dipakai, bukan tabel antara yang masih akan berubah.
- Pasangan fitur dengan korelasi sangat tinggi wajib dibaca dari matriks korelasi, karena multikolinearitas berlebihan membuat kontribusi masing-masing fitur sulit dipisahkan dan koefisien model menjadi tidak stabil, meskipun akurasi keseluruhan bisa saja tetap terlihat baik.
- **Fitur `is_weekend`** ditambahkan pada tahap ini sebagai flag biner, karena EDA menunjukkan bentuk pola jam hari kerja dan akhir pekan cukup berbeda, dan `dow_sin`/`dow_cos` tidak membedakan hari kerja dari hari libur.
- **Catatan penting untuk tahap modeling.** Sejumlah fitur bersifat konstan per sel dan terulang di seluruh 168 kombinasi hari dan jam milik sel tersebut, sehingga ukuran sampel efektif untuk fitur-fitur itu adalah jumlah sel, bukan jumlah baris. Pembagian data latih dan data uji secara acak akan menempatkan sel yang sama di kedua sisi dan membuat skor validasi menggembung tanpa dasar. Karena itu pembagian data pada Checkpoint 2 wajib dikelompokkan berdasarkan `cell_id`, misalnya dengan `GroupKFold` atau `GroupShuffleSplit`.

#### Kategorisasi Risk Tier
**Kenapa bukan dibagi tiga sama rata (persentil 33 dan 66).**
- Pembagian pada persentil 33 dan 66 memang adaptif terhadap bentuk distribusi, tetapi hasil akhirnya tetap sepertiga untuk masing-masing tier. Untuk sebuah indikator visual, proporsi seperti itu buruk secara desain produk, karena indikator merah yang menyala di sepertiga peta akan berhenti dibaca sebagai peringatan dalam hitungan hari (kelelahan peringatan), yang justru membuat peringatan yang benar-benar penting ikut diabaikan.

**Ambang batas yang dipakai: persentil 65 dan 90 (target sekitar 65% Aman, 25% Waspada, 10% Rawan).**
- Rawan sekitar sepuluh persen cukup jarang untuk tetap bermakna ketika muncul, sekaligus cukup sering untuk berguna. Proporsi ini sejalan dengan konvensi sistem peringatan publik pada umumnya, yang memang merancang level tertinggi agar jarang menyala.
- Aman sebagai mayoritas adalah gambaran yang akurat, bukan optimisme, karena di area dalam radius 600 meter dari stasiun transit, mayoritas kombinasi lokasi dan waktu memang relatif aman.

**Ambang batas dibekukan setelah dihitung.**
- Nilai `T_LOW` dan `T_HIGH` disimpan sebagai konstanta, bukan dihitung ulang setiap kali data baru masuk. Kalau ambang batas ikut bergeser mengikuti distribusi terbaru, sebuah lokasi bisa berubah warna padahal tingkat risikonya tidak berubah sama sekali, semata karena lokasi lain membaik atau memburuk. Untuk indikator yang dipakai berulang kali, stabilitas semacam ini penting agar warna dapat dipercaya.

**Validasi pemisahan tier.** Tiga hal ikut diperiksa:
1. Apakah sebaran nilai antar tier benar-benar terpisah, bukan garis tipis yang saling tumpang tindih.
2. Apakah sebuah lokasi tidak berpindah tier terlalu sering sepanjang minggu, karena indikator yang berkedip akan terasa tidak dapat dipercaya.
3. Apakah tier Rawan memang mengelompok pada jam-jam yang sesuai dengan temuan EDA. Kalau tier Rawan tersebar merata di seluruh jam, berarti komponen temporal tidak bekerja.

**Nama tier ditulis sebagai teks biasa, bukan emoji**, karena emoji tidak tersedia pada font default matplotlib dan akan memunculkan peringatan glyph pada setiap grafik. Pemetaan ke warna hijau, kuning, dan merah dilakukan di sisi aplikasi.

#### 8. Snapshot Periodik untuk Continual Learning (Persiapan Checkpoint 2)
- Snapshot tunggal dari pipeline utama cukup untuk melatih model utama, tapi tidak bisa dipakai untuk mensimulasikan kedatangan data bertahap di Checkpoint 2, karena tidak ada sumbu waktu kalender yang tersisa di baris unitnya. Karena itu ditambahkan artefak kedua berupa rangkaian snapshot berurutan waktu, dengan alur severity, temporal decay, grid penuh, penyusutan empiris, spatial smoothing, dan normalisasi yang tetap sama persis, hanya dijalankan berulang untuk beberapa periode waktu berurutan.
- Setiap periode diberi kolom `period` yang menandai urutan kronologis aslinya, dipakai nanti untuk membentuk batch data secara kronologis, bukan dengan pengacakan.

##### 8.1 Fungsi Pembangun Satu Snapshot Periode
- Jendela historis untuk sebuah periode hanya berisi kejadian sebelum periode itu dimulai, bukan seluruh histori tiga tahun, meniru kondisi dunia nyata di mana semakin ke belakang periodenya, semakin sedikit histori yang tersedia untuk model, persis seperti continual learning yang sesungguhnya.
- Matriks bobot tetangga spasial (`W`) tidak dihitung ulang tiap periode, karena koordinat sel tidak berubah antar waktu, hanya `base_value` di tiap sel yang berubah. Ini menghemat komputasi secara signifikan dibanding membangun ulang BallTree setiap periode.

##### 8.2 Normalisasi Dibekukan dari Periode Pertama
- Kalau `risk_raw` dinormalisasi ulang secara independen di tiap periode, perubahan asli antar waktu akan hilang, karena tiap periode selalu direntangkan ulang ke rentang 0 sampai 100 relatif terhadap dirinya sendiri, padahal tujuan snapshot ini justru untuk menunjukkan perubahan itu.
- **Solusi:** parameter normalisasi (`log1p` lalu min-max) dihitung sekali dari periode pertama saja, lalu dibekukan dan dipakai konsisten ke seluruh periode berikutnya. Prinsip ini sama dengan target encoding di Checkpoint 2, yaitu statistik dihitung dari data acuan, dibekukan, lalu diterapkan ke data lain tanpa dihitung ulang.

##### 8.3 Kenapa Distribusi risk_score Menjadi Lebih Skewed Setelah Perbaikan
- Setelah `K_SHRINK` diskalakan ulang mengikuti panjang tiap periode, skewness `risk_score` pada dataset time series naik menjadi 4,53, jauh lebih tinggi dibanding 1,76 pada dataset utama. Angka ini bukan tanda kemunduran, melainkan konsekuensi langsung dari perbaikan itu sendiri.
- Sebelum diperbaiki, `K_SHRINK` yang masih memakai nilai kalibrasi jendela 180 hari terlalu besar untuk jendela 30 hari, sehingga hampir seluruh sel (baik yang sangat aman maupun yang berisiko tinggi) ditarik mendekati rata-rata global. Distribusi yang tampak rendah skewness-nya pada tahap itu sebetulnya adalah artefak dari shrinkage yang terlalu berat, bukan cerminan sinyal risiko yang sesungguhnya.
- Setelah `K_SHRINK` diskalakan proporsional, sel-sel sepi diperbolehkan bernilai mendekati nol, dan sel-sel hotspot diperbolehkan menonjol sesuai kejadian yang benar-benar terjadi. Kejahatan memang secara alami terkonsentrasi secara spasial, sebagian kecil lokasi menyumbang sebagian besar kejadian, sehingga distribusi yang miring tajam ke kanan ini justru lebih mencerminkan kondisi sebenarnya.
- **Dua implikasi sebagai keterbatasan, bukan kegagalan:**
  1. Skala 0 sampai 100 menjadi jarang terpakai penuh, karena parameter normalisasi dibekukan dari periode pertama yang kebetulan memiliki risk_raw maksimum paling kecil dibanding periode-periode sesudahnya, sehingga nilai pada periode belakangan yang lebih ekstrem sebagian terpotong ke batas atas skala periode acuan.
  2. Ambang batas tier (`T_LOW`, `T_HIGH`) yang dibekukan dari dataset utama tidak dapat dipinjam langsung untuk dataset time series ini, karena konteks normalisasinya berbeda. Ambang batas tier untuk data periodik ini perlu dihitung ulang secara khusus dari distribusinya sendiri, idealnya berbasis kuantil karena bentuknya yang skewed.

### CHECKPOINT 2: Model Prediksi & Continual Learning

#### 1. Load Dataset dari Checkpoint 1
- Empat berkas dimuat, masing-masing dengan peran berbeda untuk fitur aplikasi:
  - `features_labels.csv`: dataset utama (satu baris per sel grid x hari x jam), dipakai untuk melatih model Risk Prediction, dengan kolom `risk_tier` langsung dipakai sebagai dasar Visual Risk Indicator tanpa perlu dihitung ulang.
  - `features_labels_timeseries.csv`: rangkaian snapshot risk_score berurutan waktu per periode bulanan, dipakai khusus untuk Continual Learning. Skalanya berbeda dari `features_labels.csv` karena dasar normalisasinya berbeda, sehingga tidak boleh dicampur langsung dalam satu perbandingan.
  - `model_config.csv`: parameter yang dipakai membentuk kedua dataset di atas (`T_LOW`, `T_HIGH`, `half_life_days`, `radius_km`, dst), dipakai untuk menjaga konsistensi ambang batas Visual Risk Indicator dan sebagai dokumentasi reproduksi hasil.
  - `safe_places.csv`: daftar 751 titik tempat aman lengkap koordinatnya, dipakai langsung untuk fitur Safe Place Locator.

#### 2. Baseline Non-ML

##### 2.1 Baseline Terkuat
- Dari tiga baseline dasar (global mean, rata-rata per sel, rata-rata per hari-jam), rata-rata per sel adalah yang paling kuat: MAE dan RMSE-nya paling kecil, R2-nya paling besar. R2 baseline ini menunjukkan porsi variasi risk_score yang bisa dijelaskan hanya dari identitas sel saja, jauh lebih besar dibanding porsi yang dijelaskan pola waktu semata, sementara global mean nyaris tidak menjelaskan apa-apa karena tidak memakai informasi apa pun. Ini menunjukkan risk_score sangat didominasi oleh karakteristik lokasi.

##### 2.2 Kombinasi Dua Pola
- Mengombinasikan sel dengan variabel waktu justru memperbaiki performa, bukan memperburuknya. Kombinasi Sel + Jam menjadi baseline terkuat secara keseluruhan, mengalahkan rata-rata per sel saja. Ini berbeda dari dugaan awal bahwa memecah data lebih detail akan membuat rata-rata per kombinasi jadi tidak stabil karena kekurangan sampel, ternyata karena `features_labels.csv` sekarang berbentuk grid penuh (satu baris untuk tiap kombinasi sel, hari, dan jam), setiap kombinasi Sel+Jam tetap punya representasi yang cukup untuk menghasilkan rata-rata yang stabil.
- Kombinasi Sel + Hari dan Sel + Weekend tidak seunggul Sel + Jam, menunjukkan variasi menurut jam dalam sehari lebih informatif dibanding variasi menurut hari dalam seminggu untuk risk_score di dataset ini. Kesimpulannya, variasi risk_score memang didominasi faktor lokasi, tapi menambahkan resolusi waktu berupa jam tetap memberikan sinyal tambahan yang berarti, bukan sekadar noise.

##### 2.3 Strategi Fallback
- Strategi fallback untuk baseline kombinasi terdiri dari tiga tingkat: kombinasi persis, lalu rata-rata per sel saja, lalu global mean sebagai jaring pengaman terakhir.
- Kecocokan persis di ketiga kombinasi ternyata sangat tinggi, mendekati keseluruhan data test, berbeda jauh dari perkiraan awal bahwa kombinasi lebih detail (Sel+Jam) akan sering jatuh ke fallback. Penyebabnya, karena `features_labels.csv` sekarang grid penuh, hampir setiap kombinasi sel-hari-jam yang muncul di data test juga sudah pernah muncul di data train, sehingga fallback nyaris tidak pernah diperlukan. Ini juga menjelaskan kenapa performa baseline kombinasi bisa sebaik itu, prediksinya memang berasal dari kecocokan spesifik, bukan pelarian ke rata-rata yang lebih general.

##### 2.4 Justifikasi Pilihan Baseline
- Baseline yang dipilih sebagai pembanding utama untuk model regresi adalah Kombinasi Sel + Jam, bukan rata-rata per sel saja, karena performanya secara empiris memang lebih baik, konsisten dengan temuan fallback di mana kombinasi ini mendapat kecocokan persis yang nyaris sempurna. Model regresi baru bisa dianggap memberikan nilai tambah jika MAE dan RMSE-nya lebih rendah, serta R2-nya lebih tinggi dibanding baseline ini.

#### 3. Feature Vector Assembly
- Seluruh fitur dari Checkpoint 1 dibungkus dalam satu fungsi `assemble_features_v2`, supaya representasi fitur yang dipakai saat training selalu identik dengan representasi saat serving nanti, karena inkonsistensi kecil di sini adalah salah satu sumber bug paling umum di sistem ML produksi.

##### 3.1 Frequency Encoding dan Target Encoding untuk cell_id
- Frequency encoding merepresentasikan seberapa sering sebuah sel muncul di data training, sedangkan target encoding merepresentasikan rata-rata risk_score historis sel tersebut. Target encoding dihitung hanya dari `train_df`, lalu dipetakan ke `test_df`, supaya tidak terjadi data leakage dari label test ke dalam fitur. Sel yang tidak pernah muncul di `train_df` diberi nilai `global_mean` sebagai fallback, konsisten dengan strategi fallback di bagian baseline.

##### 3.2 Pemilihan Fitur Berdasarkan Korelasi
- Korelasi tiap kandidat fitur terhadap risk_score dicek memakai data training. `cell_target_enc` memiliki korelasi paling kuat, jauh di atas fitur lain manapun, konsisten dengan temuan bagian baseline bahwa risk_score paling banyak ditentukan oleh karakteristik lokasi. `crime_diversity` dan `hist_crime_count` juga cukup relevan. Sebaliknya, `cell_freq_enc`, `dow_sin`, dan `dow_cos` memiliki korelasi yang sangat kecil, sehingga kontribusinya kemungkinan minim. `hour_sin` dan `hour_cos` tetap dipertahankan karena korelasinya jauh lebih besar, sejalan dengan temuan baseline bahwa kombinasi Sel + Jam mengungguli Sel saja.

##### 3.3 Standardisasi/Scaling
- Standardisasi tidak diterapkan, karena model yang dipakai tidak sensitif terhadap skala fitur. Jika nanti dicoba model berbasis jarak seperti KNN, standardisasi menjadi wajib, karena fitur dengan rentang nilai besar seperti crime_count bisa mendominasi perhitungan jarak dibanding fitur dengan rentang kecil seperti hour_sin atau hour_cos.

##### 3.4 Representasi Fitur (assemble_features_v2)
- Representasi fitur dibungkus dalam satu fungsi yang dipakai baik saat training maupun serving. Asumsi yang didokumentasikan agar konsisten di kedua sisi: `cell_target_enc` dan `cell_freq_enc` harus dihitung dari lookup table yang sama persis dengan yang dipakai saat training, dan sel baru yang belum pernah terlihat sebelumnya harus diberi nilai fallback `global_mean`, bukan nol atau nilai kosong.

#### 4. Model Regresi: Training

##### 4.1 Analisis Linearitas
- Indikasi non-linearitas dicek dengan membandingkan korelasi Pearson (hubungan linear) dan Spearman (hubungan monotonik termasuk non-linear) untuk tiap fitur. Selisih besar antara keduanya mengindikasikan hubungan yang tidak sepenuhnya linear. Selisih terbesar ada pada `cell_target_enc`, menunjukkan fitur ini punya komponen non-linear meskipun hubungan dasarnya tetap kuat secara linear. Fitur lain seperti `lat_r`, `lon_r`, `hour_sin`, `hour_cos`, `crime_count`, dan `crime_diversity` memiliki selisih yang jauh lebih kecil, menunjukkan hubungannya dengan risk_score sudah cukup dekat dengan linear.
- Temuan ini konsisten dengan hasil perbandingan model, di mana Linear Regression hanya sedikit di bawah Gradient Boosting dari sisi R2, masuk akal karena data memang didominasi hubungan yang cenderung linear, dengan hanya sedikit komponen non-linear terutama pada `cell_target_enc`.

##### 4.2 Pemilihan Model
- Ketiga model (Linear Regression, Random Forest, Gradient Boosting) yang dilatih dengan feature set yang sudah termasuk `cell_target_enc` semuanya mengalahkan baseline terkuat (rata-rata per sel). Gradient Boosting menjadi yang terbaik dari sisi MAE, RMSE, dan R2, disusul Linear Regression, lalu Random Forest.
- **Kenapa Linear Regression mengungguli Random Forest.** Padahal secara umum model berbasis pohon biasanya lebih unggul karena mampu menangkap hubungan non-linear dan interaksi antar fitur, hal ini terjadi karena grafik feature importance dari Random Forest dan Gradient Boosting menunjukkan `cell_target_enc` mendominasi jauh di atas fitur lain di kedua model. Karena `cell_target_enc` pada dasarnya sudah berupa rata-rata risk_score per sel, hubungannya dengan risk_score sudah relatif dekat dengan linear, sehingga model linear bisa memanfaatkan sinyal ini secara efisien tanpa perlu kompleksitas tambahan dari struktur pohon.
- **Kenapa Gradient Boosting tetap terbaik.** Kemungkinan karena sifat boosting yang melatih pohon secara berurutan untuk mengoreksi residual dari pohon sebelumnya, sehingga bisa memperbaiki kesalahan kecil yang tersisa setelah menangkap sinyal utama dari `cell_target_enc`, sesuatu yang tidak bisa dilakukan Linear Regression maupun Random Forest dengan cara yang sama.
- Feature importance ini konsisten dengan hasil korelasi sebelumnya, di mana `cell_target_enc` memang memiliki korelasi paling kuat. Fitur `lat_r`, `lon_r`, `hist_crime_count`, dan `crime_diversity` menempati urutan berikutnya di Random Forest, sementara di Gradient Boosting kontribusi fitur selain `cell_target_enc` jauh lebih kecil, menunjukkan Gradient Boosting lebih terkonsentrasi mengandalkan satu sinyal dominan tersebut.

##### 4.3 Verifikasi Asumsi Model
- Untuk verifikasi asumsi Linear Regression, plot residual dicek. Rata-rata residualnya mendekati nol, menunjukkan model tidak bias secara sistematis ke satu arah. Sebaran titik pada plot Residual vs Prediksi memusat rapat di rentang prediksi rendah (karena mayoritas data risk_score memang berada di rentang rendah), dengan sebaran yang melebar simetris di atas dan di bawah garis nol, dan menyempit di rentang prediksi lebih tinggi karena data di sana jauh lebih jarang. Histogram residual berbentuk mendekati lonceng dan terpusat di sekitar nol, mengindikasikan asumsi normalitas residual cukup terpenuhi, meskipun ada sedikit ekor di sisi kanan.
- Berdasarkan seluruh hasil ini, Gradient Boosting dipilih sebagai model final, karena secara konsisten memberikan MAE dan RMSE terendah serta R2 tertinggi dibanding Linear Regression, Random Forest, dan seluruh baseline yang diuji sebelumnya.

#### 5. Evaluasi Model vs Baseline

##### 5.1 Mengungguli Semua Baseline
- Ketiga model (Linear Regression, Random Forest, Gradient Boosting) mengungguli seluruh baseline yang diuji, baik dari sisi MAE, RMSE, maupun R2. Gradient Boosting menjadi model terbaik secara keseluruhan di ketiga metrik tersebut.

##### 5.2 Metrik Paling Relevan untuk Risk Score
- RMSE dinilai sebagai metrik yang paling relevan dibanding MAE, meskipun keduanya dilaporkan. RMSE memberi penalti lebih besar pada kesalahan yang besar, dan dalam konteks penilaian risiko, kesalahan besar (misalnya memprediksi area sangat berisiko sebagai area aman) jauh lebih berbahaya secara operasional dibanding kesalahan kecil yang tersebar merata. R2 dipakai sebagai pelengkap untuk menunjukkan seberapa besar variasi risk score yang berhasil dijelaskan model dibanding hanya menebak rata-rata.

##### 5.3 dan 5.4 MAE per Rentang Risk Score dan Analisis
- Meskipun model unggul secara rata-rata di seluruh data, MAE per rentang risk score aktual menunjukkan pola kesalahan yang jauh dari merata. Mayoritas data test berada di rentang risk score rendah, dan di situ pula model paling akurat. Namun performa model menurun tajam begitu rentang risk score aktualnya semakin tinggi, dan pada rentang paling tinggi, rata-rata prediksinya jauh di bawah rata-rata aktualnya.
- Pola ini adalah bentuk regresi ke arah rata-rata (regression to the mean), yaitu ketika model cenderung memprediksi nilai mendekati rata-rata keseluruhan karena data pelatihan didominasi oleh nilai di rentang rendah. Ini konsisten dengan sebaran risk_score itu sendiri yang memang sangat condong ke nilai rendah, sehingga rentang risk score tinggi hanya diwakili sedikit baris dari total data, dan model tidak punya cukup contoh untuk belajar pola pada nilai-nilai tinggi tersebut.
- Penyebabnya bukan karena fitur kurang informatif atau model terlalu sederhana, melainkan karena ketimpangan jumlah data (data imbalance), di mana risk score tinggi jauh lebih jarang muncul dibanding risk score rendah, sehingga model condong ke prediksi aman di sekitar rentang rendah yang dominan daripada berani memprediksi nilai tinggi yang jarang muncul di data training.

### 6. Continual Learning: Simulasi Kedatangan Data Bertahap

##### 6.1 Skema Batching
- Skema batching buatan dengan drift yang disuntikkan secara sintetis (pembagian batch tidak seragam plus tiga jenis drift berbeda per fase) sempat dicoba, tapi kemudian dialihkan ke skema yang jauh lebih jujur. `features_labels_timeseries.csv` punya kolom `period` asli yang merepresentasikan urutan waktu sesungguhnya, sehingga batch training disusun langsung dari periode-periode tersebut secara kronologis, dengan periode terakhir disisihkan sebagai holdout evaluasi yang tidak pernah dipakai training.
- Pendekatan ini menghilangkan kebutuhan akan drift buatan sama sekali. Setiap batch berukuran sama (satu batch mewakili satu periode penuh), dan drift yang muncul di antara batch adalah drift alami yang benar-benar ada di data, bukan hasil rekayasa. Ini jauh lebih jujur secara metodologis dibanding pendekatan pengacakan sebelumnya, karena kesimpulan continual learning yang diambil sekarang benar-benar mencerminkan bagaimana pola risk_score berubah dari waktu ke waktu, bukan sekadar menguji apakah mekanisme drift detection bisa mendeteksi perubahan buatan sendiri.
- Hasilnya, uji KS mendeteksi drift signifikan pada `hist_crime_count` maupun `risk_score` di seluruh batch tanpa terkecuali. Nilai statistik KS-nya cenderung membesar seiring bertambahnya jarak periode dari batch referensi, menunjukkan pergeseran distribusi yang gradual dan konsisten, bukan lompatan tiba-tiba di satu titik saja, masuk akal untuk data kriminalitas riil karena pola kejahatan memang bergeser pelan-pelan dari waktu ke waktu.

##### 6.2 Jumlah Batch
- Karena batch mengikuti periode asli di `features_labels_timeseries.csv`, jumlah batch training bukan lagi pilihan bebas seperti pada skema buatan sebelumnya, melainkan mengikuti jumlah periode yang tersedia dikurangi satu periode terakhir yang disisihkan sebagai holdout. Setiap batch otomatis berukuran sama, karena satu batch memang mewakili satu periode penuh, bukan hasil pembagian proporsi yang ditentukan sendiri.
- **Trade off:** tidak lagi bisa mengontrol secara eksplisit kapan dan jenis drift apa yang muncul di batch mana, karena semuanya bergantung pada pola asli di data. Namun ini justru membuat hasil continual learning lebih bisa dipercaya, karena mencerminkan skenario dunia nyata di mana tim data science jarang tahu persis kapan dan jenis drift apa yang akan muncul di data produksi mereka, sehingga sistem continual learning harus tetap bekerja dengan baik meski bentuk drift-nya tidak terprediksi.

#### 7. Drift Detection

##### 7.1 Menambahkan Metrik Lain (PSI)
- `detect_drift` dikembangkan dari versi tutorial dengan menambahkan dua komponen. Komponen pertama adalah Population Stability Index (PSI) sebagai pelengkap uji KS, keduanya sama-sama mengukur perubahan distribusi marginal tiap kolom tapi dengan formulasi berbeda.
- Hasilnya menunjukkan kontras yang menarik: uji KS mendeteksi drift di setiap batch untuk kedua kolom, namun PSI hampir selalu tetap jauh di bawah ambang drift signifikan di semua batch. Ini terjadi karena ukuran tiap batch sangat besar, membuat uji KS menjadi sangat sensitif, bahkan pergeseran distribusi yang kecil secara praktis pun sudah cukup untuk menghasilkan p-value yang sangat kecil. PSI, sebaliknya, mengukur besarnya pergeseran secara lebih proporsional terhadap ukuran datanya, sehingga lebih mencerminkan signifikansi praktis dibanding signifikansi statistik semata. Temuan ini jadi pengingat penting bahwa ambang p-value pada uji KS perlu dipertimbangkan bersama ukuran sampel, bukan dipakai sebagai satu-satunya indikator drift.

##### 7.2 Selain Drift Fitur (Concept Drift)
- Komponen kedua adalah pengecekan concept drift secara eksplisit, dengan membandingkan korelasi antara `crime_count` dan `risk_score` pada data referensi versus tiap batch baru. Hasilnya, korelasi ini relatif stabil di seluruh batch, pergeserannya jauh di bawah ambang yang ditetapkan setiap kali. Ini menunjukkan bahwa meskipun distribusi marginal fitur dan label bergeser dari waktu ke waktu (seperti terdeteksi KS), hubungan mendasar antara `crime_count` dan `risk_score` tetap konsisten sepanjang periode data ini, sehingga tidak ada indikasi concept drift yang berarti pada data historis yang dimiliki.

##### 7.3 Penentuan Ambang Batas
- Untuk `DRIFT_ALPHA` pada uji KS, nilai dari tutorial tetap dipertahankan. Ambang ini dinilai cukup ketat (butuh bukti statistik kuat sebelum menyimpulkan drift), namun pada praktiknya dengan ukuran tiap batch yang sangat besar, ambang tersebut hampir selalu terlampaui meski pergeserannya kecil secara praktis. Trade off-nya, ambang KS jadi kurang informatif untuk data berukuran besar seperti ini, karena hampir selalu men-trigger drift.
- Untuk PSI, ambang yang dipakai mengikuti konvensi umum di industri, dan untuk concept drift dipakai ambang pergeseran korelasi yang dipilih agak longgar. Berbeda dengan KS, kedua ambang ini terbukti jauh lebih stabil dan tidak mudah ke-trigger oleh fluktuasi kecil akibat ukuran data yang besar, sehingga PSI dan concept drift correlation shift dinilai lebih representatif untuk menilai signifikansi praktis dibanding uji KS semata pada kasus data sebesar ini.

#### 8. Checkpoint Retraining dan Model Versioning

##### 8.2 Kriteria Promosi
- Kriteria promosi yang dipakai bukan sekadar "MAE kandidat lebih baik atau sama dengan champion" seperti versi tutorial, tapi diperketat menjadi dua syarat sekaligus: kandidat harus unggul MAE minimal 0.02 (`PROMOTE_MARGIN`) dari champion, dan R2-nya tidak boleh turun dari champion. Kedua syarat ini harus terpenuhi bersamaan sebelum kandidat dipromosikan.
- **Alasan margin 0.02.** Penting untuk mencegah promosi yang hanya didasari fluktuasi acak kecil, bukan perbaikan yang benar-benar berarti. Ini terbukti pada strategi sliding window, di mana kandidat v2 sebenarnya sedikit lebih baik dari champion v1, tapi selisihnya di bawah margin 0.02, sehingga sistem memutuskan `kept_champion`. Tanpa margin ini, sistem akan terus-menerus mengganti champion untuk perbaikan yang secara praktis tidak signifikan, membuat proses retraining kurang stabil dan membuang biaya komputasi.
- **Alasan syarat R2 tidak boleh turun.** Penting sebagai pengaman kedua, karena MAE yang membaik tidak selalu menjamin kualitas model secara keseluruhan lebih baik, terutama kalau perbaikan itu terjadi karena model menjadi lebih bias ke arah nilai rata-rata, yang bisa menurunkan MAE namun juga menurunkan kemampuan menjelaskan variasi data (R2). Dengan mensyaratkan keduanya sekaligus, keyakinan lebih besar bahwa kandidat yang dipromosikan benar-benar lebih baik secara menyeluruh, bukan cuma unggul di satu metrik saja.

##### 8.3 Perbandingan Tiga Strategi Retraining
- Tiga strategi dibandingkan: cumulative (melatih ulang dari seluruh data yang terkumpul sejauh ini), sliding window (hanya melatih dari beberapa batch terakhir), dan fine-tuning (melanjutkan model champion yang sudah ada memakai `warm_start` pada Gradient Boosting, hanya menambah pohon baru dari batch terbaru, bukan retrain dari nol).
- Dengan skema batching kronologis berbasis periode asli, tidak ada batch yang mengalami drift ekstrem buatan, sehingga ketiga strategi sama-sama menunjukkan tren yang relatif mulus, MAE holdout membaik secara bertahap seiring bertambahnya batch, tanpa penurunan performa drastis di titik manapun.
- Secara keseluruhan performa akhir, cumulative menjadi yang terbaik, diikuti fine-tune yang cukup dekat, dan sliding window paling lemah. Pola ini masuk akal: cumulative diuntungkan karena terus mengakumulasi volume data historis yang membuat model semakin stabil, fine-tune mendekati performanya karena tetap mewarisi pengetahuan dari model sebelumnya sambil menambah kapasitas pohon baru secara bertahap, sementara sliding window paling terbatas karena jendela datanya yang kecil membuatnya kehilangan sinyal historis yang membantu performa dua strategi lain.
- Sliding window juga yang paling sering mempertahankan champion lama (`kept_champion`) dibanding dua strategi lain, menunjukkan bahwa dengan volume data yang terbatas per batch, kandidat barunya lebih sering gagal mengungguli champion dengan margin yang cukup meyakinkan.
- **Keputusan akhir:** cumulative dipilih sebagai strategi retraining final, karena performa akhirnya terbaik dan trennya paling konsisten membaik seiring bertambahnya data. Fine-tune dicatat sebagai alternatif yang layak dipertimbangkan bila komputasi menjadi kendala di skala produksi yang jauh lebih besar, mengingat performanya cukup dekat dengan cumulative namun dengan biaya pelatihan yang jauh lebih murah karena tidak retrain dari nol setiap kali.

##### 8.4 Model yang Digunakan
- Fungsi ini memakai `FEATURE_COLS_V2` dengan encoding `cell_target_enc` yang dihitung ulang dari data yang sedang aktif di tiap checkpoint, bukan encoding statis dari `train_df` awal, supaya konsisten dengan prinsip mencegah kebocoran data antar waktu.

##### 8.5 Dokumentasi Keputusan (Model Journey)
- Setiap keputusan retraining, baik yang berhasil dipromosikan maupun yang gagal, dicatat lewat registry (list of dict berisi `version`, `batch_index`, `train_size`, `drift_detected`, `metrics`, dan `decision`), bukan hanya menyimpan versi yang berhasil saja.
- **Model journey strategi cumulative.** Model awal (v0) dilatih dari batch pertama saja. Setiap batch baru yang datang selalu terdeteksi drift oleh KS (wajar, mengingat temuan bahwa KS sangat sensitif pada data sebesar ini), sehingga kandidat baru selalu dilatih ulang. Kandidat di batch-batch awal berhasil dipromosikan karena peningkatan MAE-nya melewati `PROMOTE_MARGIN`, namun begitu champion sudah cukup kuat, kandidat berikutnya tetap membaik tapi peningkatannya semakin tipis dan akhirnya jatuh di bawah margin, sehingga sistem mempertahankan champion yang sudah terbentuk di batch-batch pertengahan. Ini menunjukkan mekanisme champion vs challenger bekerja sebagai jaring pengaman yang wajar, mempertahankan model yang sudah cukup baik alih-alih terus mengganti champion untuk perbaikan yang praktis tidak berarti.
- **Model journey strategi sliding window.** Pola serupa terjadi, model awal (v0) sama seperti cumulative, kandidat awal juga sempat dipromosikan, namun setelahnya sistem jauh lebih sering mempertahankan champion lama dibanding dua strategi lain. Ini konsisten dengan performa akhirnya yang paling lemah, karena jendela data yang kecil membuat kandidat-kandidat barunya jarang cukup meyakinkan untuk melewati `PROMOTE_MARGIN`.
- Dengan pencatatan registry yang mencakup semua keputusan ini (baik promoted maupun kept_champion), riwayat kapan dan kenapa tiap versi model dibuat atau ditolak bisa ditelusuri kembali tanpa perlu membongkar ulang kode.

#### 9. Model Registry dan Checkpoints

##### 9.1 Informasi yang Perlu Dicatat
- Registry versi tutorial sudah mencatat `version`, `batch_index`, `train_size`, `drift_detected`, `metrics`, dan `decision`, tapi ada tiga informasi penting yang masih kurang:
  1. **Dataset fingerprint**, yaitu checksum ringkas dari data yang dipakai melatih versi model tersebut (jumlah baris, daftar kolom, dan jumlah total risk_score). Penting supaya kalau muncul pertanyaan "model v2 ini dilatih dari data yang persis sama dengan yang saya punya sekarang atau tidak", jawabannya bisa dicek lewat fingerprint tanpa harus menyimpan seluruh dataset mentahnya di registry.
  2. **Hyperparameter model**, misalnya jumlah pohon (`n_estimators`), `random_state`, atau untuk fine-tune, informasi `n_estimators_increment` dan `window_size` untuk sliding window. Penting karena ketiga strategi retraining punya konfigurasi berbeda, dan tanpa mencatat ini, sulit membedakan versi model mana yang dihasilkan dari konfigurasi mana kalau hanya melihat file checkpoint-nya saja.
  3. **`drift_trigger`**, yaitu komponen spesifik mana dari `detect_drift_v2` yang benar-benar memicu `drift_detected=True` (apakah `ks_hist_crime_count`, `ks_risk_score`, `psi_hist_crime_count`, `psi_risk_score`, atau `concept_drift`). Sebelumnya `drift_report` menyimpan detail semua komponen, tapi tidak langsung menunjukkan mana yang menjadi trigger utama. Field ini penting untuk audit cepat, dan pada data ini field ini justru mengungkap temuan menarik, yaitu `drift_trigger` di seluruh batch konsisten hanya berisi komponen KS, sementara PSI dan `concept_drift` tidak pernah menjadi trigger sama sekali, mengonfirmasi temuan bahwa uji KS jauh lebih sensitif dibanding dua komponen lainnya pada data berukuran besar seperti ini.
- Timestamp eksplisit juga ditambahkan di setiap entri, beserta field `strategy` untuk membedakan versi mana berasal dari strategi cumulative, sliding_window, atau finetune, karena ketiganya dijalankan dan dibandingkan sekaligus, bukan hanya satu strategi seperti di tutorial.

##### 9.2 Format yang Sesuai
- Format tabular (disimpan sebagai CSV, `models/registry_v2.csv`) dipakai, bukan JSON list of dict seperti versi tutorial. Alasannya, dengan tiga strategi retraining yang dibandingkan sekaligus, format tabular jauh lebih mudah untuk difilter dan dibandingkan langsung, misalnya `registry_df_v2[registry_df_v2["decision"] == "promoted"]` untuk melihat semua promosi lintas strategi, atau `registry_df_v2.groupby("strategy")["metrics.MAE"].min()` untuk membandingkan performa terbaik tiap strategi.
- Struktur ini konsisten dipertahankan di setiap versi, satu baris per keputusan (baik promoted, kept_champion, maupun skip_retrain_no_drift), dengan kolom yang sama untuk seluruh strategi, sehingga registry ini bisa langsung dibuka dan dibaca lewat spreadsheet biasa tanpa perlu membongkar kode maupun mem-parsing JSON bersarang.

#### 11. Integrasi ke Fitur Aplikasi

##### 11.0.1 Mock Koordinat untuk MRT Jakarta
- Model dilatih dari data Chicago, sehingga koordinat mentah Jakarta berada jauh di luar rentang yang pernah dilihat model. Supaya tetap bisa dipakai untuk demo aplikasi di rute MRT Jakarta, posisi relatif tiap stasiun diproyeksikan ke rentang koordinat Chicago yang dikenali model, bukan dipakai apa adanya, sambil menjaga posisi relatif antar titik.
- Skor yang dihasilkan lewat mock ini adalah simulasi pola risiko Chicago yang diproyeksikan secara geometris ke peta Jakarta, bukan prediksi berbasis kondisi keamanan Jakarta yang sesungguhnya. Karena itu, setiap respons dari `get_risk_indicator` menyertakan flag `is_mock` supaya BE dan frontend bisa menampilkan disclaimer ini ke pengguna.

##### 11.1 Visual Risk Indicator
- Fungsi menerima koordinat dan waktu, lalu mengembalikan warna indikator. Koordinat mentah dibulatkan ke resolusi grid yang sama dengan Checkpoint 1 supaya cocok dengan sel yang dikenali model, lalu risk_score hasil prediksi dipetakan ke tier memakai `T_LOW` dan `T_HIGH` yang sama dari `model_config.csv`, bukan dihitung ulang.

##### 11.2 Safe Place Locator
- Fungsi mencari tempat aman terdekat dari satu koordinat, memakai BallTree dengan metrik jarak haversine, sama seperti cara Checkpoint 1 menghitung jarak antar koordinat bumi, supaya hasilnya konsisten dengan perhitungan yang sudah divalidasi di sana.

##### 11.3 Safe Route Recommendation
- Rute teraman dibentuk lewat graf antar sel grid yang bertetangga langsung secara spasial. Setiap sisi graf diberi bobot gabungan antara jarak dan risiko rata-rata dua sel yang dihubungkan, sehingga jalur pencarian bisa memilih memutar sedikit demi menghindari sel berisiko tinggi. Sebagai pembanding, rute tercepat (hanya berdasar jarak, tanpa mempertimbangkan risiko) juga dihitung, supaya trade off antara kedua pilihan ini terlihat jelas.

---

## Refleksi Akhir

Proses pengerjaan proyek ini menekankan pentingnya membangun keputusan berbasis bukti empiris di setiap tahap, mulai dari audit data mentah sebelum membuang baris apa pun, verifikasi setiap modifier severity scoring benar-benar aktif, hingga pengujian kuantitatif untuk memilih resolusi grid dan metode normalisasi. Pendekatan iteratif dari versi baseline yang sederhana menuju versi yang dikembangkan, disertai justifikasi tertulis di setiap tahap, membantu memastikan tiap keputusan bisa dipertanggungjawabkan, bukan sekadar mengikuti kebiasaan atau template.

Tantangan terbesar pada Checkpoint 1 adalah mencegah kebocoran target (target leakage), terutama pada pemisahan jendela fitur dan label, serta memastikan grid unit dibentuk secara penuh (bukan hanya dari kombinasi yang benar-benar terjadi) supaya model punya representasi area aman yang valid. Konsekuensi dari pilihan resolusi grid yang halus (sparsity) juga harus ditangani secara eksplisit lewat penyusutan empiris dan spatial smoothing, karena membiarkan sparsity tanpa penanganan akan membuat Risk Score menjadi tidak stabil dan sulit diinterpretasi.

Pada Checkpoint 2, temuan paling menarik adalah bagaimana cell_target_enc mendominasi seluruh model, sehingga model linear yang sederhana bisa mengungguli model berbasis pohon yang lebih kompleks seperti Random Forest. Ini menjadi pengingat bahwa kompleksitas model tidak selalu berbanding lurus dengan performa, jika sinyal utama dalam data sudah relatif linear. Tantangan lain yang cukup mendasar adalah menyadari bahwa evaluasi rata-rata (seperti MAE dan RMSE keseluruhan) bisa menyembunyikan kelemahan model di area yang justru paling penting secara operasional, yaitu area risiko tinggi, akibat ketimpangan distribusi data. Hal ini menegaskan pentingnya melakukan analisis performa per segmen, bukan hanya bergantung pada satu angka ringkasan.

Pada bagian continual learning dan drift detection, pelajaran penting yang didapat adalah bahwa satu metrik statistik saja (seperti uji KS) tidak cukup untuk menyimpulkan drift yang bermakna secara praktis, terutama pada data berukuran besar yang membuat uji tersebut menjadi terlalu sensitif. Menggabungkan beberapa metrik dengan sifat berbeda (KS, PSI, dan concept drift) memberikan gambaran yang jauh lebih seimbang dan dapat dipercaya untuk pengambilan keputusan retraining di dunia nyata.

Secara keseluruhan, proyek ini memperlihatkan bahwa MLOps bukan hanya soal melatih model dengan akurasi tinggi, tetapi juga soal membangun pipeline yang bisa dipertanggungjawabkan, transparan dalam setiap keputusan, dan siap menghadapi perubahan data dari waktu ke waktu melalui mekanisme continual learning yang terstruktur.

**Yang sudah berjalan baik**
Pipeline end-to-end sudah utuh, lengkap dengan mekanisme continual learning, drift
detection, dan model versioning yang bisa diaudit tanpa membongkar kode.

**Keterbatasan yang disadari**
- Model paling lemah justru di rentang risk score tinggi, kasus yang secara operasional paling penting, karena data imbalance pada nilai ekstrem.
- Demo untuk Jakarta sepenuhnya bersifat mock (proyeksi geometris dari pola Chicago), bukan prediksi berbasis data keamanan Jakarta yang sesungguhnya.
- Graf risiko saat ini terbatas pada sel sel yang muncul di data historis Chicago, belum tentu merepresentasikan seluruh area yang mungkin dilewati rute MRT Jakarta secara presisi.

**Pengembangan lanjutan**
- Kalau tersedia data kriminalitas Jakarta asli, model bisa dilatih ulang tanpa perlu mock coordinate sama sekali.
- Strategi retraining bisa dikembangkan lebih lanjut, misalnya fine-tune dengan sebagian kecil data historis disisipkan untuk mengurangi risiko terhadap label drift di skala produksi yang lebih besar.

---

## Cara Menjalankan

1. Jalankan `notebooks/Checkpoint1.ipynb` pada Google Collab secara berurutan dari atas ke bawah untuk menghasilkan data di `data/` (atau unduh langsung, lihat `README.md` di folder tersebut).
2. Jalankan `notebooks/Checkpoint2.ipynb` pada Google Collab secara berurutan dari atas ke bawah untuk menghasilkan `models/` dan `export/` (atau unduh langsung, lihat `README.md` di folder tersebut).

---

## Struktur Folder

```text
ML-HerRoute/
├── Checkpoint1.ipynb                       # Data prep: agregasi data mentah jadi fitur per sel/waktu
├── Checkpoint2.ipynb                       # Baseline, model, continual learning, drift, registry, integrasi
│
├── data/                                    # Output Checkpoint 1, dipakai sebagai input Checkpoint 2
│    ├── features_labels.csv                 # Dataset utama: fitur + label risk_score per (sel, hari, jam)
│    ├── features_labels_timeseries.csv      # Snapshot risk_score per periode waktu asli, untuk continual learning
│    ├── model_config.csv                    # Parameter ambang batas (T_LOW, T_HIGH, dst), untuk konsistensi
│    └── safe_places.csv                     # Daftar tempat aman + koordinat
│
├── models/                                  # Riwayat training (Checkpoint 2, section 8-9), Bukan untuk deployment
│   ├── model_{strategy}_v{version}.joblib   # Checkpoint tiap versi model, tiap strategi retraining
│   └── registry_v2.csv                      # Log semua keputusan retraining lintas 3 strategi
│
└── export/                                  # Paket deployment final (Checkpoint 2, section 11), siap dipakai BE
    ├── champion_model.joblib
    ├── model_context.joblib
    ├── cell_hist_lookup.joblib
    ├── remap_bounds.joblib
    ├── risk_graph.joblib                    # Graf risiko yang sudah jadi (node, edge, bobot), bukan data mentah
    ├── safe_places.csv
    └── model_config.csv
```