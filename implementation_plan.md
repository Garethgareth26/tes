# Implementation Plan: Step 3 - Semantic Segmentation

Misi kita di Step 3 adalah mengubah paradigma deteksi cacat dari sekadar "menggambar kotak" menjadi "mewarnai setiap piksel" yang merupakan bagian dari cacat. Ini sangat krusial untuk mengukur luas area cacat (*Severity*) dengan akurasi matematis 100%.

## User Review Required

Ada keputusan penting yang perlu kamu tentukan sebelum kita memulai kodenya di Colab. Mohon baca bagian **Model Architecture Options** di bawah dan beritahu saya pilihanmu.

## Proposed Changes / Workflow

Sesuai dengan *roadmap* yang kamu tetapkan, kita akan mengikuti alur ini secara disiplin:

### 1. Dataset Selection
*   **Kriteria**: Dataset kecil, sederhana, jelas antara OK vs NG, dan memiliki *Ground Truth Mask* (gambar hitam putih di mana putih adalah piksel cacat).
*   **Pilihan**: **Magnetic Tile Surface Defects Dataset** (Tersedia di Kaggle).
    *   **Alasan**: Dataset NEU baja kita sebelumnya tidak memiliki label piksel (hanya kotak XML). Dataset Magnetic Tile ini ukurannya kecil, konteksnya manufaktur (keramik magnetik), dan sangat sempurna untuk pemula di bidang segmentasi.

### 2. Annotation & Data Engineering
*   Menggabungkan gambar asli (jpg) dengan gambar *Mask* (png).
*   Melakukan normalisasi nilai piksel *Mask* menjadi angka biner: `0` (Background/Besi Normal) dan `1` (Cacat).

### 3. Split: Train / Validation / Test
*   Kita akan langsung menerapkan ilmu dari Step 2: **Strict 3-Way Split**. Kita tidak akan mengulang kesalahan *Test Set Leakage*.

### 4. Model Architecture Options (PILIH SALAH SATU)

> [!IMPORTANT]
> **Silakan pilih arsitektur yang ingin kamu pelajari di Step 3 ini:**
> 
> **Opsi A: Membangun U-Net dari Nol (PyTorch)**
> *   **Kelebihan**: U-Net adalah *Grandfather* dari semua model segmentasi medis dan industri. Membangunnya baris-demi-baris dengan PyTorch akan membuatmu paham 100% cara kerja algoritma segmentasi tingkat piksel.
> *   **Kekurangan**: Kodenya panjang dan kita harus menulis fungsi *Training Loop* sendiri (seperti di Step 1).
> 
> **Opsi B: Menggunakan YOLOv8-Seg (Ultralytics)**
> *   **Kelebihan**: Kodenya sangat pendek (hanya 3-4 baris seperti Step 2), *training*-nya cepat, dan performanya setara standar industri modern.
> *   **Kekurangan**: Kita tidak tahu apa yang terjadi di "balik layar". *Script* konversi poligon datanya juga sedikit merepotkan.

### 5. Objective Metrics
Di Segmentasi, kita tidak menggunakan mAP dari *Bounding Box*. Kita akan belajar 2 metrik baru:
*   **Pixel-Level IoU (Intersection over Union)**: Seberapa presisi tumpang-tindih piksel tebakan vs piksel asli.
*   **Dice Coefficient (F1-Score untuk Piksel)**: Rumus matematika untuk menyeimbangkan antara salah tebak piksel (FP) dan piksel cacat yang terlewat (FN).

### 6. Error Analysis
*   Kita akan melakukan *overlay* (menumpuk) warna tebakan model di atas gambar asli, dan melihat secara transparan di area piksel mana model tersebut sering "bocor" warnanya.

## Verification Plan

1.  Mendownload dataset Magnetic Tile via `kagglehub`.
2.  Mengeksplorasi gambar dan memplot *Mask* biner (hitam-putih) untuk verifikasi visual awal.
3.  Memilih model (U-Net / YOLO-Seg) lalu menjalankan *training*.
4.  Evaluasi pada data Test menggunakan metrik IoU dan Dice.
