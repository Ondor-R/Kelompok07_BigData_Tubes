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

## Arsitektur singkat
```
CSV (data/) --> HDFS (/openaq/raw) --> PySpark (bersihkan, pivot, label AQI, fitur lag)
            --> Parquet (HDFS /openaq/clean/aqi_fitur) --> Spark MLlib (Logistic Regression, SVM linear)
            --> evaluasi (accuracy, precision, recall, F1, macro-F1, confusion matrix)
```
Semua komponen berjalan di Docker pada satu komputer: HDFS single-node (namenode dan datanode) dan Jupyter yang sudah berisi Spark (mode lokal).

## Isi repositori
| Berkas | Fungsi |
|---|---|
| `docker-compose.yml` | Lingkungan: HDFS (namenode, datanode) dan Jupyter + Spark |
| `data/GabunganDataset_BigData.csv` | Data mentah gabungan (format long, 71.655 baris) |
| `data/GabunganDataset_BigData_eda.csv` | Data wide hasil EDA Minggu 9 (pembanding validasi) |
| `notebooks/Kelompok07_Persiapan_Data.ipynb` | Pembersihan, pivot, label AQI, fitur lag, simpan Parquet ke HDFS |
| `notebooks/Kelompok07_Minggu10_Prototipe_MLlib.ipynb` | Pipeline MLlib, latih, evaluasi, analisis kesalahan |
| `notebooks/confusion_matrix_*.png` | Gambar confusion matrix data uji |

## Prasyarat
| Kebutuhan | Keterangan |
|---|---|
| Sistem operasi | Windows 10/11 (diuji), macOS, atau Linux |
| RAM | Minimal 8 GB (disarankan 16 GB atau lebih). Diuji pada PC 32 GB |
| Ruang disk | Sekitar 10 GB kosong (image Docker, terutama Jupyter + Spark, berukuran besar) |
| Virtualisasi | Aktif di BIOS (umumnya sudah aktif) |
| Perangkat lunak | Docker Desktop (Windows/macOS) atau Docker Engine + plugin Compose (Linux) |
| Internet | Diperlukan saat pertama kali menjalankan untuk mengunduh image (sekitar 10-30 menit) |

## Setup lingkungan

### 1. Pasang Docker
- **Windows:** unduh Docker Desktop dari docker.com, pasang dengan opsi **WSL 2** tercentang, restart jika diminta, lalu buka Docker Desktop dan tunggu status "Engine running" (ikon hijau).
- **macOS:** pasang Docker Desktop sesuai chip (Apple Silicon atau Intel). Pada Apple Silicon, image HDFS berjalan lewat emulasi sehingga lebih lambat.
- **Linux:** pasang Docker Engine dan plugin Compose lewat panduan resmi distribusi masing-masing.

Cek pemasangan di terminal (PowerShell di Windows):
```
docker --version
docker compose version
```
Kedua perintah harus menampilkan nomor versi.

### 2. Siapkan folder proyek
Kloning atau unduh repositori ini. Di Windows, letakkan di folder biasa (contoh `C:\aqi-bigdata`), **hindari** folder yang disinkronkan OneDrive karena sering bermasalah saat disambungkan ke container. Jika path mengandung spasi, pakai tanda kutip saat `cd`:
```
cd "D:\ITERA\Big Data\tubes\aqi-bigdata"
```
Struktur yang diharapkan:
```
aqi-bigdata/
  docker-compose.yml
  data/        (dua file CSV)
  notebooks/   (dua notebook)
```
Pastikan nama file persis `docker-compose.yml` (bukan `.txt`).

