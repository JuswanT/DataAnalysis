# DataAnalysis
Analisis Stabilitas Harga Beras dan Pengaruh Anomali Curah Hujan Harian di Kota Kendari Tahun 2025

import json
import requests  # Wajib ada untuk mengambil data dari internet
import pandas as pd
import numpy as np  # Ditambahkan untuk perhitungan statistik
import matplotlib.pyplot as plt
import matplotlib.dates as mdates
import seaborn as sns
from scipy.stats import skew, kurtosis, ttest_ind  # f_oneway dihapus karena ANOVA tidak digunakan

# ==========================================
# KONFIGURASI URL & HEADERS
# ==========================================
# URL Sumber Data (Langsung dari Link yang diberikan)
url_harga_bi = "https://www.bi.go.id/hargapangan/WebSite/TabelHarga/GetGridDataDaerah?price_type_id=1&comcat_id=cat_1%2Ccom_1%2Ccom_2%2Ccom_3%2Ccom_4%2Ccom_5%2Ccom_6&province_id=27&regency_id=72&market_id=&tipe_laporan=1&start_date=2025-02-01&end_date=2025-12-01&_=1764551221259"
url_cuaca_msn = "https://assets.msn.com/service/weather/weathertrends?apiKey=j5i4gDqHL6nGYwx5wi5kRhXjtf2c5qgFX9fzfk0TOo&cm=id-id&locale=id-id&lon=122.698&lat=-6.1954&units=C&user=m-03CF516DC2DF6F402CEF47CCC39A6E6D&ocid=msftweather&includeWeatherTrends=true&includeCalendar=false&fdhead=PRG-1SW-WXNCVF,PRG-1SW-WXTRLOG&weatherTrendsScenarios=PrecipitationTrend&days=30&insights=1&startDate=20250201&endDate=20260131"

# Headers agar request dianggap valid oleh server (menghindari blokir 403 Forbidden)
headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
    "Referer": "https://www.msn.com/",
    "Accept": "*/*"
}

print("--- MEMULAI PENGAMBILAN & ANALISIS DATA ONLINE ---")

