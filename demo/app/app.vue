<script setup lang="ts">
import {
  Eye,
  Zap,
  RotateCcw,
  Loader2,
  Check,
  AlertTriangle,
  Pencil,
  ClipboardPaste,
  ImagePlus,
  FileText,
  Crop,
  ScanText,
  ArrowDownToLine,
  Trash2,
} from 'lucide-vue-next'

const {
  isInitialized,
  isInitializing,
  initError,
  processError,
  results,
  tokenCount,
  initialize,
  detect,
  clear,
} = useMacula()

// Form state
const wasmUrl = ref('/wasm/ocr_wasm.wasm')
const artifactUrl = ref('/wasm/ocr_detector.bin')
const threshold = ref('5.0')
const inputText = ref('Teh quicK brovvn fox jumps ouer teh lazy d0g')
const ocrStatus = ref('')
const ocrLoading = ref(false)
const showSettings = ref(false)

// Bahasa OCR
const ocrLang = ref<'eng' | 'ind' | 'eng+ind'>('eng+ind')
const langOptions = [
  { value: 'eng+ind', label: '🌐 English + Indonesia' },
  { value: 'eng',     label: 'En English only' },
  { value: 'ind',     label: '🇮🇩 Indonesia only' },
]

// PDF state
const pdfLoading = ref(false)
const pdfStatus = ref('')
const pdfFileInput = ref<HTMLInputElement | null>(null)
let pdfDoc: any = null
const currentPage = ref(1)
const totalPages = ref(0)
// Crop / Preview state
const previewCanvas = ref<HTMLCanvasElement | null>(null)
const renderedCanvas = ref<HTMLCanvasElement | null>(null)
const showCropUI = ref(false)
const cropRect = ref({ x: 0, y: 0, w: 0, h: 0 })
const isDragging = ref(false)
const dragStart = ref({ x: 0, y: 0 })
const hasCrop = ref(false)
const zoomLevel = ref(1.0)
const rotation = ref(0) // 0, 90, 180, 270
const brightness = ref(1.4)
const contrastThreshold = ref(100)
const cropMode = ref<'image' | 'pdf'>('image')
// Kolom teks edit
const savedText = ref('')
const capsLock = ref(false)
const singleLine = ref(false)
// File tracking untuk rename & download
const uploadedFile = ref<File | null>(null)
const downloadUrl = ref('')
const downloadFilename = ref('')
const downloadReady = ref(false)
// Image blob
const imageBlob = ref<Blob | null>(null)
const imagePreviewUrl = ref<string | null>(null)
// Count stats
const validCount = computed(() => results.value.filter(r => r.status === 'valid').length)
const correctedCount = computed(() => results.value.filter(r => r.status === 'corrected').length)
const errorCount = computed(() => results.value.filter(r => r.status === 'error_detected').length)

async function onInit() {
  await initialize(wasmUrl.value, artifactUrl.value, parseFloat(threshold.value) || 5.0)
}

function jadikanSatuBaris() {
  if (!inputText.value.trim()) return
  inputText.value = inputText.value
    .split('\n')                    // pecah per baris
    .map(line => line.trim())       // hapus spasi di tiap ujung baris
    .filter(line => line.length > 0) // hapus baris kosong
    .join(' ')                      // gabung jadi 1 baris dengan spasi
}

function onDetect() {
  if (!inputText.value.trim()) return
  if (capsLock.value) inputText.value = inputText.value.toUpperCase()
  detect(inputText.value)
}

function onClear() {
  inputText.value = ''
  clear()
  ocrStatus.value = ''
  pdfStatus.value = ''
  showCropUI.value = false
  hasCrop.value = false
  imageBlob.value = null
  renderedCanvas.value = null
  if (imagePreviewUrl.value) {
    URL.revokeObjectURL(imagePreviewUrl.value)
    imagePreviewUrl.value = null
  }
}

// ─── TESSERACT ────────────────────────────────────────────────────────────────
async function loadTesseract() {
  if (!(window as any).Tesseract?.recognize) {
    await new Promise<void>((resolve, reject) => {
      const script = document.createElement('script')
      script.src = 'https://cdn.jsdelivr.net/npm/tesseract.js@5/dist/tesseract.min.js'
      script.onload = () => resolve()
      script.onerror = () => reject(new Error('Failed to load tesseract.js'))
      document.head.appendChild(script)
    })
  }
}

