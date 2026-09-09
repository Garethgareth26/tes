
# STUDY REPORT: STEP 1 — SUPERVISED SURFACE DEFECT CLASSIFICATION
**Industrial Quality Inspection using Deep Transfer Learning (ResNet-18)**

---

## Slide 1: Title & Executive Summary

* **Project Title**: Automated Surface Defect Classification on Hot-Rolled Steel Plates
* **Core Objective**: Membangun end-to-end pipeline supervised deep learning dari dataset inspection, 3-way data splitting, transfer learning, evaluasi metrik multi-class, hingga physical error analysis.
* **Key Results**:
  * **Test Accuracy**: **99.17%** (357 dari 360 data uji ditebak dengan benar).
  * **Macro F1-Score**: **0.9917** (kinerja merata di semua kelas tanpa bias).
  * **Training Time**: **1 menit 7 detik** (10 epoch pada GPU Google Colab).
  * **Error Analysis**: Model hanya salah pada 3 gambar, dan ketiganya terjadi pada ambiguitas morfologi visual dengan confidence margin yang sangat tipis (~52% vs 46%).

---

## Slide 2: Background & Problem Formulation

* **Industrial Context**:
  * Inspeksi manual oleh manusia di pabrik baja rentan terhadap *fatigue* (kelelahan mata), subjektivitas, dan kecepatan yang lambat.
  * Klasifikasi otomatis berbasis Computer Vision diperlukan untuk mendeteksi jenis cacat secara *real-time*.
* **Why Multi-Class Classification (Bukan sekadar OK vs NG)?**:
  * Membedakan jenis cacat (*root-cause analysis*) memungkinkan tim produksi mengetahui mesin/proses mana yang bermasalah (misalnya: *scratches* disebabkan oleh gesekan roller, sedangkan *inclusion* disebabkan oleh kontaminasi material peleburan).
* **Defect Classes (6 Jenis Cacat)**:
  1. `crazing` (retakan halus seperti jaring laba-laba)
  2. `inclusion` (kontaminasi partikel/material asing)
  3. `patches` (bercak/noda tidak teratur)
  4. `pitted_surface` (permukaan berlubang/bintik kasar)
  5. `rolled-in_scale` (kerak tergilas saat proses rolling)
  6. `scratches` (goresan linier)

---

## Slide 3: Dataset Inspection & Characteristics

* **Dataset**: NEU Surface Defect Database (Northeastern University).
* **Spesifikasi Teknis**:
  * Format: Grayscale images (resolusi asli 200×200 pixel).
  * Total Populasi Data: **1.800 gambar**.
  * Distribusi Kelas: Masing-masing kelas memiliki tepat **300 gambar** (*perfectly balanced dataset*).
* **Temuan Struktur Folder Kaggle**:
  * `train/images/`: 1.440 gambar (240 per class = 80%).
  * `validation/images/`: 360 gambar (60 per class = 20%).
  * *Insight Metodologi*: Menghindari asumsi deskripsi web dengan melakukan verifikasi kode langsung (`os.walk` dan penghitungan jumlah file).

---

## Slide 4: Experimental Methodology & 3-Way Split

```
                         TOTAL DATASET (1.800 Images)
                                      │
            ┌─────────────────────────┴─────────────────────────┐
            ▼                                                   ▼
     Folder TRAIN (1.440)                             Folder VALIDATION (360)
            │                                                   │
    ┌───────┴────────┐                                          │
    ▼                ▼                                          ▼
Train Set         Val Set                                   Test Set
(1.152 Images)   (288 Images)                              (360 Images)
80% of Train     20% of Train                              Untouched Final Exam
(Augmented)      (Clean)                                   (Clean)
```

* **Pemisahan 3 Subset (Metodologi Anti-Data Leakage)**:
  1. **Train Set (1.152 gambar / 64%)**: Digunakan model untuk memperbarui bobot via backpropagation. Diberi **Data Augmentation** (Horizontal Flip, Vertical Flip, Random Rotation ±15°).
  2. **Validation Set (288 gambar / 16%)**: Digunakan untuk memantau performa antar-epoch, pemilihan checkpoint model terbaik, dan deteksi dini *overfitting*. **Tidak diaugmentasi acak**.
  3. **Test Set (360 gambar / 20%)**: Diambil dari folder terpisah yang **steril** dan tidak pernah disentuh selama training maupun hyperparameter tuning.

---

## Slide 5: Model Architecture & Transfer Learning

* **Backbone Architecture**: **ResNet-18** (Pretrained on ImageNet-1k).
* **Alasan Pemilihan ResNet-18**:
  * *Residual Connections (Skip Connections)* mencegah masalah *vanishing gradient*.
  * Bobot pretrained ImageNet sudah sangat terlatih mengenali fitur visual dasar (*edges, corners, textures, gradients*) sehingga konvergensi sangat cepat pada dataset kecil (1.152 gambar).
  * Ringan dan efisien untuk kebutuhan komputasi industri (*edge computing / inference latency* rendah).
* **Modifikasi Head Classifier**:
  * Lapisan Fully Connected terakhir diganti:
    $$\text{model.fc} = \text{nn.Linear}(\text{in\_features}=512, \text{out\_features}=6)$$
* **Training Hyperparameters**:
  * Optimizer: `AdamW` (Learning Rate = $1 \times 10^{-4}$, Weight Decay = $1 \times 10^{-2}$)
  * Loss Function: `CrossEntropyLoss`
  * Batch Size: 32
  * Epochs: 10

---

## Slide 6: Training Dynamics & Learning Curves

