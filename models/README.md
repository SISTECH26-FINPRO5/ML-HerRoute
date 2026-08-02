## 📁 Output Model — Checkpoint & Registry (models/) - Checkpoint 2

Folder `models/` dihasilkan dari notebook `Checkpoint2.ipynb` (Section 8–9).
Karena ukuran file cukup besar, file tidak disimpan secara langsung di dalam repositori ini. Dapat mengunduh file lengkap melalui link Google Drive di bawah ini, atau generate ulang sendiri dengan menjalankan notebook dari awal dengan Google Collab.

* 🔗 **Link Google Drive:** [Download `models/` di Sini](https://drive.google.com/drive/folders/1RxLJijSwCZ5zz5mAtaN01km8pV55QYlB?usp=drive_link) 

#### Panduan Penempatan File:
Setelah mengunduh file/folder dari Google Drive:
1. Ekstrak file jika berformat zip/rar.
2. Letakkan folder hasil tersebut di root proyek, sejajar dengan notebook:
```text
    ML-HerRoute/
    └── models/    <-- Letakkan di sini
```

---

### Isi `models/` — Riwayat Checkpoint & Model Registry

Ini adalah **log lengkap proses training**, bukan artefak buat deployment. Isinya setiap versi model yang pernah dihasilkan selama simulasi continual learning, baik dari 3 strategi retraining (`cumulative`, `sliding_window`, `finetune`), baik versi yang **dipromosikan jadi champion** maupun yang **tidak** — semuanya tetap dicatat, bukan dibuang, supaya riwayatnya bisa ditelusuri kapan saja.

| File | Isi |
| :-- | :-- |
| `model_{strategy}_v{version}.joblib` | Satu file per versi model, per strategi. Isinya `dict`: `{"model": ..., "cell_target_map": ..., "global_mean_val": ...}` — model beserta konteks encoding yang dipakainya saat itu, jadi bisa langsung dipakai ulang tanpa retrain. |
| `registry_v2.csv` | Satu baris per keputusan (`initial_champion` / `promoted` / `kept_champion` / `skip_retrain_no_drift`), lintas ketiga strategi. Kolom-kolomnya: `version`, `batch_index`, `strategy`, `train_size`, `drift_detected`, `drift_report`, `metrics` (MAE/RMSE/R²), `champion_metrics_before`, `decision`, `checkpoint_path` (nunjuk ke file `.joblib` di atas), `timestamp`, `model_hyperparams`, `drift_trigger`. |

**Kegunaan:** audit & debugging. Jika ada pertanyaan "kenapa model versi ini dipromosikan", "data apa yang dipakai versi v3 strategi finetune", tinggal cek baris terkait di `registry_v2.csv`, gak perlu bongkar kode training.