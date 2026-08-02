## 📁 Output Model — Paket Deployment (export/) - Checkpoint 2

Folder `export/` dihasilkan dari notebook `Checkpoint2.ipynb` (Section 11).
Karena ukuran file cukup besar, file tidak disimpan secara langsung di dalam repositori ini. Dapat mengunduh file lengkap melalui link Google Drive di bawah ini, atau generate ulang sendiri dengan menjalankan notebook dari awal dengan Google Collab.

* 🔗 **Link Google Drive:** [Download `export/` di Sini](https://drive.google.com/drive/folders/1pT9U3DbR8GoWbHWNVN1HpH78CPLt40dU?usp=drive_link) 

#### Panduan Penempatan File:
Setelah mengunduh file/folder dari Google Drive:
1. Ekstrak file jika berformat zip/rar.
2. Letakkan folder hasil tersebut di root proyek, sejajar dengan notebook:
```text
    ML-HerRoute/
    └── export/    <-- Letakkan di sini
```

---

### Isi `export/` — Paket Deployment Final (untuk konsumsi BE)

Ini adalah **hasil akhir siap pakai**, cuma satu model champion terpilih beserta semua konteks pendukung yang dibutuhkan fitur-fitur aplikasi (Risk Indicator, Safe Place Locator, Safe Route Recommendation). Tujuannya supaya BE tinggal `joblib.load()` semuanya sekali saat startup, tanpa perlu training ulang atau akses ke notebook/dataset mentah sama sekali.

| File | Isi | Dipakai untuk |
| :-- | :-- | :-- |
| `champion_model.joblib` | Model regresi final terpilih (prediktor `risk_score`) | Semua fitur berbasis prediksi |
| `model_context.joblib` | `dict`: `cell_target_map`, `global_mean_val`, `T_LOW`, `T_HIGH`, `GRID_DECIMALS`, `FEATURE_COLS_V2` | Menyusun fitur (`assemble_features_v2_dynamic`) & menentukan tier warna risiko |
| `cell_hist_lookup.joblib` | `dict` berisi `cell_hist_lookup` (histori kejadian per sel Chicago asli) | `snap_to_nearest_cell()` — nempelin titik ke sel Chicago terdekat |
| `remap_bounds.joblib` | `dict`: `JKT_LAT_RANGE`, `JKT_LON_RANGE`, `CHI_LAT_RANGE`, `CHI_LON_RANGE` | `remap_coord()` — proyeksi koordinat Jakarta ke rentang yang dikenali model |
| `risk_graph_source.joblib` | `dict` berisi `df_for_graph` (`cell_id`, `lat_r`, `lon_r`, `risk_score`) — **data mentah**, bukan graf jadi | Dipakai untuk bangun ulang graf via `build_risk_graph()` + `connect_components()` saat BE startup |
| `safe_places.csv` | Daftar tempat aman (nama, tipe amenity, koordinat) | `find_nearest_safe_places()` |
| `model_config.csv` | Threshold `T_LOW`/`T_HIGH`, `grid_decimals` | Konsistensi tier & pembulatan koordinat dengan notebook |

**Catatan penting untuk BE:**
- `risk_graph_source.joblib` **bukan** graf yang sudah jadi, cuma datanya. Fungsi `build_risk_graph()` (+ helper `connect_components()`, yang menyambungkan klaster-klaster sel yang terisolasi) harus tetap dipanggil sekali saat aplikasi start, hasilnya baru di-cache di memory (jangan dibangun ulang tiap request).
- Semua koordinat Jakarta yang masuk lewat endpoint **wajib** melalui `remap_coord()` dulu sebelum dipakai fungsi lain, karena model dilatih dari data Chicago. Setiap respons prediksi harus menyertakan flag `is_mock: true` supaya FE bisa tampilkan disclaimer ke pengguna.