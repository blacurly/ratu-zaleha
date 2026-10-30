Folder untuk testing pipeline dan file csv daftar green-jobsnya

# Laras Karya: Pipeline Rekomendasi Lowongan Hijau

**Assignment Pipeline Awal · Kelompok Ratu Zaleha**

Dokumen ini menjelaskan fitur yang dipilih, masalah yang diselesaikan, data, alur analisis, dan cara menjalankan notebook `recommendation_pipeline_summary.ipynb`.

## 1. Problem dan pengguna sasaran

Perempuan yang kembali bekerja setelah jeda karier, berpindah karier, atau memulai karier hijau sulit menelusuri lowongan satu per satu. Kriteria yang penting bagi mereka (kesesuaian skill, sektor, pola kerja, lokasi, dan benefit seperti jam fleksibel atau program return-to-work) jarang bisa disaring dengan mudah di bursa kerja umum.

> **Pertanyaan yang dijawab fitur ini:** *Lowongan hijau mana yang paling sesuai dengan profil pengguna, dan apa alasannya?*

**Pengguna sasaran:** pengguna Laras Karya yang sudah mengisi asesmen dan melengkapi profilnya.

## 2. Fitur yang dipilih dan perannya di final project

Fitur **Rekomendasi Lowongan Hijau + "Mengapa cocok untuk Anda"**.

- Di aplikasi Laras Karya, fitur ini berjalan setelah asesmen selesai dan mengisi tabel `job_recommendations`. Halaman *Rekomendasi Lowongan* lalu menampilkan lowongan teratas beserta skor, alasan, dan skill gap.
- Skill gap yang sama nantinya menjadi dasar saran pelatihan di *Jalur Belajar*.
- Notebook ini adalah **prototipe pipeline-nya**: input dan output sudah sesuai kebutuhan aplikasi, tetapi datanya masih berasal dari file CSV, belum dari database.

**Prinsip desain: aturan dulu, AI sesudahnya.**

| Lapisan | Tugas | Alasan |
|---|---|---|
| Aturan (rule-based) | Filter, skor 7 komponen, peringkat | Deterministik, bisa diaudit, tidak bisa "mengarang" lowongan |
| AI | Mengusulkan skill dari cerita pengalaman, dan menulis penjelasan untuk hasil peringkat | AI dipakai di tempat ia kuat (bahasa), bukan untuk memutuskan siapa yang layak |

Skor adalah ukuran **kesesuaian** profil dengan lowongan, **bukan** peluang diterima kerja.

## 3. Data dan input

### Dataset lowongan: `green-jobs.csv`
- **Sumber:** lowongan hasil *scraping* memakai Apify dari JobStreet.
- **Ukuran:** 50 lowongan, 10 kolom.
- **Kolom:** `id`, `job_title`, `skills`, `company_name`, `sector`, `benefits`, `province`, `experience_level`, `min_education`, `work_arrangement`.
- **Kualitas data (sebelum cleaning):**

| Kolom | Baris kosong (dari 50) |
|---|---|
| `experience_level` | 34 |
| `benefits` | 25 |
| `province` | 16 |
| `company_name` | 11 |
| `min_education` | 9 |
| `sector` | 5 |

  Dua lowongan memiliki skill `[Perlu review manual]`.
- **Catatan isi:** `sector` berisi nama peran (misalnya `environmental-officer`), bukan sektor industri; `province` berisi kota atau kabupaten; `benefits` berisi benefit umum (Medical, Dental, Parking) dan tidak memuat benefit ramah perempuan.
- Data hanya dipakai untuk keperluan pembelajaran dan prototipe.

### Profil kandidat (demo, buatan sendiri)
Berbentuk dictionary: `skills`, `interest_sectors`, `work_arrangement_pref`, `province`, `open_to_relocate`, `career_path`, `experience_level`, `max_education`, `preferences`, dan daftar lowongan yang sudah dilamar. Profil demo adalah perempuan yang kembali bekerja setelah jeda karier, domisili Jakarta Selatan.