### 3. Konfigurasi `docker-compose.yml`
```yaml
services:
  namenode:
    image: bde2020/hadoop-namenode:2.0.0-hadoop3.2.1-java8
    container_name: namenode
    environment:
      - CLUSTER_NAME=aqi
      - CORE_CONF_fs_defaultFS=hdfs://namenode:9000
      - CORE_CONF_hadoop_http_staticuser_user=root
      - HDFS_CONF_dfs_replication=1
    ports:
      - "9870:9870"
      - "9000:9000"
    volumes:
      - namenode_data:/hadoop/dfs/name
      - ./data:/data

  datanode:
    image: bde2020/hadoop-datanode:2.0.0-hadoop3.2.1-java8
    container_name: datanode
    environment:
      - CORE_CONF_fs_defaultFS=hdfs://namenode:9000
      - HDFS_CONF_dfs_replication=1
    depends_on:
      - namenode
    volumes:
      - datanode_data:/hadoop/dfs/data

  jupyter:
    image: quay.io/jupyter/pyspark-notebook:latest
    container_name: jupyter
    ports:
      - "8888:8888"
    environment:
      - JUPYTER_TOKEN=kelompok07
    volumes:
      - ./notebooks:/home/jovyan/work
      - ./data:/home/jovyan/data
    depends_on:
      - namenode

volumes:
  namenode_data:
  datanode_data:
```

Penjelasan komponen:

| Container | Fungsi | Port di komputer |
|---|---|---|
| `namenode` | Pengelola metadata HDFS. Folder `./data` terpasang sebagai `/data` agar CSV mudah diunggah | 9870 (antarmuka web), 9000 (HDFS) |
| `datanode` | Penyimpan blok data HDFS. Replikasi diatur 1 karena hanya ada satu datanode | - |
| `jupyter` | Jupyter Lab + Spark (mode lokal). Folder `./notebooks` terpasang sebagai `/home/jovyan/work` dan `./data` sebagai `/home/jovyan/data` | 8888 |

Data HDFS disimpan di *volume* Docker (`namenode_data`, `datanode_data`), sehingga tetap ada setelah container dimatikan.

### 4. Jalankan container
Di folder proyek:
```
docker compose up -d
docker compose ps
```
Pertama kali akan mengunduh beberapa GB image. Setelah selesai, `docker compose ps` harus menampilkan **3 container** (`namenode`, `datanode`, `jupyter`) berstatus `running`/`Up`. Tunggu sekitar 30 detik agar HDFS siap.

### 5. Verifikasi HDFS dan Jupyter
- **HDFS:** buka `http://localhost:9870`, menu *Datanodes*. Harus terlihat **1 datanode** aktif.
- **Jupyter:** buka `http://localhost:8888`, masukkan token `kelompok07`. Folder `work` berisi notebook, folder `data` berisi CSV.

### 6. Siapkan HDFS dan unggah data
```
docker exec namenode hdfs dfs -mkdir -p /openaq/raw
docker exec namenode hdfs dfs -mkdir -p /openaq/clean
docker exec namenode hdfs dfs -chmod -R 777 /openaq
docker exec namenode sh -c "hdfs dfs -put -f /data/*.csv /openaq/raw/"
docker exec namenode hdfs dfs -ls /openaq/raw
```
Perintah terakhir harus menampilkan **2 file CSV**. Perintah `chmod 777` wajib, karena Jupyter berjalan sebagai user berbeda dari `root` dan perlu izin tulis untuk menyimpan Parquet dan model.

Struktur HDFS yang dipakai proyek:

| Jalur HDFS | Isi |
|---|---|
| `/openaq/raw` | CSV mentah dan CSV hasil EDA |
| `/openaq/clean/aqi_fitur` | Parquet berisi label AQI dan fitur (dibuat notebook persiapan data) |
| `/openaq/models/lr`, `/openaq/models/svm` | Model MLlib terlatih (dibuat notebook prototipe) |

Spark di dalam container Jupyter membaca HDFS lewat alamat `hdfs://namenode:9000/...`.