function preprocessCanvas(src: HTMLCanvasElement): HTMLCanvasElement {
  const out = document.createElement('canvas')
  out.width = src.width
  out.height = src.height
  const ctx = out.getContext('2d')!
  ctx.drawImage(src, 0, 0)

  const imageData = ctx.getImageData(0, 0, out.width, out.height)
  const data = imageData.data
  let min = 255, max = 0
  for (let i = 0; i < data.length; i += 4) {
    const gray = 0.299 * data[i] + 0.587 * data[i+1] + 0.114 * data[i+2]
    if (gray < min) min = gray
    if (gray > max) max = gray
  }

  const range = max - min || 1

  for (let i = 0; i < data.length; i += 4) {
    let gray = 0.299 * data[i] + 0.587 * data[i+1] + 0.114 * data[i+2]
    gray = ((gray - min) / range) * 255
    gray = Math.min(255, gray * brightness.value)
    const val = gray > contrastThreshold.value ? 255 : 0
    data[i] = data[i+1] = data[i+2] = val
  }

  ctx.putImageData(imageData, 0, 0)
  return out
}

async function runOcrOnCanvas(canvas: HTMLCanvasElement) {
  await loadTesseract()
  const processed = preprocessCanvas(canvas)
  const { data } = await (window as any).Tesseract.recognize(processed, ocrLang.value)
  return (data?.text || '').trim()
}

// ─── CROP HELPERS ─────────────────────────────────────────────────────────────
function getCanvasPos(e: MouseEvent | TouchEvent, canvas: HTMLCanvasElement) {
  const rect = canvas.getBoundingClientRect()
  const scaleX = canvas.width / rect.width
  const scaleY = canvas.height / rect.height
  const clientX = 'touches' in e ? e.touches[0].clientX : (e as MouseEvent).clientX
  const clientY = 'touches' in e ? e.touches[0].clientY : (e as MouseEvent).clientY
  return {
    x: (clientX - rect.left) * scaleX,
    y: (clientY - rect.top) * scaleY,
  }
}

function onCropMouseDown(e: MouseEvent | TouchEvent) {
  if (!previewCanvas.value) return
  e.preventDefault()
  const pos = getCanvasPos(e, previewCanvas.value)
  dragStart.value = pos
  cropRect.value = { x: pos.x, y: pos.y, w: 0, h: 0 }
  isDragging.value = true
  hasCrop.value = false
  drawPreview()
}

function onCropMouseMove(e: MouseEvent | TouchEvent) {
  if (!isDragging.value || !previewCanvas.value) return
  e.preventDefault()
  const pos = getCanvasPos(e, previewCanvas.value)
  cropRect.value = {
    x: Math.min(pos.x, dragStart.value.x),
    y: Math.min(pos.y, dragStart.value.y),
    w: Math.abs(pos.x - dragStart.value.x),
    h: Math.abs(pos.y - dragStart.value.y),
  }
  drawPreview()
}

function onCropMouseUp() {
  if (!isDragging.value) return
  isDragging.value = false
  if (cropRect.value.w > 10 && cropRect.value.h > 10) {
    hasCrop.value = true
  }
  drawPreview()
}

function drawPreview() {
  if (!previewCanvas.value || !renderedCanvas.value) return
  const src = renderedCanvas.value
  const ctx = previewCanvas.value.getContext('2d')!
  const rad = (rotation.value * Math.PI) / 180
  const isRotated = rotation.value === 90 || rotation.value === 270

  // Sesuaikan ukuran canvas dengan rotasi
  previewCanvas.value.width = isRotated ? src.height : src.width
  previewCanvas.value.height = isRotated ? src.width : src.height

  ctx.clearRect(0, 0, previewCanvas.value.width, previewCanvas.value.height)

  // Gambar dengan rotasi
  ctx.save()
  ctx.translate(previewCanvas.value.width / 2, previewCanvas.value.height / 2)
  ctx.rotate(rad)
  ctx.drawImage(src, -src.width / 2, -src.height / 2)
  ctx.restore()

  // Gambar crop overlay TANPA transform (koordinat layar langsung)
  if (cropRect.value.w > 0 && cropRect.value.h > 0) {
    const { x, y, w, h } = cropRect.value
    const cw = previewCanvas.value.width
    const ch = previewCanvas.value.height

    // 4 kotak gelap di luar area crop
    ctx.fillStyle = 'rgba(0,0,0,0.45)'
    ctx.fillRect(0, 0, cw, y)           // atas
    ctx.fillRect(0, y + h, cw, ch)      // bawah
    ctx.fillRect(0, y, x, h)            // kiri
    ctx.fillRect(x + w, y, cw, h)       // kanan

    // Border orange
    ctx.strokeStyle = '#f97316'
    ctx.lineWidth = 2
    ctx.strokeRect(x, y, w, h)
  }
}

