# Semantic Search & LLM Application

Prototype asisten AI customer service untuk e-commerce yang mencari FAQ paling relevan terhadap pertanyaan pelanggan, lalu menyusun jawaban natural menggunakan Large Language Model (LLM). Prototype ini menggunakan semantic search berbasis embedding dan FAISS sebagai vector index.

## 1. Problem Statement dan Knowledge Base

Tim customer service e-commerce menerima ratusan pertanyaan berulang mengenai pemesanan, pengiriman, pembayaran, dan pengembalian barang. Tujuan prototype ini adalah membantu menjawab pertanyaan tersebut secara otomatis berdasarkan knowledge base FAQ, termasuk ketika pelanggan menggunakan sinonim atau parafrasa yang berbeda dari teks FAQ.

Knowledge base yang digunakan adalah file CSV `ecommerce_faq_knowledge_base.csv` dengan **59 entri dan 4 kolom**:

- `faq_id`: ID entri FAQ.
- `category`: kategori pertanyaan.
- `question`: pertanyaan FAQ.
- `answer`: jawaban resmi untuk FAQ tersebut.

Notebook membaca file dari path relatif `../data/ecommerce_faq_knowledge_base.csv`. Pastikan struktur folder proyek sesuai dengan path tersebut.

## 2. Alur Sistem

Alur utama sistem adalah **ingest → embed → store → retrieve → generate**.

1. **Ingest** — Membaca knowledge base CSV menggunakan pandas. Setiap entri FAQ disiapkan sebagai satu unit yang dapat di-retrieve dengan menggabungkan pertanyaan dan jawaban dalam format `Q: ... A: ...`.
2. **Embed** — Mengubah setiap unit FAQ menjadi vektor numerik menggunakan model embedding `gemini-embedding-001` melalui Gemini API. Hasil pengujian notebook menghasilkan embedding berukuran 3.072 dimensi untuk masing-masing dari 59 entri.
3. **Store** — Menyimpan vektor di index FAISS menggunakan `faiss.IndexFlatL2`, yang melakukan pencarian berdasarkan jarak Euclidean kuadrat (L2). Index ini dibangun di memori saat notebook dijalankan.
4. **Retrieve** — Pertanyaan pengguna juga diubah menjadi embedding. FAISS mencari `top_k` vektor terdekat; secara default, fungsi `semantic_search()` mengambil tiga FAQ teratas. Sistem mengembalikan pertanyaan, jawaban, kategori, dan jarak hasil pencarian.
5. **Generate** — Pertanyaan pengguna dan FAQ hasil retrieval disusun ke dalam prompt. Prompt menginstruksikan LLM untuk menjawab hanya berdasarkan FAQ yang diberikan dan, jika informasi tidak tersedia, menyatakan keterbatasan serta menyarankan pengguna menghubungi customer service. Jawaban dibuat menggunakan `gemini-3.1-flash-lite` dengan `temperature=0.3`.

Secara ringkas:

```text
FAQ CSV
  → gabungkan question + answer
  → Gemini Embeddings API
  → vektor FAQ di FAISS
  → embedding pertanyaan pengguna
  → retrieve top-k FAQ
  → susun prompt berisi FAQ terpilih
  → Gemini LLM
  → jawaban untuk pelanggan
```

## 3. Hasil Pengujian dan Analisis Relevansi

Notebook menguji lima query pada Section 8. Penilaian di bawah merangkum hasil retrieval dan respons yang diamati ketika notebook dijalankan.

| # | Query pengguna | FAQ teratas / hasil retrieval | Penilaian |
|---|---|---|---|
| 1 | “Apakah ada diskon untuk member baru?” | “Apakah member baru mendapat diskon khusus?” | **Baik.** Sistem mengenali bahwa “diskon untuk member baru” sepadan maknanya dengan “diskon khusus”. LLM menjawab tentang voucher selamat datang untuk transaksi pertama, sesuai dengan FAQ. |
| 2 | “Bagaimana caranya mendaptkan invoice?” | “Bagaimana cara mendapatkan bukti pembayaran atau invoice?” | **Baik.** Kata “invoice” membantu retrieval menemukan FAQ yang tepat meskipun ada variasi penulisan dan struktur kalimat. Jawaban LLM mengenai pengunduhan invoice dari detail pesanan setelah pembayaran dikonfirmasi sesuai dengan knowledge base. |
| 3 | “Bagaimana cara reset kode sandi?” | “Saya lupa password akun, bagaimana cara mengaturnya ulang?” | **Baik.** Semantic search menghubungkan “reset kode sandi” dengan “lupa password” dan “mengaturnya ulang”. Instruksi reset yang diberikan LLM sesuai dengan FAQ. |
| 4 | “Kenapa penjual mengatakan stok habis padahal stok yang tertera di aplikasi masih ada?” | “Apakah stok yang tertera di aplikasi selalu update secara real-time?” | **Relevan, tetapi LLM menyarankan ekskalasi.** FAQ berkaitan dengan status stok, tetapi LLM menambahkan jika jawaban tidak menjawab kendala user secara spesifik maka disarankan menghubungi customer service. |
| 5 | “Apa yang membuat aplikasi ini lebih unggul daripada aplikasi lain?” | “Bagaimana cara membandingkan beberapa produk sekaligus?” | **Tidak relevan.** Kemiripan konsep “membandingkan” tidak berarti kedua pertanyaan memiliki maksud yang sama: query menanyakan keunggulan aplikasi dibandingkan kompetitor, sedangkan FAQ membahas perbandingan produk. LLM menyatakan informasi belum tersedia dan menyarankan menghubungi customer service, sesuai instruksi prompt. |

