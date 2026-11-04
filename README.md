# Klasifikasi Kualitas Udara Dua Titik di Malaysia dengan SVM dan Logistic Regression Berbasis Big Data (Spark dan HDFS)

**Mahadata (IF25-40402) | Kelompok 07 | Kelas Mahadata R | Institut Teknologi Sumatera**

Anggota:
- Muhammad Nelwan Fakri (122140173)
- Ali Akbar (123140162)
- Reyhan Oktavian Putra (123140202)
- Yeremia Situmorang (123140048)

## Deskripsi
Klasifikasi kategori kualitas udara (AQI, PM2.5, standar EPA) per jam pada dua titik pemantauan OpenAQ
di Malaysia (Taman Tun Dr. Ismail dan Tambulaung), 1 Januari - 28 Oktober 2026.
Prototipe Minggu 10 membandingkan **Logistic Regression** dan **SVM linear** (`OneVsRest(LinearSVC)`) memakai Spark MLlib.
SVM kernel RBF tidak tersedia di MLlib, sehingga prototipe memakai SVM linear.

## Isi repositori
| Berkas | Fungsi |
|---|---|
| `docker-compose.yml` | Lingkungan: HDFS (namenode, datanode) dan Jupyter + Spark |
| `data/GabunganDataset_BigData.csv` | Data mentah gabungan (format long, 71.655 baris) |
| `data/GabunganDataset_BigData_eda.csv` | Data wide hasil EDA Minggu 9 (pembanding validasi) |
| `notebooks/Kelompok07_Persiapan_Data.ipynb` | Pembersihan, pivot, label AQI, fitur lag, simpan Parquet ke HDFS |
| `notebooks/Kelompok07_Minggu10_Prototipe_MLlib.ipynb` | Pipeline MLlib, latih, evaluasi, analisis kesalahan |
| `notebooks/confusion_matrix_*.png` | Gambar confusion matrix data uji |

## Cara menjalankan (Windows, Docker Desktop dengan WSL2)
1. Pasang Docker Desktop dan pastikan statusnya berjalan.
2. Di PowerShell, masuk ke folder repositori, lalu jalankan:
   ```
   docker compose up -d
   docker compose ps
   ```
   Harus ada 3 container aktif (namenode, datanode, jupyter).
3. Unggah data ke HDFS:
   ```
   docker exec namenode hdfs dfs -mkdir -p /openaq/raw
   docker exec namenode hdfs dfs -mkdir -p /openaq/clean
   docker exec namenode hdfs dfs -chmod -R 777 /openaq
   docker exec namenode sh -c "hdfs dfs -put -f /data/*.csv /openaq/raw/"
   docker exec namenode hdfs dfs -ls /openaq/raw
   ```
4. Buka Jupyter di `http://localhost:8888` (token: `kelompok07`), folder `work`.
5. Jalankan berurutan:
   1. `Kelompok07_Persiapan_Data.ipynb` (menghasilkan Parquet di `/openaq/clean/aqi_fitur`)
   2. `Kelompok07_Minggu10_Prototipe_MLlib.ipynb` (*Kernel -> Restart & Run All*)

HDFS UI: `http://localhost:9870`.

## Ringkasan hasil (data uji, Sep-Okt 2026)
| Model | Accuracy | Macro-F1 |
|---|---|---|
| Logistic Regression | 0,671 | 0,667 |
| SVM linear (OvR LinearSVC) | 0,526 | 0,533 |

Split berdasarkan waktu (latih Jan-Agu, uji Sep-Okt), parameter default, bobot kelas lewat `weightCol`.
Analisis kesalahan ada di bagian akhir notebook prototipe.