* **Durasi Total**: 1 menit 7 detik di GPU Colab (Tesla T4).
* **Ringkasan Progres Epoch**:
  * **Epoch 1**: Train Acc = 90.28% | Val Acc = **97.57%** (Transfer learning langsung bekerja cepat).
  * **Epoch 2**: Train Acc = 98.96% | Val Acc = **99.31%**.
  * **Epoch 3**: Train Acc = 99.22% | Val Acc = **100.00%** (Checkpoint terbaik tersimpan).
  * **Epoch 4-10**: Loss konvergen stabil mendekati nol (Val Loss akhir = 0.0017).
* **Analisis Kurva Belajar (Learning Curves)**:
  * **Karakteristik Sehat**: Tidak terjadi divergensi (*overfitting*), kurva loss validasi menurun beriringan dengan loss training.
  * **Fenomena Val Loss < Train Loss di Epoch 1**: Terjadi karena data training sengaja dipersulit dengan augmentasi rotasi dan flip acak, sedangkan data validasi dievaluasi dalam kondisi gambar bersih.

---

## Slide 7: Evaluasi Final pada Independent Test Set

Hasil pengujian pada **360 gambar** yang belum pernah dilihat sama sekali:

| Class | Precision | Recall | F1-Score | Support (Jumlah Data Uji) |
| :--- | :---: | :---: | :---: | :---: |
| **crazing** | 1.0000 | 0.9833 | 0.9916 | 60 |
| **inclusion** | 0.9833 | 0.9833 | 0.9833 | 60 |
| **patches** | 0.9836 | 1.0000 | 0.9917 | 60 |
| **pitted_surface** | **1.0000** | **1.0000** | **1.0000** | 60 |
| **rolled-in_scale**| **1.0000** | **1.0000** | **1.0000** | 60 |
| **scratches** | 0.9833 | 0.9833 | 0.9833 | 60 |
| **Accuracy** | | | **0.9917** | **360** |
| **Macro Avg** | **0.9917** | **0.9917** | **0.9917** | **360** |

* **Highlight Performa**:
  * `pitted_surface` dan `rolled-in_scale` meraih akurasi sempurna (**100% Precision dan 100% Recall**).
  * Semua kelas memiliki F1-Score di atas **0.983**.

---

## Slide 8: Confusion Matrix Breakdown

```
ACTUAL \ PREDICTED  crazing  inclusion  patches  pitted  scale  scratches
crazing               59         0         1       0       0        0
inclusion              0        59         0       0       0        1
patches                0         0        60       0       0        0
pitted_surface         0         0         0      60       0        0
rolled-in_scale        0         0         0       0      60        0
scratches              0         1         0       0       0       59
```

* **Temuan Diagonal**: 357 dari 360 data berada tepat di diagonal utama (tebakan benar).
* **3 Kasus Misklasifikasi**:
  1. 1 sampel `crazing` terprediksi `patches`.
  2. 1 sampel `inclusion` terprediksi `scratches`.
  3. 1 sampel `scratches` terprediksi `inclusion`.
* *Catatan Penting*: Terjadi **kebingungan simetris (*symmetric confusion*)** antara `inclusion` dan `scratches`.

---

## Slide 9: Deep-Dive Error Analysis (Visual & Physical Investigation)

Menganalisis 3 gambar yang gagal diprediksi dengan benar:

### 1. Actual: Crazing (34.5%) ➔ Predicted: Patches (63.9%)
* **Analisis Visual**: Di samping garis retakan halus, terdapat bercak-bercak gelap (*dark blotches*) yang kontras di bagian kanan tengah dan kiri bawah.
* **Penyebab**: Lapisan konvolusi mendeteksi area bercak gelap sebagai fitur dominan (*patches*), sehingga model ragu-ragu menentukan apakah ini retakan atau bercak noda.

### 2. Actual: Inclusion (46.1%) ➔ Predicted: Scratches (52.3%)
* **Analisis Visual**: Inklusi material asing tersebut mengalami deformasi mekanis saat baja digilas panas (*hot-rolling*), sehingga bentuknya tertarik memanjang lurus secara vertikal.
* **Penyebab**: Morfologi inklusi yang memanjang secara geometri identik dengan goresan (*scratch*). Confidence model terpecah hampir 50:50 (**52.3% vs 46.1%**).

### 3. Actual: Scratches (42.3%) ➔ Predicted: Inclusion (53.8%)
* **Analisis Visual**: Goresan di bagian atas tidak berupa garis menerus, melainkan bintik-bintik putih diskrit akibat pantulan cahaya lampu inspeksi.
* **Penyebab**: Bintik-bintik diskrit tersebut dibaca oleh model sebagai inklusi/kotoran partikel asing. Confidence model kembali terpecah ketat (**53.8% vs 42.3%**).

---

## Slide 10: Industrial Implications & Deployment Strategy

* **Key Takeaway 1 — Model Tidak Overconfident**:
  Pada semua kesalahan, probabilitas model berkisar di angka **52% - 63%**, menandakan model menyadari adanya ambiguitas visual pada sampel tersebut.
* **Key Takeaway 2 — Rejection Threshold (Human-in-the-Loop)**:
  * Di lini produksi nyata, dapat diterapkan aturan:
    $$\text{IF } \max(P) < 0.70 \implies \text{Tandai untuk Review Operator Manusia}$$
  * Dengan mekanisme ini, sistem mencapai **100% akurasi otomatis** pada data dengan confidence tinggi, sekaligus mencegah lolosnya produk cacat.
* **Next Roadmap (Next Step)**:
  * Melanjutkan ke **Step 2: Object Detection (YOLO / Bounding Box)** untuk melokalisasi koordinat pasti dan menghitung dimensi luas cacat baja.