### Ringkasan hasil

- **3 dari 5 query** menunjukkan retrieval yang baik, terutama untuk sinonim dan parafrasa.
- **1 query** memperoleh FAQ yang berkaitan, tetapi LLM menambahkan saran untuk menghubungi customer service jika jawaban yang diberikan tidak menyelasaikan kendala.
- **1 query** menunjukkan ketidakcocokan maksud meskipun ada kemiripan kata. Dalam pengujian ini, LLM menangani ketiadaan informasi dengan menyatakan keterbatasannya.

Angka di atas merupakan evaluasi kualitatif atas lima contoh, bukan metrik benchmark menyeluruh. Hasil generasi dapat berubah ketika API/model diperbarui atau notebook dijalankan kembali.

## 4. Keterbatasan Sistem

1. **Cakupan knowledge base terbatas.** Sistem hanya dapat menjawab berdasarkan 59 entri FAQ yang tersedia. Pertanyaan di luar cakupan FAQ tidak dapat dijawab secara andal.
2. **Belum ada ambang relevansi atau abstention berbasis skor.** FAISS selalu mengembalikan tetangga terdekat untuk query, termasuk ketika semua hasil sebenarnya tidak relevan. Sistem belum menggunakan ambang jarak untuk menolak konteks yang lemah.
3. **Risiko jawaban tidak grounded.** Instruksi prompt meminta LLM hanya menggunakan FAQ, tetapi model masih dapat menambahkan informasi yang tidak terdapat di konteks, seperti pada query stok.
4. **Evaluasi masih kecil dan kualitatif.** Lima query belum cukup untuk mengukur performa secara menyeluruh. Belum ada dataset uji berlabel, metrik retrieval seperti Recall@k/MRR, atau metrik groundedness jawaban.
5. **Index disimpan di memori.** FAISS index dibuat ulang saat notebook dijalankan dan tidak dipersistenkan sebagai artefak. Untuk data lebih besar, pertimbangkan penyimpanan index dan metadata secara terpisah.
6. **Ketergantungan pada layanan eksternal.** Pembuatan embedding dan jawaban memerlukan akses internet, Gemini API, API key yang valid, serta kuota layanan.
7. **Pencarian menggunakan jarak L2.** Notebook memakai `IndexFlatL2` dan tidak melakukan normalisasi embedding secara eksplisit. Pengaturan metrik perlu dievaluasi dan konsisten dengan model embedding yang digunakan.
8. **Belum siap untuk penggunaan produksi.** Belum ada penanganan lengkap untuk timeout/retry, logging dan monitoring, evaluasi berkelanjutan, kontrol akses, perlindungan data pelanggan, maupun mekanisme eskalasi ke agen manusia yang terintegrasi.

## 5. Cara Menjalankan Notebook

### Prasyarat

- Python yang kompatibel dengan versi library di `requirements.txt`.
- Jupyter Notebook atau JupyterLab.
- Gemini API key dengan akses ke model embedding dan generasi yang digunakan.
- File knowledge base `ecommerce_faq_knowledge_base.csv`.

### Struktur folder

Notebook mengasumsikan CSV berada satu tingkat di atas folder notebook, dalam folder `data`. Contoh struktur:

```text
project/
├── data/
│   └── ecommerce_faq_knowledge_base.csv
├── notebooks/
│   └── starter_notebook_semantic_search.ipynb
└── requirements.txt
```

Jika notebook disimpan di lokasi berbeda, sesuaikan path CSV pada cell pertama agar menunjuk ke file yang benar.

### Langkah menjalankan

1. Clone atau unduh repository proyek, lalu buka terminal di direktori proyek.
2. Buat dan aktifkan virtual environment (opsional, tetapi disarankan):

   **Windows (PowerShell):**
   ```powershell
   python -m venv .venv
   .\.venv\Scripts\Activate.ps1
   ```

   **macOS/Linux:**
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   ```

3. Instal dependensi yang terdaftar pada `requirements.txt`:

   ```bash
   python -m pip install --upgrade pip
   pip install -r requirements.txt
   ```

4. Atur environment variable `GEMINI_API_KEY`. Jangan menaruh API key langsung di notebook atau mengunggahnya ke repository.

   **Windows (PowerShell):**
   ```powershell
   $env:GEMINI_API_KEY="YOUR_API_KEY"
   ```

   **macOS/Linux:**
   ```bash
   export GEMINI_API_KEY="YOUR_API_KEY"
   ```

   Jalankan Jupyter dari terminal yang sama agar notebook mewarisi environment variable tersebut.

5. Jalankan Jupyter:

   ```bash
   jupyter notebook
   ```

   Buka notebook `starter_notebook_semantic_search.ipynb` (atau versi notebook yang telah dianalisis), kemudian jalankan cell secara berurutan dari atas ke bawah.

6. Pastikan cell pemuatan CSV berhasil menemukan 59 entri, embedding selesai dibuat, dan FAISS index berisi 59 vektor. Jalankan Section 8 untuk mengulangi pengujian query.