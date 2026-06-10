# Macula

**Spot the errors your OCR engine left behind.**

Macula is a lightweight OCR error detection and correction engine built in Zig. It compiles to WebAssembly and runs entirely in the browser with zero server dependency. The full engine plus model weighs in at roughly 800 KB (680 KB WASM + 118 KB model), small enough to auto-load on page open without the user lifting a finger.

The name comes from the *macula* of the human eye, the small region responsible for sharp central vision. That is what this project does for OCR output: it looks closely at each word to catch the mistakes that slipped through.

---

## Why Does This Exist?

OCR engines are good at reading text, but they are not perfect. They confuse characters that look visually similar:

| What OCR reads | What it should be | Why                           |
| -------------- | ----------------- | ----------------------------- |
| `rn`           | `m`               | Looks identical in many fonts |
| `cl`           | `d`               | Joined strokes                |
| `vv`           | `w`               | Two v's merge into w          |
| `0`            | `O`               | Zero vs. letter O             |
| `d0g`          | `dog`             | Mixed digit substitution      |

These errors are invisible to the OCR engine itself but obvious to a human reader. Macula catches them automatically.

---

## How It Works

Macula uses a three-layer detection pipeline. Each layer is fast and operates with zero heap allocation at query time.

```mermaid
flowchart LR
    A["OCR Text"] --> B["Tokenizer"]
    B --> C{"MPHF\nDictionary"}
    C -- "Known word" --> D["Valid"]
    C -- "Unknown word" --> E{"Confusion\nExpansion"}
    E -- "Candidate found\nin dictionary" --> F["Corrected"]
    E -- "No candidate" --> G["Error Detected"]
```

### Layer 1: MPHF Dictionary Lookup

Every word is checked against a dictionary using a Minimal Perfect Hash Function (CHD algorithm). This gives O(1) lookup with a 16-bit fingerprint for false-positive rejection (~1 in 65,536 chance of a false match). Known words pass through instantly.

### Layer 2: Confusion Expansion

If a word is not in the dictionary, Macula applies 200 OCR-specific visual confusion rules. For example, it will try replacing `rn` with `m`, or `0` with `O`, and check if the resulting word exists in the dictionary. It supports up to two simultaneous edits (two-edit beam search), which is how it catches `brovvn` as `brown` (two substitutions: `0` to `o` and `vv` to `w`).

### Layer 3: GRU Scoring (Optional)

A character-level GRU language model (int8 quantized, under 40 KB) can score word plausibility. This catches errors that bypass layers 1 and 2. The GRU is optional and currently disabled in the WASM build to keep things lean.

---

## Live Demo