## Menjalankan notebook
1. Buka Jupyter di `http://localhost:8888` dan masuk ke folder `work`.
2. Jalankan berurutan:
   1. `Kelompok07_Persiapan_Data.ipynb`: *Kernel -> Restart Kernel and Run All Cells*. Menghasilkan Parquet di `/openaq/clean/aqi_fitur` (13.135 baris).
   2. `Kelompok07_Minggu10_Prototipe_MLlib.ipynb`: *Kernel -> Restart Kernel and Run All Cells*. Melatih model, mengevaluasi, dan membuat `confusion_matrix_*.png` di folder `notebooks`.
3. Cek hasil di HDFS:
   ```
   docker exec namenode hdfs dfs -ls /openaq/clean/aqi_fitur
   docker exec namenode hdfs dfs -ls /openaq/models
   ```

## Menyalakan dan mematikan lagi (tanpa setup ulang)
Menyalakan kembali (data HDFS tetap ada):
```
cd "D:\ITERA\Big Data\tubes\aqi-bigdata"
docker compose up -d
docker compose ps
docker exec namenode hdfs dfs -ls /openaq/raw
```
Mematikan:
```
docker compose stop
```
Container tidak menyala sendiri setelah komputer dimatikan, jadi `docker compose up -d` perlu dijalankan lagi.

**Hindari** `docker compose down -v` dan fitur *Clean / Purge data* di Docker Desktop: keduanya menghapus volume sehingga data HDFS hilang. Perintah `docker compose down` tanpa `-v` aman (volume tetap ada).

**Mengulang dari nol (menghapus semua data HDFS):**
```
docker compose down -v
docker compose up -d
```
lalu ulangi langkah 6 dan jalankan kedua notebook.

## Pemecahan masalah
| Masalah | Penyebab dan solusi |
|---|---|
| `docker` tidak dikenali | Docker Desktop belum terpasang atau belum berjalan. Buka Docker Desktop dan tunggu status "Engine running" |
| `docker compose up` gagal karena port dipakai | Port 8888, 9870, atau 9000 sudah dipakai program lain. Ubah angka sebelah kiri pada `ports` (misalnya `"8889:8888"`) lalu buka alamat dengan port baru |
| `docker compose` tidak menemukan file | Terminal belum berada di folder yang berisi `docker-compose.yml`, atau nama file salah (`.txt`) |
| Datanode tidak muncul di `localhost:9870` | Tunggu 1 menit, lalu `docker compose restart datanode` |
| Pesan "safe mode" saat membaca HDFS | Tunggu 1 menit. Jika masih muncul: `docker exec namenode hdfs dfsadmin -safemode leave` |
| `Permission denied` saat menyimpan Parquet atau model | Izin belum diatur. Jalankan `docker exec namenode hdfs dfs -chmod -R 777 /openaq` |
| `Connection refused` ke `namenode:9000` di notebook | Container belum aktif. Cek `docker compose ps` dan jalankan `docker compose up -d` |
| `hdfs dfs -ls /openaq/raw` kosong atau folder tidak ada | CSV belum diunggah, atau folder `data` kosong. Isi folder `data`, lalu ulangi langkah 6 |
| Jupyter tidak bisa dibuka | Pastikan container `jupyter` berstatus `Up`, dan token yang dimasukkan `kelompok07` |
| Lambat atau container mati sendiri | RAM Docker kurang. Pada Windows (WSL2) tutup program berat, atau atur batas memori WSL lewat file `.wslconfig` |
| Folder proyek di OneDrive bermasalah | Pindahkan folder proyek ke lokasi yang tidak disinkronkan (misalnya `C:\aqi-bigdata`) |

## Ringkasan hasil (data uji, Sep-Okt 2026)
| Model | Accuracy | Macro-F1 |
|---|---|---|
| Logistic Regression | 0,671 | 0,667 |
| SVM linear (OvR LinearSVC) | 0,526 | 0,533 |

Split berdasarkan waktu (latih Jan-Agu, uji Sep-Okt), parameter default, bobot kelas lewat `weightCol`.
Analisis kesalahan ada di bagian akhir notebook prototipe.