### Cerita pengalaman (demo, buatan sendiri)
Teks bebas yang menjadi bahan ekstraksi skill.

## 4. Output

Tabel peringkat (maksimal 20 lowongan) yang disimpan ke `job_recommendations_output.csv`. Tiap baris berisi:

| Kolom | Isi |
|---|---|
| `match_score` | Skor kesesuaian 0 sampai 100 |
| `breakdown` | Nilai tiap komponen, skill yang cocok, dan skill gap |
| `known_dimensions` | Jumlah komponen yang datanya tersedia (maksimal 7) |
| `reason` | Alasan berbasis aturan |
| `already_applied` | Penanda lowongan yang sudah dilamar |
| `ai_explanation`, `ai_next_step`, `ai_model` | Penjelasan dan saran langkah dari lapisan AI (saat ini simulasi) |
| `input_hash` | Sidik jari masukan, agar hitung ulang bisa dilewati bila profil tidak berubah |

## 5. Alur analisis

```
Cerita pengalaman ─► [EKSTRAKSI SKILL (AI)] ─► pengguna konfirmasi ─► skill profil
                                                                          │
green-jobs.csv ─► [CLEANING] ─► [FILTER] ─► [SKOR 7 komponen] ─► [PERINGKAT Top-N]
                                                                          │
                       [AI: penjelasan + langkah berikutnya, hanya Top-K] ─► [VALIDASI] ─► CSV hasil
```

Notebook terdiri dari 10 tahap:

| # | Tahap | Isi |
|---|---|---|
| 1 | Setup data | Memuat CSV, memeriksa kolom dan nilai kosong |
| 2 | Kolom dan konfigurasi | Deteksi kolom, bobot, nilai netral |
| 3 | Cleaning | Menyeragamkan teks, memecah skill, memetakan nilai ke standar |
| 4 | Profil kandidat | Membangun profil yang sudah dinormalisasi |
| 5 | Ekstraksi skill | AI mengusulkan skill dari cerita; hanya yang dikonfirmasi masuk profil |
| 6 | Filter dan skor | Filter keras lalu skor 7 komponen berbobot |
| 7 | Peringkat dan penjelasan | Urutan, alasan, skill gap, dan dampak skill hasil ekstraksi |
| 8 | Eksperimen bobot | Membandingkan tiga skema bobot |
| 9 | Lapisan AI | Penjelasan personal untuk hasil teratas, dengan validasi |
| 10 | Uji dan kesimpulan | 15 uji otomatis, simpan hasil, interpretasi |

### Aturan penting
- **Filter keras:** status `published`, lowongan hijau, dan belum lewat deadline. Dataset ini tidak punya kolom tersebut, sehingga semua lowongan lolos; aturannya diuji terpisah pada tiga lowongan buatan.
- **Data kosong dihitung netral (50), bukan 0.** Jumlah komponen yang datanya benar-benar ada dicatat di `known_dimensions`.
- **Penyebut skor skill adalah jumlah skill lowongan**, bukan skill pengguna.

### Bobot 7 komponen (nilai awal)

| Komponen | Bobot | Aturan singkat |
|---|---|---|
| `skill` | 25% | Persentase skill yang diminta lowongan dan sudah dimiliki |
| `sector` | 20% | Minat sektor sama dengan sektor lowongan |
| `work_style` | 10% | Onsite, hybrid, atau remote |
| `location` | 10% | Provinsi sama, remote, atau bersedia relokasi |
| `experience` | 15% | Level pengalaman memenuhi syarat |
| `education` | 10% | Jenjang pendidikan memenuhi syarat minimum |
| `benefits` | 10% | Kebutuhan pengguna disediakan lowongan |

Bobot adalah asumsi awal dan perlu dikalibrasi dengan data nyata (klik, simpan, lamar).

### Guardrail AI
- AI tidak menambah lowongan, tidak mengubah skor atau urutan.
- Data pribadi dan preferensi pengguna tidak dikirim ke AI.
- Output AI divalidasi (JSON valid, `job_id` ada di daftar, tanpa kata sensitif); bila gagal, alasan berbasis aturan dipakai.
- Ekstraksi skill hanya dari kosakata dataset, wajib disertai kutipan yang benar-benar ada di cerita, tidak dipanggil tanpa persetujuan, dan baru berlaku setelah pengguna mengonfirmasi.

