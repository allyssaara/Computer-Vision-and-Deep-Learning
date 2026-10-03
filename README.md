# Computer-Vision-and-Deep-Learning
RET503

## Analisis singkat

**Hasil utama (percobaan akhir, EfficientNet-B0, mode `partial`, 10 epoch):**

| | Akurasi | Balanced acc | Macro-F1 | Salah prediksi |
|---|---|---|---|---|
| Val (81 gambar) | 98,8% | 98,3% | 0,986 | 1 |
| Test (83 gambar) | 100% | 100% | 1,000 | 0 |

Epoch terpilih: 4 (macro-F1 val tertinggi). Waktu latih sekitar 37 detik di GPU Colab. Parameter yang dilatih: 3,16 juta dari 4,01 juta.

**1. Pemilihan mode `partial`.**
Data latih hanya 195 gambar dan tampilan rambu (kotak bertanda berwarna) berbeda dari foto benda di ImageNet. Menurut matriks keputusan di materi (slide 10), data sedikit dengan domain berbeda mengarah ke fine-tuning parsial, sehingga blok akhir (`features[6:]`) dan classifier dilatih dengan learning rate 1e-4 dan 1e-3. Keterbatasannya: hanya satu mode yang dijalankan, jadi tidak ada bukti dari percobaan ini bahwa `partial` lebih baik daripada `feature` atau `scratch`. Pemilihannya didasarkan pada argumen dari materi, bukan perbandingan. Karena pelatihan hanya sekitar 37 detik, perbandingan tiga mode mudah dilakukan sebagai pengembangan.

**2. Kurva pelatihan.**
Loss train turun dari sekitar 0,65 ke 0,15 dan loss val dari 0,56 ke 0,145. Akurasi val naik dari 84% (epoch 1) ke 98,8% (epoch 4) lalu stabil di 97,5% pada epoch 5 sampai 10. Tidak terlihat overfitting: loss val tidak naik, dan akurasi train (95,8%) justru lebih rendah daripada val. Itu wajar, karena augmentasi hanya diterapkan pada train dan akurasi train dirata-rata selama epoch berjalan. Perbedaan epoch 4 (98,8%) dan epoch 5 sampai 10 (97,5%) hanya satu gambar dari 81, sehingga pemilihan epoch 4 tidak bermakna secara statistik. Loss train belum sepenuhnya mendatar di epoch 10.

**3. Kesalahan.**
Satu-satunya kesalahan ada di val: sebuah gambar `huruf_angka` (label asli `White V`) ditebak `panah`. Rasio luas tanda terhadap frame pada gambar itu hanya 0,011 (sekitar 1% frame), dengan latar lorong terang yang berbeda dari kebanyakan gambar. Dugaan penyebabnya: tanda terlalu kecil sehingga detail huruf hilang saat gambar diperkecil ke 224×224. Ini sejalan dengan temuan di data: seperempat gambar memiliki rasio tanda di bawah sekitar 0,013. Dengan hanya satu kesalahan, pola ini adalah dugaan, bukan kesimpulan.

**4. Kelas timpang dan metrik.**
Pada data penuh hasil tahap 1 (1132:179), model yang selalu menjawab `huruf_angka` akan mendapat sekitar 90,6% di val, sehingga akurasi tinggi tidak bisa dipakai sebagai bukti model baik. Karena itu data diseimbangkan (180:179), bobot kelas menjadi sekitar 1,0, dan baseline kelas mayoritas turun menjadi 51,3% (train), 64,2% (val), dan sekitar 61% (test). Dengan baseline itu, akurasi 98,8% dan 100% jauh lebih bermakna. Harga yang dibayar: tiap label asli tinggal sekitar 7 gambar (180 gambar untuk 25 huruf dan angka), sehingga cakupan tiap huruf/angka tipis.

**5. Data leakage dan bias latar.**
Pembagian dilakukan per sesi (blok 15 gambar berurutan menurut nama file), bukan per gambar, untuk mengurangi kemiripan antar frame di train dan test. Kemiripan antar frame tetap mungkin tersisa karena sesi hanya didekati dari urutan nama file. Pada delapan contoh acak per kelas, kedua kelas muncul dengan latar yang sama (dedaunan di atas dan lantai bata di bawah), jadi latar tidak otomatis membedakan kelas, tetapi ini belum membuktikan model bebas dari shortcut latar. Uji tambahan: [ISI: jumlah gambar, hasil benar/total, dan kesalahan yang muncul]. Gambar uji tambahan berasal dari proyek Roboflow yang sama (versi lain), bukan dari kamera atau lingkungan lain, sehingga uji ini tidak menunjukkan generalisasi ke kondisi baru, dan bisa memuat gambar yang sama atau mirip dengan data latih [CEK: pastikan gambar uji tambahan tidak sama dengan gambar di train].

**6. Latensi.**
Isi dari tabel latensi di atas. Pada pengukuran sebelumnya EfficientNet-B0 mencapai sekitar 8,7 ms di GPU Colab (di bawah jatah inferensi 35 ms) dan sekitar 86,5 ms di CPU Colab (di atas jatah) [CEK: ganti dengan angka dari tabel latensi terakhirmu]. Angka ini diukur di Colab, bukan di perangkat robot, jadi hanya perbandingan kasar. Apakah model memenuhi 15 FPS di robot bergantung pada perangkat target dan harus diukur di sana.

**7. Keterbatasan dan rencana perbaikan.**
- Val dan test kecil (81 dan 83 gambar): satu gambar sama dengan sekitar 1,2% akurasi. Nol kesalahan dari 83 gambar test hanya menunjukkan laju kesalahan sejati kemungkinan di bawah sekitar 3,6% (aturan tiga, tingkat keyakinan 95%), bukan bahwa model sempurna.
- Tanda kecil di dalam frame: coba input lebih besar (mis. 320) atau pendekatan dua tahap (detektor lalu klasifikasi pada potongan), dengan memperhatikan kenaikan latensi.
- Hanya dua kelas, sehingga frame tanpa tanda tetap dipaksa menjadi salah satunya. Perlu kelas latar atau ambang keyakinan.
- Seluruh data, termasuk uji tambahan, berasal dari proyek Roboflow yang sama dan bukan dari kamera robot sendiri. Perlu pengujian pada citra dari kamera dan lingkungan robot.
- Hanya satu mode dan satu seed. Perlu perbandingan mode lain dan beberapa pengulangan untuk melihat kestabilan hasil.
- `kondisi_cahaya` di metadata adalah estimasi otomatis dari kecerahan gambar, bukan pencatatan manual.

## Kesimpulan
- EfficientNet-B0 dengan fine-tuning parsial mengklasifikasikan frame rambu menjadi `huruf_angka` dan `panah` dengan akurasi val 98,8% (macro-F1 0,986) dan akurasi test 100% (83 gambar), jauh di atas baseline kelas mayoritas (sekitar 51 sampai 64%).
- Model stabil sejak epoch 4 sampai 5, tanpa tanda overfitting, dengan waktu latih sekitar 37 detik.
- Satu-satunya kesalahan berasal dari tanda yang sangat kecil di dalam frame, yang menunjukkan ukuran objek sebagai keterbatasan utama.
- Hasil ini berlaku untuk dataset ini saja. Ukuran val dan test yang kecil, dua kelas, dan data bukan dari kamera robot sendiri berarti generalisasi ke robot belum terbukti. Uji pada citra robot sendiri adalah langkah berikutnya.
- Latensi pada GPU memenuhi anggaran 35 ms; kepatuhan pada perangkat robot belum diukur.