function getCroppedCanvas(): HTMLCanvasElement {
  const src = renderedCanvas.value!
  if (!hasCrop.value || cropRect.value.w < 10) return src
  const out = document.createElement('canvas')
  out.width = cropRect.value.w
  out.height = cropRect.value.h
  out.getContext('2d')!.drawImage(
    src,
    cropRect.value.x, cropRect.value.y, cropRect.value.w, cropRect.value.h,
    0, 0, cropRect.value.w, cropRect.value.h,
  )
  return out
}

function resetCrop() {
  hasCrop.value = false
  cropRect.value = { x: 0, y: 0, w: 0, h: 0 }
  drawPreview()
}

// ─── IMAGE HANDLER ────────────────────────────────────────────────────────────
function onPaste(e: ClipboardEvent) {
  const items = e.clipboardData?.items || []
  for (const item of items) {
    if (item.type.startsWith('image/')) {
      const blob = item.getAsFile()
      if (blob) { loadImageToCanvas(blob); e.preventDefault() }
      return
    }
  }
}

function onImageFile(e: Event) {
  const file = (e.target as HTMLInputElement).files?.[0]
  if (file) { uploadedFile.value = file; loadImageToCanvas(file) }
}

async function loadImageToCanvas(blob: Blob) {
  imageBlob.value = blob
  cropMode.value = 'image'
  ocrStatus.value = 'Gambar dimuat — drag untuk crop area, lalu klik Extract.'
  const url = URL.createObjectURL(blob)
  const img = new Image()
  img.onload = () => {
    const c = document.createElement('canvas')
    c.width = img.naturalWidth
    c.height = img.naturalHeight
    c.getContext('2d')!.drawImage(img, 0, 0)
    renderedCanvas.value = c
    showCropUI.value = true
    nextTick(() => {
      if (!previewCanvas.value) return
      previewCanvas.value.width = c.width
      previewCanvas.value.height = c.height
      drawPreview()
    })
    URL.revokeObjectURL(url)
  }
  img.src = url
}

async function onExtractOcr() {
  if (!renderedCanvas.value) { ocrStatus.value = 'Pilih gambar dulu.'; return }
  ocrLoading.value = true
  ocrStatus.value = 'Menjalankan Scan...'
  try {
    const text = await runOcrOnCanvas(getCroppedCanvas())
    let result = capsLock.value ? text.toUpperCase() : text
    if (singleLine.value) {
      result = result.split('\n').map(l => l.trim()).filter(l => l.length > 0).join(' ')
    }
    inputText.value = result
    ocrStatus.value = text.length > 0
      ? `Selesai — ${text.length} karakter diekstrak${hasCrop.value ? ' (area terpilih)' : ''}.`
      : 'Tidak ada teks ditemukan.'
  }
  catch (err: any) { ocrStatus.value = `Scan gagal: ${err.message}` }
  finally { ocrLoading.value = false }
}

// ─── PDF HANDLER ──────────────────────────────────────────────────────────────
async function renderPage(pageNum: number) {
  if (!pdfDoc) return
  pdfLoading.value = true
  try {
    const page = await pdfDoc.getPage(pageNum)
    const viewport = page.getViewport({ scale: 2.0 })
    const c = document.createElement('canvas')
    c.width = viewport.width
    c.height = viewport.height
    await page.render({ canvasContext: c.getContext('2d')!, viewport }).promise
    renderedCanvas.value = c
    cropMode.value = 'pdf'
    showCropUI.value = true
    resetCrop()
    await nextTick()
    if (previewCanvas.value) {
      previewCanvas.value.width = c.width
      previewCanvas.value.height = c.height
      drawPreview()
    }
    pdfStatus.value = `Halaman ${pageNum} dari ${totalPages.value} — drag untuk crop.`
  }
  catch (err: any) { pdfStatus.value = `Gagal render halaman: ${err.message}` }
  finally { pdfLoading.value = false }
}

