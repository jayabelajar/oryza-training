# Oryza Vision

Modul computer vision untuk mengklasifikasikan penyakit daun dan hama padi. Repository ini berisi notebook Google Colab, model TensorFlow Lite siap integrasi, metadata preprocessing, label, serta contoh gambar uji.

> Hasil model merupakan indikasi awal, bukan diagnosis definitif. Validasi dengan foto dari kondisi lapangan dan perangkat target tetap diperlukan.

## Model

| Tugas | Arsitektur | Kelas | Artefak |
|---|---|---:|---|
| Penyakit daun | MobileNetV3-Large | 5 | [`models/disease`](models/disease) |
| Hama | EfficientNet-B0 | 3 | [`models/pest`](models/pest) |

Kedua model menerima tensor `float32` dengan layout NHWC dan bentuk `[1, 224, 224, 3]`. Model menghasilkan *raw logits*, sehingga aplikasi perlu menjalankan softmax sebelum memilih kelas dengan skor tertinggi.

### Urutan label penyakit

1. `Bacterial_Blight`
2. `Brown_Spot`
3. `Healthy`
4. `Leaf_Blast`
5. `Leaf_Scald`

### Urutan label hama

1. `Brown_Planthopper`
2. `Rice_Leaf_Roller`
3. `Rice_Leafhopper`

Gunakan file `*_labels.json` sebagai sumber urutan label saat integrasi. Jangan mengurutkan label kembali secara alfabetis di aplikasi.

## Preprocessing

Model tidak menyertakan preprocessing di dalam graph. Input harus diproses dengan urutan berikut:

1. Baca gambar sebagai RGB dan terapkan orientasi EXIF.
2. Ubah ukuran sisi terpendek menjadi 256 piksel.
3. Ambil *center crop* 224 × 224 piksel.
4. Ubah nilai piksel ke skala `[0, 1]`.
5. Normalisasi setiap channel dengan rumus `(pixel / 255 - mean) / std`.

```text
mean = [0.485, 0.456, 0.406]
std  = [0.229, 0.224, 0.225]
```

Spesifikasi lengkap tersedia di:

- [`models/disease/disease_preprocessing.json`](models/disease/disease_preprocessing.json)
- [`models/pest/pest_preprocessing.json`](models/pest/pest_preprocessing.json)

## Struktur repository

```text
oryza-vision/
├── examples/                 # Gambar untuk uji manual
├── models/
│   ├── disease/              # Model, label, dan preprocessing penyakit
│   └── pest/                 # Model, label, dan preprocessing hama
├── notebooks/
│   ├── disease-training.ipynb
│   ├── pest-training.ipynb
│   ├── pest-model-comparison.ipynb
│   └── export-to-tflite.ipynb
├── reports/
│   └── conversion-report.json
└── README.md
```

## Menjalankan training

Notebook dirancang untuk Google Colab dan memasang dependensinya sendiri. Untuk menjalankan eksperimen:

1. Buka notebook yang dibutuhkan di Google Colab.
2. Pilih runtime GPU bila tersedia.
3. Tinjau konfigurasi pada cell awal, terutama epoch, batch size, seed, dan lokasi Google Drive.
4. Jalankan seluruh cell secara berurutan.
5. Simpan checkpoint, metrik, dan manifest hasil eksperimen.

Notebook utama:

- [`disease-training.ipynb`](notebooks/disease-training.ipynb) — kurasi data, training, dan evaluasi model penyakit.
- [`pest-training.ipynb`](notebooks/pest-training.ipynb) — training dan evaluasi EfficientNet-B0 untuk hama.
- [`pest-model-comparison.ipynb`](notebooks/pest-model-comparison.ipynb) — perbandingan MobileNetV3-Large dan EfficientNet-B0 pada split yang sama.
- [`export-to-tflite.ipynb`](notebooks/export-to-tflite.ipynb) — konversi checkpoint PyTorch ke TensorFlow Lite dan validasi hasil ekspor.

## Dataset

Notebook mengunduh dataset publik secara langsung dari Hugging Face:

- Penyakit: [Project-AgML/rice_leaf_disease_classification](https://huggingface.co/datasets/Project-AgML/rice_leaf_disease_classification)
- Hama: [Project-AgML/CT_PestNet_crop_pest_classification](https://huggingface.co/datasets/Project-AgML/CT_PestNet_crop_pest_classification)

Eksperimen penyakit mengambil lima dari enam kelas yang tersedia. Eksperimen hama mengambil maksimal 400 gambar per kelas untuk tiga kelas target. Kedua pipeline memakai seed `42` dan pembagian data train/validation/test sebesar 70%/15%/15% setelah proses kurasi.

Dataset tidak disimpan di repository. Periksa kembali lisensi dan ketentuan penggunaan pada halaman sumber sebelum mendistribusikan data atau model turunannya.

## Validasi ekspor

Menurut [`reports/conversion-report.json`](reports/conversion-report.json), kedua model TFLite lolos perbandingan terhadap model asal pada 10 input sintetis:

| Model | Ukuran | Selisih logit maksimum | Kesamaan top-1 |
|---|---:|---:|---:|
| Penyakit | 16,19 MB | `1.0192e-05` | 100% |
| Hama | 15,49 MB | `1.5050e-06` | 100% |

Validasi tersebut memeriksa konsistensi konversi, bukan akurasi pada data nyata. Sebelum rilis, ukur juga akurasi per kelas, macro F1, latensi, penggunaan memori, dan performa pada foto lapangan.

## Integrasi aplikasi

Saat memasang model di Flutter atau aplikasi mobile lain:

- Salin model, label, dan metadata preprocessing sebagai satu paket.
- Pastikan interpreter membaca input `float32` dalam urutan RGB.
- Terapkan preprocessing yang identik dengan metadata.
- Jalankan softmax pada logits, kemudian petakan indeks ke file label.
- Tangani orientasi kamera, foto buram, objek di luar kelas, dan confidence rendah.
- Bandingkan beberapa prediksi aplikasi dengan hasil notebook menggunakan gambar yang sama.

Contoh gambar tersedia di [`examples`](examples). Gambar tersebut hanya untuk pemeriksaan alur inferensi, bukan benchmark akurasi.

## Keterbatasan

- Split dilakukan pada tingkat gambar karena dataset tidak menyediakan identitas tanaman atau sesi pemotretan yang konsisten.
- Deduplikasi tidak menjamin seluruh gambar yang sangat mirip telah terpisah antar-split.
- Akurasi dataset uji tidak otomatis mewakili pencahayaan, latar, kamera, atau fase pertumbuhan di lapangan.
- Model hanya mengenali kelas yang tercantum dan dapat memberi confidence tinggi pada objek di luar distribusi training.

## Lisensi

Repository ini belum menyertakan lisensi proyek. Hak penggunaan dataset mengikuti ketentuan dari masing-masing sumber.