## 6. Cara menjalankan

**Google Colab (disarankan)**
1. Buka `recommendation_pipeline_summary.ipynb` di Google Colab.
2. Pilih **Runtime → Run all**.
3. Saat sel setup data meminta file, **unggah `green-jobs.csv`** dari folder ini.
4. Hasil akan tersimpan sebagai `job_recommendations_output.csv` di sesi Colab.

**Lokal (Jupyter)**
1. Taruh `green-jobs.csv` di folder yang sama dengan notebook.
2. Jalankan seluruh sel. Kebutuhan: Python 3, `pandas`, `numpy`.

Tidak diperlukan API key atau koneksi database.

## 7. Hasil awal

Dijalankan pada 50 lowongan dan 1 profil demo:

- Seluruh 50 lowongan diproses setelah cleaning. Tiga teratas: Environment Engineer (66,7, sudah dilamar), HSE Officer (56,9), dan Oil Refinery (56,8). Selisih peringkat 2 dan 3 hanya 0,1 poin, jadi tidak boleh dibaca sebagai "pasti lebih baik".
- Skor tertinggi hanya 66,7 karena profil awal hanya punya 4 skill dan komponen `benefits` bernilai 0 untuk semua lowongan (dataset tidak memuat benefit ramah perempuan). Skor dibaca sebagai kesesuaian relatif antar lowongan.
- Ekstraksi skill: dari satu cerita, 8 skill diusulkan dan 7 di antaranya baru. Setelah dikonfirmasi, skor Environment Engineer naik dari 66,7 ke 75,0, dan tidak ada lowongan yang skornya turun.
- Urutan teratas sensitif terhadap bobot (tahap 8), jadi bobot perlu dikalibrasi.
- 15 uji otomatis lulus, termasuk uji guardrail AI (lowongan karangan, output bukan JSON, dan penebakan kondisi pribadi semuanya ditolak).

## 8. Keterbatasan

- Hanya 50 lowongan dan 1 profil demo; belum ada evaluasi dengan pengguna nyata.
- Penjelasan AI dan ekstraksi skill masih **simulasi** (template dan kata kunci), belum LLM sungguhan.
- Komponen `benefits` belum bermakna karena datanya tidak memuat benefit yang dicari.
- Kolom `sector` berisi nama peran, dan lowongan dengan data tidak lengkap bisa terangkat karena data kosong dihitung netral.
- Skill pada dataset berupa teks bebas; sinonim tidak dicocokkan.
- Notebook belum memakai struktur database aplikasi (ERD) secara penuh.

## 9. Langkah berikutnya

1. Sambungkan LLM sungguhan (Gemini) untuk ekstraksi skill dan penjelasan, lalu evaluasi 20 sampel secara manual.
2. Samakan skema dengan ERD aplikasi (`user_skills`, `job_skills`, `job_recommendations`) dan simpan hasil ke database.
3. Lengkapi data: benefit ramah perempuan, sektor yang baku, dan pemetaan kota ke provinsi.
4. Kalibrasi bobot dengan data klik, simpan, dan lamar.
5. Hubungkan skill gap ke saran pelatihan di Jalur Belajar.
6. Integrasikan ke frontend.

**Prinsip responsible AI:** rekomendasi membantu pengguna menemukan peluang, bukan menjadi keputusan otomatis untuk menolak atau menerima kandidat. Atribut sensitif tidak dipakai, dan peringkat perlu diuji agar tidak merugikan kelompok tertentu.

## 10. Isi folder

| File | Keterangan |
|---|---|
| `recommendation_pipeline_summary.ipynb` | Notebook pipeline |
| `green-jobs.csv` | Dataset lowongan |
| `README.md` | Dokumentasi ini |
| `job_recommendations_output.csv` | Dihasilkan saat notebook dijalankan |

