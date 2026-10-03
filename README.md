## Dokumen Desain Awal (DESAIN_AWAL.md) - FINAL

### 1. Misi Projek
Sistem persepsi pada robot berfungsi untuk mendeteksi dan mengklasifikasikan rambu petunjuk arah (panah) serta penanda informasi (huruf_angka) di jalurnya secara real-time. Hasil klasifikasi ini digunakan oleh robot sebagai input utama pengambilan keputusan navigasi, seperti berbelok mengikuti arah panah atau melambat/berhenti saat mendeteksi tanda huruf/angka.

### 2. Kelas Objek
- **Kelas klasifikasi (2 kelas):** `huruf_angka` (96 gambar) dan `panah` (99 gambar) pada training set.
- **Pembagian:** Train: 195 | Val: 81 | Test: 83.
- **Label asli yang digabung:** Angka 1-9 dan huruf A-Z digabung menjadi `huruf_angka`. Panah kiri, kanan, atas, bawah digabung menjadi `panah`.

### 3. Kamera dan Dudukan
- **Kamera:** Kamera robot menggunakan resolusi input standar 224x224 RGB setelah resizing, dipasang pada tinggi sekitar 15-20 cm dari permukaan tanah dengan sudut inklinasi 10 derajat ke bawah, disesuaikan untuk jarak kerja optimal deteksi rambu antara 30 cm hingga 1 meter.

### 4. Unit Komputasi
- **Pelatihan:** Google Colab (T4 GPU).
- **Target Robot:** Jetson Nano / Smorphi, dikonfigurasi pada mode daya maksimum (10W) dengan RAM terbagi untuk GPU dan CPU.

### 5. Target Kinerja
- **Akurasi val:** >= 90% (Terpenuhi: **98.8%** pada Epoch 4, Balanced Acc: **98.3%**, Macro-F1: **0.986**).
- **Latensi inferensi:** <= 35 ms (Terpenuhi di GPU Colab: **8.7 ms** / 115.3 FPS, Belum terpenuhi di CPU Colab: **86.5 ms** / 11.6 FPS).

---

## Laporan Hasil Pengujian (README.md) - FINAL

### Ringkasan Hasil Utama

| Split | Jumlah Gambar | Akurasi | Balanced Acc | Macro-F1 | Salah Prediksi |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Val** | 81 | **98.8%** | **98.3%** | **0.986** | 1 |
| **Test** | 83 | **100.0%** | **100.0%** | **1.000** | 0 |
| **Tambahan (HP)** | 10 | **90.0%** | **90.0%** | **0.900** | 1 |

### Analisis Singkat Jawab Pertanyaan

1. **Mengapa memilih mode partial?**
   Mode `partial` dipilih karena jumlah data latih kita terbatas (195 gambar) dan domain citra rambu robot berbeda secara signifikan dari citra umum ImageNet. Dengan membekukan layer awal dan membuka blok akhir (unfreeze dari block 6), kita memanfaatkan detektor tepi dasar dari ImageNet sembari menyesuaikan detektor pola spesifik untuk rambu kita. Keterbatasannya adalah kita tidak memiliki pembanding empiris langsung terhadap performa mode `feature` atau `scratch` pada dataset ini.

2. **Bagaimana kurva train dan val?**
   Kurva menunjukkan konvergensi yang sangat cepat. Akurasi mulai stabil dan melampaui target 90% sejak epoch ke-4 (mencapai 98.8% pada val). Gap antara loss train dan loss val relatif sempit, yang menandakan bahwa augmentasi data seperti RandomResizedCrop dan ColorJitter berhasil mereduksi risiko overfitting.

3. **Kelas mana yang lebih sering salah?**
   Hanya ada 1 kesalahan prediksi pada data validation (kelas `huruf_angka` dengan sub-label White V terprediksi sebagai `panah`). Berdasarkan analisis `kesalahan.csv`, kesalahan ini disebabkan oleh rasio objek yang sangat kecil (hanya 1.1% dari frame) dikombinasikan dengan bentuk huruf 'V' yang memiliki kemiripan geometris bersudut tajam menyerupai ujung tanda panah.

4. **Dampak ketimpangan kelas & Mengapa akurasi saja tidak cukup?**
   Kelas `panah` lebih sedikit di training set. Penerapan `class_w` (bobot kelas) memastikan loss fungsi memberikan penalti lebih besar jika salah memprediksi kelas minoritas. Akurasi standar saja tidak cukup karena model yang naif menebak kelas mayoritas dapat memperoleh akurasi semu yang tinggi (baseline mayoritas mencapai 64.2% di val); oleh sebab itu, kita wajib memantau balanced accuracy dan macro-F1 (keduanya di atas 98%).

5. **Risiko data leakage dan bias latar?**
   Kebocoran data dicegah dengan melakukan split train/val/test berdasarkan ID sesi pemotretan, memastikan latar belakang atau kondisi pencahayaan yang sama tidak muncul di kedua set. Namun, bias latar belakang studio yang homogen masih tersisa dan berpotensi menurunkan performa di lingkungan nyata.
   - *Uji Tambahan HP:* Berhasil memprediksi 9 dari 10 gambar (90%). Satu kesalahan terjadi pada `huruf_5.jpg` (gambar huruf 'Y' berwarna merah di atas latar putih terang dengan tanaman hijau di jendela) yang salah diprediksi sebagai `panah` (confidence 53%). Ini menunjukkan bias latar luar jendela yang terlalu mencolok dapat mengacaukan model klasifikasi.

6. **Apakah latensi memenuhi anggaran?**
   Pada GPU Colab, latensi rata-rata adalah 8.7 ms (sangat aman di bawah batas anggaran 35 ms). Namun, pada CPU Colab latensinya mencapai 86.5 ms (melebihi jatah). Keterbatasannya adalah pengukuran dilakukan pada spesifikasi server cloud Colab, bukan pada prosesor mikro di robot fisik.

7. **Keterbatasan dan rencana perbaikan:**
   - Menambahkan kelas latar belakang kosong (background/negative class) agar frame tanpa rambu tidak dipaksa masuk ke kelas `panah` atau `huruf_angka`.
   - Melatih model dengan resolusi input yang lebih tinggi (misal 320x320) untuk membantu deteksi objek berukuran kecil.
   - Melakukan uji coba langsung serta kalibrasi model pada hardware Smorphi menggunakan data tangkapan kamera robot sendiri.