async function onPdfFile(e: Event) {
  const file = (e.target as HTMLInputElement).files?.[0]
  if (!file) return
  if (file.type !== 'application/pdf') { pdfStatus.value = 'File harus berformat PDF.'; return }
  uploadedFile.value = file

  pdfLoading.value = true
  pdfStatus.value = '1/2 Memuat library PDF...'
  inputText.value = ''
  showCropUI.value = false

  try {
    const pdfjsLib = await import('https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.4.168/pdf.min.mjs' as any)
    pdfjsLib.GlobalWorkerOptions.workerSrc = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.4.168/pdf.worker.min.mjs'

    pdfStatus.value = '2/2 Merender halaman PDF...'
    const arrayBuffer = await file.arrayBuffer()
    const pdf = await pdfjsLib.getDocument({ data: arrayBuffer }).promise
    pdfDoc = pdf
    totalPages.value = pdf.numPages
    currentPage.value = 1
    await renderPage(1)
  }
  catch (err: any) { pdfStatus.value = `Gagal memuat PDF: ${err.message}` }
  finally {
    pdfLoading.value = false
    if (pdfFileInput.value) pdfFileInput.value.value = ''
  }
}

async function onExtractPdf() {
  if (!renderedCanvas.value) return
  pdfLoading.value = true
  pdfStatus.value = 'Menjalankan Scan...'
  try {
    const text = await runOcrOnCanvas(getCroppedCanvas())
    let result = capsLock.value ? text.toUpperCase() : text
    if (singleLine.value) {
      result = result.split('\n').map(l => l.trim()).filter(l => l.length > 0).join(' ')
    }
    inputText.value = result
    pdfStatus.value = text.length > 0
      ? `Selesai — ${text.length} karakter diekstrak${hasCrop.value ? ' (area terpilih)' : ''}.`
      : 'Tidak ada teks ditemukan.'
  }
  catch (err: any) { pdfStatus.value = `Scan gagal: ${err.message}` }
  finally { pdfLoading.value = false }
}

