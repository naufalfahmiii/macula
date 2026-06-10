# OCR SCAN

> Pengembangan aplikasi OCR berbasis browser dari project **Macula** (github.com/xirf/macula). Project asli sudah mendukung scan gambar dan deteksi kesalahan teks via WebAssembly. Dikembangkan lebih lanjut dengan penambahan fitur scan PDF, pilihan bahasa OCR, crop area, zoom, rotasi, OCR Enhancement, pengelolaan teks, dan download file — semua berjalan di browser tanpa server.
>
> 🌐 **Live Demo:** [ocrscanteks.netlify.app](https://ocrscanteks.netlify.app)

---

## ✨ Fitur

### Fitur Existing (Project Macula Asli)
- 🖼️ **Scan Gambar** — upload JPG, PNG, dan format gambar lainnya
- 🔬 **Run Detection** — deteksi kesalahan OCR per kata via Macula WASM engine
- 📊 **Tabel Hasil** — menampilkan status Valid / Corrected / Error per token beserta confidence score

### Fitur yang Dikembangkan
- 📄 **Scan PDF** — upload PDF scan, render per halaman dengan navigasi
- 🔢 **Navigasi Halaman PDF** — navigator « ‹ 1/N › » dinamis mengikuti jumlah halaman
- 🌐 **Pilihan Bahasa OCR** — English, Indonesia, atau English+Indonesia
- ✂️ **Crop Area** — pilih area tertentu pada gambar/PDF sebelum di-OCR
- 🔍 **Zoom Control** — perbesar tampilan preview untuk seleksi area lebih presisi
- 🔄 **Rotate** — putar gambar 90°, 180°, 270° untuk dokumen miring
- 🔧 **OCR Enhancement** — slider brightness & threshold untuk dokumen gelap/pudar
- 🔠 **CAPS Lock** — hasil OCR otomatis huruf kapital
- ↔️ **1 Baris** — gabung semua baris menjadi satu baris teks
- 📝 **Kolom Edit Teks** — tampung hasil scan sementara antar halaman/dokumen
- 💾 **Download File** — rename file otomatis berdasarkan isi teks hasil scan

---

## 🛠️ Tech Stack

**Frontend**
- Nuxt `4.x` — framework Vue.js untuk SSG/SPA
- Vue `3.x` — reactive UI framework
- Tailwind CSS `4.x` — utility-first CSS framework
- Lucide Vue Next — icon library

**OCR & Processing**
- Tesseract.js `5.x` — OCR engine client-side (sudah ada di project asli)
- PDF.js `4.4.168` — render halaman PDF ke canvas (dikembangkan)

**Detection Engine**
- Macula WASM (Zig) — deteksi kesalahan teks OCR (engine asli)
- `ocr_wasm.wasm` ~680 KB — compiled engine
- `ocr_detector.bin` ~118 KB — model kamus deteksi

**Deployment**
- Netlify — static hosting

---

## 📋 Prasyarat

Pastikan semua tools berikut sudah terinstall sebelum memulai:

**Node.js** `v18.0.0` atau lebih baru
```bash
node --version   # pastikan v18+
```

**npm** `v8.0.0` atau lebih baru (sudah termasuk bersama Node.js)
```bash
npm --version    # pastikan v8+
```

**Git** (untuk clone repository)
```bash
git --version
```

> Unduh Node.js di [nodejs.org](https://nodejs.org) — pilih versi LTS.
> Unduh Git di [git-scm.com](https://git-scm.com).

---

## 🚀 Instalasi dan Menjalankan

### Langkah 1 — Clone Repository

```bash
git clone https://github.com/xirf/macula
cd macula
```

### Langkah 2 — Masuk ke Folder Demo

```bash
cd demo
```

### Langkah 3 — Install Dependencies

```bash
npm install
```

Proses ini akan mengunduh semua package yang dibutuhkan ke folder `node_modules/`. Tunggu hingga selesai.

### Langkah 4 — Pastikan File WASM Tersedia

Cek apakah file berikut sudah ada di folder `public/wasm/`:

```
demo/public/wasm/
├── ocr_wasm.wasm      (~680 KB)
└── ocr_detector.bin   (~118 KB)
```

Jika belum ada, download manual:
```bash
# Buat folder jika belum ada
mkdir -p public/wasm

# Download file WASM
curl -L -o public/wasm/ocr_wasm.wasm "https://github.com/xirf/macula/raw/main/zig-out/bin/ocr_wasm.wasm"
curl -L -o public/wasm/ocr_detector.bin "https://github.com/xirf/macula/raw/main/data/ocr_detector.bin"
```

### Langkah 5 — Jalankan Development Server

```bash
npm run dev
```

Buka browser dan akses `http://localhost:3000`

---

## 📁 Struktur Folder

```
macula/
├── build.zig                   Zig build system
├── src/
│   ├── root.zig                Public API entry point
│   ├── tokenizer.zig           Zero-copy byte tokenizer
│   ├── hash.zig                FNV-1a 64-bit hash + fingerprinting
│   ├── mphf.zig                Minimal Perfect Hash Function (CHD)
│   ├── confusion.zig           OCR confusion matrix (200 rules)
│   ├── candidate.zig           Confusion candidate generator (two-edit beam)
│   ├── gru.zig                 Int8 quantized GRU language model
│   ├── detector.zig            Detection pipeline orchestrator
│   ├── binary.zig              Binary artifact format spec
│   ├── loader.zig              Artifact deserialization
│   └── wasm.zig                WebAssembly entry point and exports
├── demo/                       Aplikasi web (Nuxt 4 + Tailwind CSS v4)
│   ├── app/
│   │   ├── app.vue             Komponen utama aplikasi
│   │   └── composables/
│   │       └── useMacula.ts    Composable Macula WASM engine
│   ├── public/
│   │   ├── wasm/
│   │   │   ├── ocr_wasm.wasm   Engine Macula (hasil kompilasi Zig)
│   │   │   └── ocr_detector.bin Model kamus deteksi
│   │   └── _redirects          Konfigurasi Netlify SPA routing
│   └── nuxt.config.ts          Konfigurasi Nuxt
├── scripts/
│   ├── preprocess.py           Dataset to wordlist + confusion pairs
│   ├── train_gru.py            PyTorch GRU training and int8 export
│   ├── expand_dict.py          Merge with large English word list
│   ├── compile_artifact.py     Package into ocr_detector.bin
│   └── evaluate.py             Precision/recall/F1 evaluation
└── data/
    └── ocr_detector.bin        ~118 KB compiled model artifact
```

---

## 🌐 Build dan Deploy ke Netlify

### Langkah 1 — Build Production

```bash
cd demo
npx nuxt generate
```

Proses ini menghasilkan folder `.output/public` berisi file statis siap deploy.

### Langkah 2 — Cek Hasil Build

Pastikan file penting ada di dalam `.output/public/`:
```
.output/public/
├── wasm/
│   ├── ocr_wasm.wasm      ← harus ada
│   └── ocr_detector.bin   ← harus ada
├── _redirects             ← harus ada
└── index.html
```

Jika folder `wasm/` tidak ada, pastikan file WASM sudah ada di `public/wasm/` sebelum build.

### Langkah 3 — Deploy ke Netlify

**Cara Manual (Netlify Drop):**
1. Buka [app.netlify.com/drop](https://app.netlify.com/drop)
2. Drag folder `.output/public` ke halaman tersebut
3. Tunggu proses upload selesai
4. Salin URL yang diberikan Netlify

**Update/Redeploy:**
1. Jalankan ulang `npx nuxt generate`
2. Drag ulang folder `.output/public` ke Netlify Drop
3. Netlify otomatis memperbarui site yang sudah ada

> **Penting:** File `_redirects` di dalam `public/` wajib ada agar routing SPA berjalan. Isinya:
> /* /index.html 200

---

## 📖 Cara Pakai

### Scan Gambar
1. Pilih **Bahasa OCR** sesuai dokumen (English / Indonesia / English+Indonesia)
2. Klik **Choose image** → pilih file gambar
3. Drag pada preview untuk memilih area (opsional)
4. Atur **Brightness/Threshold** jika gambar gelap
5. Klik **Extract Text** atau **Extract Area** (jika ada crop)

### Scan PDF
1. Pilih **Bahasa OCR** sesuai dokumen
2. Klik **Choose PDF (Scan)** → pilih file PDF
3. Gunakan navigator **« ‹ 1/N › »** untuk berpindah halaman
4. Drag pada preview untuk memilih area (opsional)
5. Klik **Extract Semua** atau **Extract Area** (jika ada crop)

### Pengelolaan Hasil
1. Aktifkan **CAPS ON** dan/atau **1 BARIS ON** sesuai kebutuhan
2. Edit hasil di **Kolom Hasil Scan**
3. Klik **Simpan** → hasil masuk ke **Kolom Edit Teks**
4. Scan halaman/dokumen lain → ulangi langkah 1–3
5. Edit teks final di Kolom Edit Teks
6. Klik **Download** → file diunduh dengan nama otomatis dari isi teks

### Deteksi Kesalahan OCR
1. Pastikan teks sudah ada di Kolom Hasil Scan
2. Klik **Run Detection**
3. Scroll ke bawah untuk melihat tabel hasil deteksi per kata

---

## 🔧 Panduan OCR Enhancement

Gunakan slider **Brightness** dan **Threshold** sesuai kondisi dokumen:

**Dokumen normal / cerah** → Brightness `1.4x` · Threshold `100`

**Dokumen agak gelap** → Brightness `1.8x–2.0x` · Threshold `90`

**Dokumen sangat gelap / arsip tua** → Brightness `2.2x–2.5x` · Threshold `70–80`

**Dokumen terlalu terang / overexposed** → Brightness `1.0x–1.2x` · Threshold `110–120`

---

## 🙏 Credit

- [Macula](https://github.com/xirf/macula) oleh **Anka Tama (xirf)** — OCR error detection engine (Zig + WebAssembly)
- [Tesseract.js](https://github.com/naptha/tesseract.js) — OCR engine
- [PDF.js](https://github.com/mozilla/pdf.js) — PDF renderer oleh Mozilla