The interactive demo runs entirely in your browser — no server required.
Try it live at: **[https://macula.andka.id](https://macula.andka.id)**

The WASM engine and model auto-load directly from GitHub (~800 KB total payload) on page open.

Paste any OCR output into the text box and hit "Run Detection". You will see each token classified as Valid, Corrected, or Error, with confidence scores and correction suggestions.

```mermaid
flowchart LR
    subgraph Browser
        A["Nuxt App"] --> B["useMacula composable"]
        B --> C["WebAssembly Module<br />680 KB"]
        C --> D["Model Artifact<br/>118 KB"]
    end
```

---

## Quick Start

### Prerequisites

- [Zig](https://ziglang.org/) 0.16 or later
- Python 3.10+ (for the data pipeline)
- Node.js 18+ (for the demo)

### Build and Test

```bash
# Run all unit tests (53 tests across 9 modules)
zig build test

# Build the WebAssembly module
zig build wasm
```

### Build the Model Artifact

The model artifact (`data/ocr_detector.bin`) is a single binary file that contains the dictionary, confusion rules, and GRU weights. To rebuild it from scratch:

```bash
# 1. Download the ICDAR dataset
kaggle datasets download arjav007/icdar-eng -p data --unzip

# 2. Preprocess: extract words, mine confusion pairs
python scripts/preprocess.py

# 3. Train the GRU language model (requires PyTorch)
python scripts/train_gru.py

# 4. Expand dictionary with a large English word list
python scripts/expand_dict.py

# 5. Compile everything into ocr_detector.bin
python scripts/compile_artifact.py

# 6. Evaluate precision, recall, and F1
python scripts/evaluate.py
```

### Use as a Zig Library

```zig
const ocr = @import("ocr");

// Load the binary artifact
var artifact = try ocr.loader.loadFromFile(allocator, "data/ocr_detector.bin");
defer artifact.deinit();

// Initialize the detector (must be done after the artifact is at its final memory location)
artifact.detector = ocr.OcrDetector.init(
    &artifact.mphf,
    &artifact.confusion_matrix,
    if (artifact.gru != null) &artifact.gru.? else null,
    5.0, // NLL threshold
);

// Process text
var results: [256]ocr.OcrDetector.DetectionResult = undefined;
const text = "Teh quicK brovvn fox";
const n = artifact.detector.processText(text, &results);

for (results[0..n]) |r| {
    switch (r.status) {
        .valid => {},
        .corrected => std.debug.print("{s} -> {s}\n", .{ r.tokenSlice(text), r.correctionSlice().? }),
        .error_detected => std.debug.print("error: {s}\n", .{r.tokenSlice(text)}),
    }
}
```

---

## Project Structure

```
macula/
  build.zig                   Zig build system
  src/
    root.zig                  Public API entry point
    tokenizer.zig             Zero-copy byte tokenizer
    hash.zig                  FNV-1a 64-bit hash + fingerprinting
    mphf.zig                  Minimal Perfect Hash Function (CHD)
    confusion.zig             OCR confusion matrix (200 rules)
    candidate.zig             Confusion candidate generator (two-edit beam)
    gru.zig                   Int8 quantized GRU language model
    detector.zig              Detection pipeline orchestrator
    binary.zig                Binary artifact format spec
    loader.zig                Artifact deserialization
    wasm.zig                  WebAssembly entry point and exports
  demo/                       Nuxt 4 + Tailwind CSS v4 web demo
  scripts/
    preprocess.py             Dataset to wordlist + confusion pairs
    train_gru.py              PyTorch GRU training and int8 export
    expand_dict.py            Merge with large English word list
    compile_artifact.py       Package into ocr_detector.bin
    evaluate.py               Precision/recall/F1 evaluation
  data/
    ocr_detector.bin          ~118 KB compiled model artifact
```

## Binary Artifact Format

All multi-byte values are little-endian. The format is defined in `src/binary.zig`.

```mermaid
block-beta
    columns 5
    A["Header\n32 B"]:1
    B["MPHF Seeds\n20 KB"]:1
    C["Fingerprints\n59 KB"]:1
    D["Confusion\n4 KB"]:1
    E["GRU Weights\n37 KB"]:1
```

| Section           | Size        | Description                                               |
| ----------------- | ----------- | --------------------------------------------------------- |
| Header            | 32 B        | Magic number `OCRD`, version, section counts              |
| MPHF Seeds        | 20 KB       | One u32 seed per bucket for displacement hashing          |
| MPHF Fingerprints | 59 KB       | One u16 fingerprint per slot for false-positive rejection |
| Confusion Pairs   | 4 KB        | 200 mined OCR error patterns with probabilities           |
| GRU Weights       | 37 KB       | 64-hidden-unit int8 quantized language model              |
| **Total**         | **~118 KB** |                                                           |

---

## Design Principles

- **Zero heap allocation at query time.** All scoring uses stack buffers and caller-provided slices. No garbage collection pressure.
- **O(1) dictionary lookup.** The MPHF maps every known word to a unique slot. Lookups are a hash, a table read, and a fingerprint comparison.
- **Tiny footprint.** The entire engine plus model fits in under 800 KB. It loads in milliseconds even on slow connections.
- **Offline compilation.** All expensive work (MPHF construction, GRU training, dictionary merging) happens ahead of time in Python. The runtime is pure lookup.
- **Modular TDD.** Each Zig module contains its own inline tests. Run `zig build test` to exercise all 53 tests across 9 modules.

---

## License

MIT. See [LICENSE](LICENSE) for the full text.

**Data sources:**

- Dataset: [ICDAR English Monograph OCR](https://www.kaggle.com/datasets/arjav007/icdar-eng) (CC-BY-SA-4.0)
- Word list: [dwyl/english-words](https://github.com/dwyl/english-words) (Unlicense)

# UPDATE DEVELOPED
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