function simpanSementara() {
  if (!uploadedFile.value) return

  const ext = uploadedFile.value.name.split('.').pop() || 'txt'
  const rawName = savedText.value.trim().slice(0, 300)
  const safeName = rawName
    .replace(/[/\\:*?"<>|]/g, '')
    .trim() || 'file'

  downloadFilename.value = `${safeName}.${ext}`

  if (downloadUrl.value) URL.revokeObjectURL(downloadUrl.value)
  downloadUrl.value = URL.createObjectURL(uploadedFile.value)
  downloadReady.value = true
}

// ─── BADGE ────────────────────────────────────────────────────────────────────
function statusBadgeVariant(status: string) {
  switch (status) {
    case 'valid': return 'success'
    case 'corrected': return 'warning'
    case 'error_detected': return 'error'
    default: return 'outline'
  }
}
function statusLabel(status: string) {
  switch (status) {
    case 'valid': return 'Valid'
    case 'corrected': return 'Corrected'
    case 'error_detected': return 'Error'
    default: return status
  }
}

onMounted(() => { document.addEventListener('paste', onPaste as any); onInit() })
onUnmounted(() => { document.removeEventListener('paste', onPaste as any); if (imagePreviewUrl.value) URL.revokeObjectURL(imagePreviewUrl.value) })
</script>

<template>
  <div class="min-h-screen bg-muted">
    <!-- Header -->
    <header class="sticky top-0 z-50 border-b border-border bg-background/95 backdrop-blur-sm">
      <div class="mx-auto flex h-14 max-w-4xl items-center justify-between px-4">
        <div class="flex items-center gap-2.5">
          <Eye class="h-5 w-5 text-primary" :stroke-width="2.5" />
          <span class="text-lg font-bold tracking-tight">SCAN TEKS</span>
        </div>
        <span class="hidden text-sm text-muted-foreground sm:inline">Scan Teks · Client-side WASM</span>
      </div>
    </header>

    <main class="mx-auto max-w-4xl space-y-4 px-4 py-6">
      <!-- Initialize -->
      <UiCard v-if="isInitializing || initError || showSettings || !isInitialized" class="p-5">
        <div class="flex items-center justify-between">
          <div>
            <h2 class="text-sm font-semibold">Model</h2>
            <p class="mt-0.5 text-xs text-muted-foreground">
              {{ isInitialized ? 'Ready to detect.' : 'Load the WASM engine and dictionary artifact.' }}
            </p>
          </div>
          <div class="flex items-center gap-2">
            <button class="text-xs text-muted-foreground underline-offset-2 hover:underline cursor-pointer" @click="showSettings = !showSettings">
              {{ showSettings ? 'Hide' : 'Settings' }}
            </button>
            <UiButton :disabled="isInitializing" size="sm" @click="onInit">
              <Loader2 v-if="isInitializing" class="h-3.5 w-3.5 animate-spin" />
              <Zap v-else class="h-3.5 w-3.5" />
              {{ isInitializing ? 'Loading…' : isInitialized ? 'Reinitialize' : 'Initialize' }}
            </UiButton>
          </div>
        </div>
        <Transition name="slide">
          <div v-if="showSettings" class="mt-4 grid gap-3 sm:grid-cols-3">
            <div>
              <label class="mb-1 block text-xs font-medium text-muted-foreground">WASM URL</label>
              <UiInput v-model="wasmUrl" class="text-xs" />
            </div>
            <div>
              <label class="mb-1 block text-xs font-medium text-muted-foreground">Artifact URL</label>
              <UiInput v-model="artifactUrl" class="text-xs" />
            </div>
            <div>
              <label class="mb-1 block text-xs font-medium text-muted-foreground">Threshold</label>
              <UiInput v-model="threshold" type="number" step="0.1" class="text-xs" />
            </div>
          </div>
        </Transition>
        <p v-if="initError" class="mt-3 text-xs text-destructive">{{ initError }}</p>
      </UiCard>

      <!-- Input Card -->
      <UiCard class="p-5">
        <div class="flex flex-wrap items-start justify-between gap-3">
          <div>
            <h2 class="text-sm font-semibold">Image / PDF Input</h2>
            <p class="mt-0.5 text-xs text-muted-foreground">Upload gambar atau PDF scan. Drag pada preview untuk crop area tertentu.</p>
          </div>
          <!-- Pilihan Bahasa -->
          <div class="flex items-center gap-2 shrink-0">
            <label class="text-xs text-muted-foreground font-medium whitespace-nowrap">Bahasa:</label>
            <select
              v-model="ocrLang"
              class="text-xs border border-border rounded-md px-2 py-1.5 bg-background text-foreground cursor-pointer hover:border-primary transition-colors"
            >
              <option v-for="opt in langOptions" :key="opt.value" :value="opt.value">{{ opt.label }}</option>
            </select>
          </div>
        </div>

        <!-- Tombol Upload -->
        <div class="mt-3 flex flex-wrap gap-2 items-center">
          <!-- Image -->
          <label class="inline-flex h-9 cursor-pointer items-center gap-1.5 rounded-lg border border-dashed border-border px-3 text-xs text-muted-foreground transition-colors hover:border-primary hover:text-primary">
            <ImagePlus class="h-3.5 w-3.5" />
            Choose image
            <input type="file" accept="image/*" class="hidden" @change="onImageFile" />
          </label>
          <UiButton variant="outline" size="sm" :disabled="!renderedCanvas || ocrLoading || cropMode !== 'image'" @click="onExtractOcr">
            <Loader2 v-if="ocrLoading && cropMode === 'image'" class="h-3.5 w-3.5 animate-spin" />
            <ClipboardPaste v-else class="h-3.5 w-3.5" />
            {{ hasCrop && cropMode === 'image' ? 'Extract Area' : 'Extract Text' }}
          </UiButton>

          <span class="text-muted-foreground text-xs">|</span>

          <!-- PDF -->
          <label
            class="inline-flex h-9 items-center gap-1.5 rounded-lg border border-dashed px-3 text-xs transition-colors"
            :class="pdfLoading ? 'cursor-not-allowed border-border text-muted-foreground opacity-50' : 'cursor-pointer border-border text-muted-foreground hover:border-primary hover:text-primary'"
          >
            <Loader2 v-if="pdfLoading" class="h-3.5 w-3.5 animate-spin" />
            <FileText v-else class="h-3.5 w-3.5" />
            {{ pdfLoading ? pdfStatus : 'Choose PDF (Scan)' }}
            <input ref="pdfFileInput" type="file" accept=".pdf" class="hidden" :disabled="pdfLoading" @change="onPdfFile" />
          </label>
          <UiButton variant="outline" size="sm" :disabled="!renderedCanvas || pdfLoading || cropMode !== 'pdf'" @click="onExtractPdf">
            <Loader2 v-if="pdfLoading && cropMode === 'pdf'" class="h-3.5 w-3.5 animate-spin" />
            <ScanText v-else class="h-3.5 w-3.5" />
            {{ hasCrop && cropMode === 'pdf' ? 'Extract Area' : 'Extract Semua' }}
          </UiButton>
        </div>

        <!-- Status -->
        <div class="mt-2 space-y-1">
          <p v-if="ocrStatus" class="text-xs text-muted-foreground">{{ ocrStatus }}</p>
          <p v-if="pdfStatus && !pdfLoading" class="text-xs" :class="pdfStatus.startsWith('Gagal') ? 'text-destructive' : 'text-muted-foreground'">{{ pdfStatus }}</p>
        </div>

        <!-- Preview + Crop Canvas -->
        <div v-if="showCropUI" class="mt-4">
        <!-- Navigasi halaman PDF -->
        <div v-if="totalPages > 1" class="flex items-center gap-2 mb-3">
          <button
            class="w-7 h-7 rounded border border-border text-xs hover:border-primary hover:text-primary transition-colors disabled:opacity-40"
            :disabled="currentPage <= 1 || pdfLoading"
            @click="currentPage = 1; renderPage(1)"
          >«</button>
          <button
            class="w-7 h-7 rounded border border-border text-xs hover:border-primary hover:text-primary transition-colors disabled:opacity-40"
            :disabled="currentPage <= 1 || pdfLoading"
            @click="currentPage--; renderPage(currentPage)"
          >‹</button>
          <span class="text-xs font-mono text-muted-foreground px-1">
            {{ currentPage }} / {{ totalPages }}
          </span>
          <button
            class="w-7 h-7 rounded border border-border text-xs hover:border-primary hover:text-primary transition-colors disabled:opacity-40"
            :disabled="currentPage >= totalPages || pdfLoading"
            @click="currentPage++; renderPage(currentPage)"
          >›</button>
          <button
            class="w-7 h-7 rounded border border-border text-xs hover:border-primary hover:text-primary transition-colors disabled:opacity-40"
            :disabled="currentPage >= totalPages || pdfLoading"
            @click="currentPage = totalPages; renderPage(totalPages)"
          >»</button>
          <Loader2 v-if="pdfLoading" class="h-3.5 w-3.5 animate-spin text-muted-foreground" />
        </div>
          <div class="flex items-center justify-between mb-2 flex-wrap gap-2">
            <p class="text-xs text-muted-foreground flex items-center gap-1.5">
              <Crop class="h-3 w-3" />
              Drag pada gambar untuk memilih area yang ingin di scan
            </p>
            <div class="flex items-center gap-3">
              <!-- Zoom control -->
              <div class="flex items-center gap-1.5">
                <label class="text-xs text-muted-foreground">Zoom:</label>
                <button
                  class="w-6 h-6 rounded border border-border text-xs hover:border-primary hover:text-primary transition-colors"
                  @click="zoomLevel = Math.max(0.3, zoomLevel - 0.1)"
                >−</button>
                <span class="text-xs font-mono w-10 text-center">{{ Math.round(zoomLevel * 100) }}%</span>
                <button
                  class="w-6 h-6 rounded border border-border text-xs hover:border-primary hover:text-primary transition-colors"
                  @click="zoomLevel = Math.min(2.0, zoomLevel + 0.1)"
                >+</button>
                <button
                  class="text-xs text-muted-foreground hover:underline cursor-pointer"
                  @click="zoomLevel = 1.0"
                >Reset</button>
                <!-- Rotate control -->
                <span class="text-border text-xs mx-1">|</span>
                <label class="text-xs text-muted-foreground">Rotate:</label>
                <button
                  class="w-6 h-6 rounded border border-border text-xs hover:border-primary hover:text-primary transition-colors"
                  title="Putar kiri 90°"
                  @click="rotation = (rotation - 90 + 360) % 360; resetCrop(); drawPreview()"
                >↺</button>
                <button
                  class="w-6 h-6 rounded border border-border text-xs hover:border-primary hover:text-primary transition-colors"
                  title="Putar kanan 90°"
                  @click="rotation = (rotation + 90) % 360; resetCrop(); drawPreview()"
                >↻</button>
                <span class="text-xs font-mono text-muted-foreground">{{ rotation }}°</span>
              </div>
              <button v-if="hasCrop" class="text-xs text-orange-500 hover:underline cursor-pointer" @click="resetCrop">
                ✂️ Reset crop
              </button>
            </div>
          </div>

          <!-- Canvas wrapper dengan scroll -->
          <div class="overflow-auto rounded-lg border border-border bg-muted/30" style="max-height: 600px;">
            <canvas
              ref="previewCanvas"
              class="cursor-crosshair"
              :style="{
                width: renderedCanvas ? Math.round(renderedCanvas.width * zoomLevel) + 'px' : 'auto',
                height: renderedCanvas ? Math.round(renderedCanvas.height * zoomLevel) + 'px' : 'auto',
                display: 'block',
              }"
              style="touch-action: none;"
              @mousedown="onCropMouseDown"
              @mousemove="onCropMouseMove"
              @mouseup="onCropMouseUp"
              @mouseleave="onCropMouseUp"
              @touchstart="onCropMouseDown"
              @touchmove="onCropMouseMove"
              @touchend="onCropMouseUp"
            />
          </div>
          <!-- Preprocessing controls -->
          <div class="mt-3 flex flex-wrap items-center gap-4 rounded-lg border border-border bg-muted/30 px-3 py-2">
            <span class="text-xs text-muted-foreground font-medium">🔧 Enhancement:</span>
            <div class="flex items-center gap-2">
              <label class="text-xs text-muted-foreground whitespace-nowrap">Brightness: {{ brightness.toFixed(1) }}x</label>
              <input
                type="range" min="0.5" max="3.0" step="0.1"
                :value="brightness"
                @input="brightness = parseFloat(($event.target as HTMLInputElement).value)"
                class="w-24 h-1.5 accent-primary cursor-pointer"
              />
            </div>
            <div class="flex items-center gap-2">
              <label class="text-xs text-muted-foreground whitespace-nowrap">Threshold: {{ contrastThreshold }}</label>
              <input
                type="range" min="50" max="200" step="5"
                :value="contrastThreshold"
                @input="contrastThreshold = parseInt(($event.target as HTMLInputElement).value)"
                class="w-24 h-1.5 accent-primary cursor-pointer"
              />
            </div>
            <button
              class="text-xs text-muted-foreground hover:underline cursor-pointer"
              @click="brightness = 1.4; contrastThreshold = 100"
            >Reset</button>
          </div>
          <p v-if="hasCrop" class="mt-1.5 text-xs text-orange-500 font-medium">
            ✂️ Area terpilih: {{ Math.round(cropRect.w) }} × {{ Math.round(cropRect.h) }} px
            — klik tombol Extract untuk scan area ini saja.
          </p>
        </div>
      </UiCard>

      <!-- Text Input + Run -->
      <UiCard class="p-5">
        <div class="flex items-center justify-between flex-wrap gap-2">
          <div class="flex items-center gap-2">
            <h2 class="text-sm font-semibold">Kolom Hasil Scan</h2>
            <button
              class="inline-flex items-center gap-1 px-2 py-0.5 rounded text-xs font-medium border transition-colors"
              :class="capsLock
                ? 'bg-primary text-primary-foreground border-primary'
                : 'bg-background text-muted-foreground border-border hover:border-primary'"
              @click="capsLock = !capsLock"
            >
              🔠 {{ capsLock ? 'CAPS ON' : 'CAPS OFF' }}
            </button>
          </div>
          <div class="flex gap-2">
            <button
              class="inline-flex items-center gap-1 px-2 py-0.5 rounded text-xs font-medium border transition-colors"
              :class="singleLine
                ? 'bg-primary text-primary-foreground border-primary'
                : 'bg-background text-muted-foreground border-border hover:border-primary'"
              @click="singleLine = !singleLine"
            >
              ↔ {{ singleLine ? '1 BARIS ON' : '1 BARIS OFF' }}
            </button>   
            <UiButton variant="ghost" size="sm" @click="onClear">
              <RotateCcw class="h-3.5 w-3.5" />
              Clear
            </UiButton>
            <UiButton size="sm" :disabled="!isInitialized || !inputText.trim()" @click="onDetect">
              <Zap class="h-3.5 w-3.5" />
              Run Detection
            </UiButton>
          </div>
        </div>
        <UiTextarea
          v-model="inputText"
          placeholder="Paste atau ketik teks scan di sini, atau upload gambar/PDF di atas…"
          class="mt-3 font-mono text-sm"
          :rows="4"
        />
      </UiCard>
      <!-- Kolom Teks Sementara -->
      <UiCard class="p-5">
        <div class="flex items-center justify-between flex-wrap gap-2">
          <div>
            <h2 class="text-sm font-semibold">Kolom Edit Teks</h2>
            <p class="mt-0.5 text-xs text-muted-foreground">
              Simpan hasil scan di sini sementara scan bagian lain.
              <span v-if="savedText" class="text-primary font-medium">{{ savedText.length }} karakter.</span>
            </p>
          </div>
          <div class="flex gap-2 flex-wrap">
            <UiButton
              variant="outline"
              size="sm"
              :disabled="!inputText.trim()"
              @click="simpanSementara()"
            >
            <ArrowDownToLine class="h-3.5 w-3.5" />
              Simpan
            </UiButton>
            <UiButton
              variant="ghost"
              size="sm"
              :disabled="!savedText"
              @click="savedText = ''"
            >
              <Trash2 class="h-3.5 w-3.5" />
              Hapus
            </UiButton>
            <a
              v-if="downloadReady && downloadUrl"
              :href="downloadUrl"
              :download="downloadFilename"
              class="inline-flex items-center gap-1.5 h-9 px-3 rounded-md border border-border text-xs text-muted-foreground hover:border-primary hover:text-primary transition-colors"
            >
            <ArrowDownToLine class="h-3.5 w-3.5" />
              Download ({{ downloadFilename }})
            </a>
          </div>
        </div>
        <UiTextarea
          v-model="savedText"
          placeholder="Teks hasil scan yang disimpan akan muncul di sini. Bisa diedit manual juga."
          class="mt-3 font-mono text-sm bg-muted/40"
          :rows="5"
        />
        <p class="mt-2 text-xs text-muted-foreground">
          💡 Alur: scan → <strong>Kolom Hasil</strong> → edit teks <strong>Edit & Copy Hasil Scan</strong> → scan bagian lain → simpan → download file 
        </p>
      </UiCard>

      <!-- Results -->
      <UiCard v-if="results.length > 0 || processError" class="overflow-hidden">
        <div class="border-b border-border px-5 py-4">
          <div class="flex items-center justify-between">
            <h2 class="text-sm font-semibold">Results</h2>
            <span class="text-xs text-muted-foreground">{{ tokenCount }} token{{ tokenCount !== 1 ? 's' : '' }} processed</span>
          </div>
          <div v-if="results.length > 0" class="mt-2 flex gap-4 text-xs">
            <span class="flex items-center gap-1 text-emerald-600"><Check class="h-3 w-3" /> {{ validCount }} valid</span>
            <span class="flex items-center gap-1 text-amber-600"><Pencil class="h-3 w-3" /> {{ correctedCount }} corrected</span>
            <span class="flex items-center gap-1 text-red-600"><AlertTriangle class="h-3 w-3" /> {{ errorCount }} errors</span>
          </div>
        </div>
        <p v-if="processError" class="px-5 py-3 text-xs text-destructive">{{ processError }}</p>
        <div v-if="results.length > 0" class="overflow-x-auto">
          <table class="w-full text-left text-sm">
            <thead>
              <tr class="border-b border-border text-xs text-muted-foreground">
                <th class="px-5 py-2 font-medium">#</th>
                <th class="px-5 py-2 font-medium">Token</th>
                <th class="px-5 py-2 font-medium">Status</th>
                <th class="px-5 py-2 font-medium">Correction</th>
                <th class="px-5 py-2 font-medium text-right">Confidence</th>
                <th class="px-5 py-2 font-medium text-right">Span</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(row, i) in results" :key="i" class="border-b border-border last:border-0 transition-colors hover:bg-muted/50">
                <td class="px-5 py-2.5 text-xs text-muted-foreground tabular-nums">{{ i + 1 }}</td>
                <td class="px-5 py-2.5 font-mono text-xs font-medium">{{ row.token }}</td>
                <td class="px-5 py-2.5">
                  <UiBadge :variant="statusBadgeVariant(row.status) as any">{{ statusLabel(row.status) }}</UiBadge>
                </td>
                <td class="px-5 py-2.5 font-mono text-xs">{{ row.correction ?? '—' }}</td>
                <td class="px-5 py-2.5 text-right font-mono text-xs tabular-nums">{{ row.confidence.toFixed(4) }}</td>
                <td class="px-5 py-2.5 text-right text-xs text-muted-foreground tabular-nums">{{ row.start }}–{{ row.start + row.len }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </UiCard>
    </main>

    <footer class="border-t border-border py-6 text-center text-xs text-muted-foreground">
      Scan Teks. Powered by Zig + WebAssembly.
    </footer>
  </div>
</template>

<style>
.slide-enter-active, .slide-leave-active { transition: all 0.2s ease; overflow: hidden; }
.slide-enter-from, .slide-leave-to { opacity: 0; max-height: 0; margin-top: 0; }
.slide-enter-to, .slide-leave-from { opacity: 1; max-height: 200px; }
</style>