try:
    # ---------------------------------------------------------
    # 1. FETCH & OLAH DATA HARGA PANGAN
    # ---------------------------------------------------------
    print(f"1. Mengunduh data harga pangan...")
    response_bi = requests.get(url_harga_bi, headers=headers)
    response_bi.raise_for_status()
    data_bi = response_bi.json()
    
    # Ambil list data
    raw_bi = data_bi.get('data', [])
    if not raw_bi:
        raise ValueError("Data harga kosong atau format JSON BI berubah.")
        
    df_bi = pd.DataFrame(raw_bi)
    
    # Deteksi kolom tanggal (format dd/mm/yyyy)
    date_cols = [c for c in df_bi.columns if '/' in c]
    
    # Melt (Melebar -> Memanjang)
    df_bi_long = df_bi.melt(
        id_vars=['name'], 
        value_vars=date_cols, 
        var_name='Tanggal', 
        value_name='Harga'
    )
    
    # Cleaning Harga & Tanggal
    df_bi_long['Harga'] = pd.to_numeric(
        df_bi_long['Harga'].astype(str).str.replace('.', '', regex=False).str.replace(',', '', regex=False), 
        errors='coerce'
    )
    df_bi_long['Tanggal'] = pd.to_datetime(df_bi_long['Tanggal'], format='%d/%m/%Y')
    
    # Aggregasi: Hitung Rata-rata Harga Beras Per Hari (Semua jenis digabung)
    df_harga_daily = df_bi_long.groupby('Tanggal')['Harga'].mean().reset_index()
    df_harga_daily.rename(columns={'Harga': 'Harga_Beras_Rata2'}, inplace=True)
    
    print(f"   > Data harga berhasil diambil: {len(df_harga_daily)} hari data.")

    # ---------------------------------------------------------
    # 2. FETCH & OLAH DATA CUACA
    # ---------------------------------------------------------
    print(f"2. Mengunduh data cuaca dari MSN Weather...")
    response_msn = requests.get(url_cuaca_msn, headers=headers)
    response_msn.raise_for_status()
    data_msn = response_msn.json()
        
    # Navigasi struktur JSON MSN
    # (Menyesuaikan struktur JSON kompleks)
    try:
        trends = data_msn['value'][0]['responses'][0]['trendChart']
    except (KeyError, IndexError):
        # Fallback jika struktur json sedikit berbeda
        trends = data_msn.get('trendChart', {})

    if not trends:
        raise ValueError("Data tren cuaca tidak ditemukan dalam respon MSN.")

    weather_data = []
    for date_str, values in trends.items():
        weather_data.append({
            'Tanggal': pd.to_datetime(date_str),
            # Key '11' = Curah Hujan (mm), Key '72' = Peluang Hujan (%)
            'Curah_Hujan_mm': values['trendDays'].get('11', 0.0),
            'Peluang_Hujan_Persen': values['trendDays'].get('72', 0.0)
        })
        
    df_cuaca = pd.DataFrame(weather_data)
    print(f"   > Data cuaca berhasil diambil: {len(df_cuaca)} hari data.")

    # ---------------------------------------------------------
    # 3. PENGGABUNGAN DATA (MERGE)
    # ---------------------------------------------------------
    print("3. Menggabungkan data (Merge)...")
    
    # Inner join: Hanya ambil tanggal yang ada di KEDUA data
    df_final = pd.merge(df_harga_daily, df_cuaca, on='Tanggal', how='inner')
    df_final = df_final.sort_values('Tanggal')
    
    if df_final.empty:
        print("❌ PERINGATAN: Tidak ada tanggal yang cocok antara data Harga dan Cuaca.")
        print("   Cek rentang tanggal di kedua URL Anda.")
    else:
        print(f"   > Berhasil menggabungkan {len(df_final)} data baris (irisan tanggal).")

        # ---------------------------------------------------------
        # 3b. DATA CLEANING & PREPROCESSING
        # ---------------------------------------------------------
        print("\n=== 🧹 PEMBERSIHAN DATA SEBELUM UJI STATISTIK ===")
        initial_count = len(df_final)
        
        # 1. Hapus Baris dengan Nilai Kosong (NaN/Null)
        df_clean = df_final.dropna(subset=['Harga_Beras_Rata2', 'Curah_Hujan_mm'])
        nan_dropped = initial_count - len(df_clean)
        
        # 2. Filter Harga Tidak Valid (Misal Harga <= 0 atau Harga < 1000 perak)
        # Ini penting karena harga 0 akan merusak statistik
        df_clean = df_clean[df_clean['Harga_Beras_Rata2'] > 1000]
        price_invalid = (initial_count - nan_dropped) - len(df_clean)
        
        # 3. Filter Curah Hujan Tidak Valid (Negatif)
        df_clean = df_clean[df_clean['Curah_Hujan_mm'] >= 0]
        
        print(f"   > Data Awal Merge: {initial_count} baris")
        print(f"   > Dibuang (NaN/Kosong): {nan_dropped} baris")
        print(f"   > Dibuang (Harga tdk valid): {price_invalid} baris")
        print(f"   > Data Bersih Siap Uji: {len(df_clean)} baris")
        
        # Update df_final dengan data bersih
        df_final = df_clean.copy()

        # ---------------------------------------------------------
        # 4. SUMMARY STATISTIK KONKRET (LENGKAP)
        # ---------------------------------------------------------
        print("\n=== 📊 SUMMARY STATISTIK LENGKAP ===")
        
        def calculate_detailed_stats(series):
            data = series.dropna()
            n = len(data)
            if n == 0: return pd.Series()
            
            mean_val = data.mean()
            std_val = data.std()
            
            return pd.Series({
                'count': n,
                'mean': mean_val,
                'median': data.median(),
                'mode': data.mode().iloc[0] if not data.mode().empty else np.nan,
                'std': std_val,
                'var': data.var(),
                'min': data.min(),
                'max': data.max(),
                'range': data.max() - data.min(),
                'q1': data.quantile(0.25),
                'q3': data.quantile(0.75),
                'iqr': data.quantile(0.75) - data.quantile(0.25),
                'cv': (std_val / mean_val) * 100 if mean_val != 0 else np.nan,
                'mad': np.mean(np.abs(data - mean_val)),
                'skew': skew(data, bias=False),
                'kurt': kurtosis(data, bias=False)
            })

        # Terapkan fungsi statistik ke kolom harga dan hujan
        stats_summary = df_final[['Harga_Beras_Rata2', 'Curah_Hujan_mm']].apply(calculate_detailed_stats).round(2)
        print(stats_summary)
        
        # ---------------------------------------------------------
        # 5. UJI HIPOTESIS (HANYA UJI T)
        # ---------------------------------------------------------
        print("\n=== 🧪 UJI HIPOTESIS STATISTIK ===")
        
        # Cek Distribusi Nol
        zero_rain_count = (df_final['Curah_Hujan_mm'] == 0).sum()
        total_data = len(df_final)
        if total_data > 0:
            print(f"[INFO] Hari Tanpa Hujan (0 mm): {zero_rain_count} dari {total_data} hari ({zero_rain_count/total_data:.1%} data)")
        else:
            print("[INFO] Tidak ada data tersisa setelah pembersihan.")

        # Cek apakah harga konstan (Varians = 0)
        harga_std = df_final['Harga_Beras_Rata2'].std()
        
        if harga_std == 0 or pd.isna(harga_std):
            print("\n⚠️ PERINGATAN: Harga Beras KONSTAN (Tidak Berubah).")
            print("   Uji statistik (T-Test) tidak dapat dilakukan karena standar deviasi = 0.")
            print("   Ini berarti harga beras sama persis setiap hari di dalam dataset yang diambil.")
        else:
            # Uji T (Independent T-Test)
            print("\n1. Uji T (Independent Sample T-Test):")
            print("   Metode: Membandingkan rata-rata harga pada dua kelompok berbeda (Hari Kering vs Basah).")
            print("   Tujuan: Mengetahui apakah perbedaan harga antar kondisi cuaca ini nyata atau kebetulan.")
            
            group_no_rain = df_final[df_final['Curah_Hujan_mm'] == 0]['Harga_Beras_Rata2']
            group_rain = df_final[df_final['Curah_Hujan_mm'] > 0]['Harga_Beras_Rata2']
            
            if len(group_no_rain) > 1 and len(group_rain) > 1:
                # Cek variansi dalam grup
                if group_no_rain.std() == 0 and group_rain.std() == 0:
                     print("   => TIDAK DAPAT DILAKUKAN: Variansi harga di dalam kedua grup adalah 0 (Harga Konstan).")
                else:
                    t_stat, p_val_t = ttest_ind(group_no_rain, group_rain, equal_var=False)
                    print(f"   - T-Statistic: {t_stat:.4f}")
                    print(f"   - P-Value: {p_val_t}")
                    if pd.isna(p_val_t):
                         print("   => HASIL 'nan': Kemungkinan data harga identik di kedua grup.")
                    elif p_val_t < 0.05:
                        print("   => SIGNIFIKAN: Ada perbedaan harga beras yang nyata saat hujan vs tidak hujan.")
                    else:
                        print("   => TIDAK SIGNIFIKAN: Terjadinya hujan tidak mempengaruhi harga beras secara statistik.")
            else:
                print("   => TIDAK DAPAT DILAKUKAN: Salah satu kelompok data (Hujan/Tidak Hujan) kosong atau terlalu sedikit.")

        # ---------------------------------------------------------
        # 6. KESIMPULAN & VISUALISASI
        # ---------------------------------------------------------
        # Hitung Korelasi Pearson
        corr_matrix = df_final[['Harga_Beras_Rata2', 'Curah_Hujan_mm', 'Peluang_Hujan_Persen']].corr()
        korelasi_harga_hujan = corr_matrix.loc['Harga_Beras_Rata2', 'Curah_Hujan_mm']
        
        print("\n=== 🏁 KESIMPULAN AKHIR ===")
        
        # 1. Interpretasi Korelasi
        print(f"1. Hubungan Korelasi (Pearson: {korelasi_harga_hujan:.4f}):")
        if pd.isna(korelasi_harga_hujan):
             print("   - Hubungan: TIDAK TERDEFINISI (Salah satu variabel konstan/variansi 0).")
        elif abs(korelasi_harga_hujan) < 0.1:
            sifat = "Sangat Lemah / Tidak Ada Hubungan"
            print(f"   - Sifat Hubungan: {sifat}")
        elif abs(korelasi_harga_hujan) < 0.3:
            sifat = "Lemah"
            print(f"   - Sifat Hubungan: {sifat}")
        elif abs(korelasi_harga_hujan) < 0.5:
            sifat = "Sedang"
            print(f"   - Sifat Hubungan: {sifat}")
        else:
            sifat = "Kuat"
            print(f"   - Sifat Hubungan: {sifat}")

        # 2. Interpretasi Stabilitas Harga (CV)
        cv_harga = stats_summary.loc['cv', 'Harga_Beras_Rata2']
        print(f"2. Stabilitas Harga Beras (CV: {cv_harga}%):")
        if pd.isna(cv_harga) or cv_harga == 0:
             print("   - Harga Beras KONSTAN (Tidak ada fluktuasi sama sekali).")
        elif cv_harga < 5:
            print("   - Harga Beras SANGAT STABIL.")
        elif cv_harga < 15:
            print("   - Harga Beras CUKUP STABIL.")
        else:
            print("   - Harga Beras FLUKTUATIF.")

        # ---------------------------------------------------------
        # 7. VISUALISASI GRAFIK
        # ---------------------------------------------------------
        print("\n7. Membuat grafik visualisasi...")
        
        # Setup Plot Style
        sns.set_style("whitegrid")
        fig, ax1 = plt.subplots(figsize=(12, 6))

        # Sumbu Y1 (Kiri): Harga Beras
        color_price = 'tab:red'
        ax1.set_xlabel('Tanggal (2025)', fontsize=12)
        ax1.set_ylabel('Harga Rata-rata Beras (Rp)', color=color_price, fontsize=12)
        ax1.plot(df_final['Tanggal'], df_final['Harga_Beras_Rata2'], color=color_price, linewidth=2, label='Harga Beras')
        ax1.tick_params(axis='y', labelcolor=color_price)

        # Sumbu Y2 (Kanan): Curah Hujan
        ax2 = ax1.twinx()  
        color_rain = 'tab:blue'
        ax2.set_ylabel('Curah Hujan (mm)', color=color_rain, fontsize=12)  
        # Bar chart transparan untuk hujan
        ax2.bar(df_final['Tanggal'], df_final['Curah_Hujan_mm'], color=color_rain, alpha=0.3, width=2, label='Curah Hujan')
        ax2.tick_params(axis='y', labelcolor=color_rain)

        # Judul & Format
        t_val_str = f"{t_stat:.3f}" if 't_stat' in locals() else "N/A"
        plt.title(f'Komparasi Harga Beras vs Curah Hujan (Kendari)\nKorelasi: {korelasi_harga_hujan:.2f} | T-Stat: {t_val_str}', fontsize=14)
        ax1.xaxis.set_major_formatter(mdates.DateFormatter('%b %Y'))
        ax1.xaxis.set_major_locator(mdates.MonthLocator())
        fig.tight_layout()

        # Simpan Gambar
        plt.savefig('analisis_harga_vs_cuaca.png')
        plt.show()
        
        print("\n✅ Grafik berhasil disimpan sebagai 'analisis_harga_vs_cuaca.png'")

except requests.exceptions.RequestException as e:
    print(f"\n❌ GAGAL MENGUNDUH DATA: {e}")
    print("Pastikan komputer terhubung internet dan URL tidak diblokir.")
except ValueError as e:
    print(f"\n❌ ERROR DATA: {e}")
except Exception as e:
    print(f"\n❌ Terjadi kesalahan tak terduga: {e}")

    
<img width="1200" height="600" alt="analisis_harga_vs_cuaca" src="https://github.com/user-attachments/assets/33d2a411-ea4c-4bd1-aba5-195deeb21ae2" />
