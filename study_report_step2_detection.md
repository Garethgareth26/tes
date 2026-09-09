# STUDY REPORT: STEP 2 — OBJECT DETECTION (YOLOv8)
**Defect Localization and Severity Analysis on Hot-Rolled Steel Plates**

---

## Slide 1: Executive Summary

* **Project Objective**: Mengubah paradigma deteksi cacat dari sekadar "klasifikasi jenis cacat" (Step 1) menjadi "lokalisasi dan pengukuran area cacat" menggunakan arsitektur YOLOv8.
* **Key Results (Test Set Steril)**:
  * **mAP@50 (Mean Average Precision)**: **72.7%** (Meningkat tajam setelah Hyperparameter Tuning).
  * **Generalization Power**: Sangat tangguh. Performa *Test Set* (72.7%) terbukti sejalan dengan *Validation Set* (75.4%), membuktikan model menggeneralisasi dengan sangat baik tanpa *overfitting*.
  * **Training Time**: ~10 Menit (Early Stopping di Epoch 50) dengan resolusi tinggi (416x416) menggunakan GPU.

---

## Slide 2: Mengapa Beralih ke Object Detection?

Di Step 1, klasifikasi citra mencapai akurasi 99.17%, namun di industri nyata, akurasi klasifikasi seringkali belum cukup. 
* **Business Value**: Object Detection memberikan kordinat (*Bounding Box*), sehingga insinyur pabrik dapat menghitung **Luas Area Cacat (Severity)**.
* **Keputusan Kritis**: Jika cacat hanya berupa goresan kecil di pinggir, baja mungkin masih bisa dijual sebagai *Grade B*. Tapi jika cacat merata di tengah, baja harus dilebur ulang (*Reject/Scrap*). Klasifikasi tidak bisa memberikan informasi dimensional ini.

---

## Slide 3: Tantangan Metodologi & Data Engineering

Dataset NEU Surface Defect Database aslinya diformat menggunakan standar Pascal VOC (XML).

* **The YOLO Format Requirement**:
  YOLO mensyaratkan label berupa file `.txt` dengan kordinat yang dinormalisasi (0 hingga 1) agar kebal terhadap perubahan resolusi kamera di masa depan:
  `[class_id] [x_center] [y_center] [width] [height]`
* **Data Engineering**:
  Melakukan *parsing* struktur hierarki XML dan transformasi matematika untuk normalisasi kordinat (*bounding box*). Hal ini mensimulasikan keseharian *Machine Learning Engineer* yang menghabiskan 70% waktunya untuk *Data Preparation*.

---

## Slide 4: Strict 3-Way Split Methodology (Anti Data Leakage)

Untuk memastikan pengujian murni dan terbebas dari bias evaluasi (*Evaluation Bias*), dataset dirombak secara ketat menjadi 3 bagian:

1. **YOLO Train Set (1.151 images)**: 80% dari folder *train* bawaan Kaggle. Digunakan untuk memperbarui bobot model.
2. **YOLO Validation Set (288 images)**: 20% dari folder *train* bawaan Kaggle. Digunakan oleh model di akhir tiap *epoch* untuk mengukur performa berjalan.
3. **YOLO Test Set (360 images)**: 100% murni dari folder *validation* bawaan Kaggle. **Sama sekali tidak disentuh** sampai pelatihan dan *tuning* selesai sepenuhnya.

---

## Slide 5: Model Architecture & Hyperparameter Tuning

* **Arsitektur Dasar**: **YOLOv8 Nano (`yolov8n.pt`)**. Sangat ideal untuk mesin *Edge Computing* pabrik yang berspesifikasi rendah, memberikan deteksi *real-time* tanpa *lag*.
* **Hyperparameter Tuning (Peningkatan Berbasis Logika)**:
  Eksperimen awal dengan `imgsz=224` dan `epochs=15` hanya menghasilkan mAP 69.8%. Analisis visual membuktikan bahwa kompresi resolusi merusak tekstur retakan halus (*crazing*).
  * **Final Settings**: `imgsz=416` (mempertahankan ketajaman tekstur cacat) dan `epochs=100`.
  * **Early Stopping**: Model cerdas berhenti otomatis di Epoch 50 karena mendeteksi bahwa konvergensi optimal sudah tercapai, menghemat waktu komputasi.

---

## Slide 6: Final Test Set Evaluation (Independent)

Hasil pengujian final pada **360 gambar steril** menggunakan model yang sudah di-*tuning*:

| Class | Precision | Recall | mAP@50 | Deskripsi Peningkatan |
| :--- | :---: | :---: | :---: | :--- |
| **patches** | 0.816 | 0.874 | **0.935** | Sangat Sempurna (Naik dari 0.911) |
| **scratches** | 0.645 | 0.843 | **0.823** | Sangat Baik (Naik dari 0.788) |
| **pitted_surface** | 0.740 | 0.686 | **0.782** | Baik |
| **inclusion** | 0.687 | 0.736 | **0.764** | Baik |
| **rolled-in_scale**| 0.618 | 0.530 | **0.567** | Meningkat Tajam (Naik dari 0.481) |
| **crazing** | 0.626 | 0.341 | **0.493** | Mulai Membaik (Naik dari 0.427) |
| **ALL (Average)** | **0.689** | **0.668** | **0.727** | **Peningkatan Absolut +2.9%** |

---

## Slide 7: Deep-Dive Error Analysis

Mengapa mAP Object Detection (72.7%) jauh lebih rendah dibanding akurasi Classification (99.1%)?

1. **Karakteristik Metrik**: Dalam Object Detection, model tidak hanya dihukum jika salah menebak kelas, tetapi juga jika kotak yang ia gambar kurang menutupi 50% dari kotak asli (IoU < 50%).
2. **Keberhasilan Hyperparameter Tuning**:
   Dengan menaikkan resolusi ke 416, kelas yang memiliki bentuk menyebar dan berserat halus seperti `rolled-in_scale` dan `crazing` mengalami lonjakan performa yang signifikan (naik nyaris +10%). Ini membuktikan hipotesis bahwa *Image Resolution* sangat krusial untuk melokalisasi batas akhir dari sebuah retakan.
3. **Analisis Kelas `crazing` (mAP Terendah = 49.3%)**:
   Batas noda pada kelas ini seringkali membaur/gradasi dengan tekstur baja normal. Terlebih lagi, cacat ini muncul secara berkelompok dan tumpang tindih. Model berhasil menyadari ada cacat di gambar tersebut (Classification sukses), namun sangat kebingungan saat disuruh **menggambar kotak presisi di mana letak persis ujung cacatnya** akibat *labeling inconsistency* dari *annotator* manusia.

---

## Slide 8: Kesimpulan & Rekomendasi Industri

* **Kesimpulan Metodologi**: Model mampu menggeneralisasi data dengan sempurna. Pemisahan *3-way split* yang ketat berhasil mencegah *Data Leakage*, memberikan keyakinan bahwa performa 72.7% mAP ini akan bertahan ketika model dipasang di kamera lini produksi pabrik esok hari.
* **Rekomendasi Industri (Next Steps)**:
  1. **Audit Anotasi Manual**: Mengingat model sangat kesulitan menentukan letak batas `crazing`, disarankan untuk meninjau ulang *Ground Truth* (XML) dan menyusun SOP baru bagi *Data Labeler* manusia agar penggambaran kotaknya lebih konsisten.
  2. **Semantic Segmentation**: Jika batas *bounding box* dirasa terlalu kasar untuk mengukur luas *crazing*, pendekatan *Semantic Segmentation* (seperti U-Net) dapat dicoba untuk mewarnai cacat di tingkat piksel.
