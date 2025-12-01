# 🌾 Analisis Stabilitas Harga Beras vs Anomali Cuaca (Kendari 2025)

![Python](https://img.shields.io/badge/Python-3.10%2B-blue?style=for-the-badge&logo=python&logoColor=white)
![Data Analysis](https://img.shields.io/badge/Data_Analysis-Pandas_&_SciPy-orange?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-success?style=for-the-badge)

> **Project ini bertujuan untuk menginvestigasi apakah curah hujan harian memiliki dampak signifikan terhadap fluktuasi harga beras di Kota Kendari menggunakan uji statistik parametrik.**

---

## 📊 Gambaran Hasil (Visualization)

<p align="center">
  <img src="https://github.com/user-attachments/assets/33d2a411-ea4c-4bd1-aba5-195deeb21ae2" alt="Grafik Analisis Harga Beras vs Curah Hujan" width="100%">
</p>

---

## 🛠️ Fitur & Metodologi

Sistem ini melakukan pengambilan data secara *real-time* dan melakukan analisis statistik otomatis dengan alur kerja sebagai berikut:

1.  **Automated Data Fetching (ETL)**
    * **Harga Pangan:** Mengambil data harian dari API Pusat Informasi Harga Pangan Strategis (PIHPS) Bank Indonesia.
    * **Data Cuaca:** Mengambil data tren curah hujan harian dari MSN Weather API.
2.  **Data Cleaning & Preprocessing**
    * Pembersihan *Missing Values* (NaN).
    * Sinkronisasi tanggal (Inner Join) antara dua sumber data berbeda.
    * Filtering anomali data (harga tidak valid/negatif).
3.  **Statistical Analysis**
    * **Descriptive Stats:** Mean, Median, Skewness, Kurtosis, & CV (Coefficient of Variation) untuk mengukur stabilitas harga.
    * **Independent Sample T-Test (Uji T):** Menguji hipotesis apakah rata-rata harga beras berbeda secara signifikan pada **Hari Hujan** vs **Hari Kering**.
    * **Pearson Correlation:** Mengukur kekuatan hubungan linear antara intensitas hujan (mm) dan harga (Rp).

---

## 🧮 Penjelasan Statistik (Scientific Approach)

Project ini tidak hanya menampilkan grafik, tetapi juga melakukan validasi ilmiah menggunakan **SciPy**:

| Metode Statistik | Tujuan Penggunaan |
| :--- | :--- |
| **Independent T-Test** | Membandingkan dua kelompok: **(1) Harga saat Curah Hujan = 0** dan **(2) Harga saat Curah Hujan > 0**. Jika *p-value* < 0.05, maka cuaca mempengaruhi harga secara signifikan. |
| **Coefficient of Variation (CV)** | Menentukan tingkat volatilitas harga. Jika CV < 5%, harga dikategorikan "Sangat Stabil". |
| **Pearson Correlation** | Melihat pola hubungan. (Contoh: Apakah semakin tinggi curah hujan, harga semakin mahal?). |

---

## 💻 Tech Stack

Project ini dibangun menggunakan ekosistem Python untuk Data Science:

* `requests`: Untuk HTTP Request ke API BI & MSN.
* `pandas`: Manipulasi dan pembersihan data tabular.
* `numpy`: Operasi numerik array.
* `scipy.stats`: Perhitungan Skewness, Kurtosis, dan **T-Test**.
* `matplotlib` & `seaborn`: Visualisasi data (Dual Axis Chart).

---

## 🚀 Cara Menjalankan (How to Run)

Pastikan Anda memiliki Python terinstal.

1.  **Clone Repository**
    ```bash
    git clone [https://github.com/username-anda/repository-anda.git](https://github.com/username-anda/repository-anda.git)
    cd repository-anda
    ```

2.  **Install Library**
    ```bash
    pip install requests pandas numpy matplotlib seaborn scipy
    ```

3.  **Jalankan Script**
    ```bash
    python analysis.py
    ```

---

## 📝 Hasil Analisis (Contoh Output)

Berdasarkan eksekusi kode pada data tahun 2025:

* **Stabilitas Harga:** Terpantau stabil/fluktuatif berdasarkan nilai CV.
* **Signifikansi Cuaca:** * *T-Statistic:* 12.7310
    * *P-Value:* 5.5508
    * **Kesimpulan:** (Misal: "Tidak ditemukan bukti statistik bahwa hujan harian mempengaruhi harga beras secara langsung di pasar Kota Kendari").

---

<p align="center">
  Dibuat dengan ❤️ oleh Juswan
  <br>
  <i>Mahasiswa Teknologi Informasi 2023 - ISTEK Aisyiyah Kendari</i>
</p>
