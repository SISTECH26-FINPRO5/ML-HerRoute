## 📁 Output Data - Checkpoint 1

Folder `data/` dihasilkan dari notebook `Checkpoint1.ipynb`.
Karena ukuran file cukup besar, file tidak disimpan secara langsung di dalam repositori ini. Dapat mengunduh file lengkap melalui link Google Drive di bawah ini, atau generate ulang sendiri dengan menjalankan notebook dari awal dengan Google Collab.

* 🔗 **Link Google Drive:** [Download Data Hasil di Sini](https://drive.google.com/drive/folders/1UzjRl8eDwNEjZcMb4LBOpe_DyhfobrDo?usp=sharing)

#### Panduan Penempatan File:
Setelah mengunduh file/folder dari Google Drive:
1. Ekstrak file jika berformat zip/rar.
2. Letakkan file hasil tersebut ke dalam direktori proyek berikut:
```text
    ML-HerRoute/
    └── data/
        └── output/    <-- Letakkan file di sini
```

#### Isi Folder

| File | Isi |
| :-- | :-- |
| `features_labels.csv` | Dataset utama, satu baris per unit analisis (sel grid x hari x jam), berisi fitur ruang-waktu dan label `risk_score` (0-100) beserta `risk_tier` (Aman/Waspada/Rawan). Dipakai untuk melatih model Risk Prediction, dan `risk_tier`-nya langsung jadi dasar Visual Risk Indicator (🟢/🟡/🔴) tanpa dihitung ulang. |
| `features_labels_timeseries.csv` | Rangkaian snapshot `risk_score` berurutan waktu (per periode bulanan), khusus dipakai buat simulasi Continual Learning. Skala `risk_score`-nya beda sama `features_labels.csv` (basis normalisasi beda), jadi jangan dicampur langsung dalam satu perbandingan. |
| `model_config.csv` | Bukan dataset fitur — satu baris parameter (`T_LOW`, `T_HIGH`, `half_life_days`, `radius_km`, dst) yang dipakai bentuk dua dataset di atas. Dipakai buat konsistensi threshold Visual Risk Indicator dan dokumentasi reproduksi hasil. |
| `safe_places.csv` | Daftar 751 titik tempat aman (minimarket, apotek, pos polisi, rumah sakit, stasiun CTA) lengkap koordinatnya. Dipakai langsung untuk fitur Safe Place Locator. |