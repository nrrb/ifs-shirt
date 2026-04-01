<script setup>
import { ref, computed, onMounted, nextTick, watch } from 'vue'

// --- IFS Corner Systems (defaults) ---
const DEFAULTS = {
  fern: [
    { a: 0.00, b: 0.00, c: 0.00, d: 0.16, tx: 0.00, ty: 0.00, p: 0.01 },
    { a: 0.85, b: 0.04, c: -0.04, d: 0.85, tx: 0.00, ty: 1.60, p: 0.85 },
    { a: 0.20, b: -0.26, c: 0.23, d: 0.22, tx: 0.00, ty: 1.60, p: 0.07 },
    { a: -0.15, b: 0.28, c: 0.26, d: 0.24, tx: 0.00, ty: 0.44, p: 0.07 },
  ],
  sierpinski: [
    { a: 0.50, b: 0.00, c: 0.00, d: 0.50, tx: 0.00, ty: 0.00, p: 0.34 },
    { a: 0.50, b: 0.00, c: 0.00, d: 0.50, tx: 0.25, ty: 0.50, p: 0.33 },
    { a: 0.50, b: 0.00, c: 0.00, d: 0.50, tx: 0.50, ty: 0.00, p: 0.33 },
    { a: 0.00, b: 0.00, c: 0.00, d: 0.00, tx: 0.50, ty: 0.50, p: 0.00 },
  ],
  snowflake: [
    { a: 0.333, b: 0.000, c: 0.000, d: 0.333, tx: 0.000, ty: 0.000, p: 0.25 },
    { a: 0.167, b: -0.289, c: 0.289, d: 0.167, tx: 0.333, ty: 0.000, p: 0.25 },
    { a: 0.167, b: 0.289, c: -0.289, d: 0.167, tx: 0.500, ty: 0.289, p: 0.25 },
    { a: 0.333, b: 0.000, c: 0.000, d: 0.333, tx: 0.667, ty: 0.000, p: 0.25 },
  ],
  dragon: [
    { a: 0.00, b: -0.50, c: 0.50, d: 0.00, tx: 0.00, ty: 0.00, p: 0.25 },
    { a: 0.00, b:  0.50, c: -0.50, d: 0.00, tx: 0.50, ty: 0.50, p: 0.25 },
    { a: 0.00, b:  0.50, c: -0.50, d: 0.00, tx: 1.00, ty: 0.00, p: 0.25 },
    { a: 0.00, b: -0.50, c: 0.50, d: 0.00, tx: 0.50, ty: 0.00, p: 0.25 },
  ],
}

// Live corner fns — start as deep copies of defaults
const cornerFns = ref({
  fern:       DEFAULTS.fern.map(f => ({ ...f })),
  sierpinski: DEFAULTS.sierpinski.map(f => ({ ...f })),
  snowflake:  DEFAULTS.snowflake.map(f => ({ ...f })),
  dragon:     DEFAULTS.dragon.map(f => ({ ...f })),
})

// Track which corners have been overridden
const overridden = ref({ fern: false, sierpinski: false, snowflake: false, dragon: false })

// --- IFS math helpers ---
function buildCumProbs(fns) {
  const cum = []
  let sum = 0
  for (const fn of fns) { sum += fn.p; cum.push(sum) }
  return cum
}

function pickFn(fns, cum) {
  const r = Math.random()
  for (let i = 0; i < cum.length; i++) {
    if (r < cum[i]) return fns[i]
  }
  return fns[fns.length - 1]
}

function computeBBox(fns, warmupIter = 5000) {
  let x = 0, y = 0
  let minX = Infinity, maxX = -Infinity, minY = Infinity, maxY = -Infinity
  const pCum = buildCumProbs(fns)
  for (let i = 0; i < warmupIter; i++) {
    const fn = pickFn(fns, pCum)
    const nx = fn.a * x + fn.b * y + fn.tx
    const ny = fn.c * x + fn.d * y + fn.ty
    x = nx; y = ny
    if (i > 50) {
      if (x < minX) minX = x
      if (x > maxX) maxX = x
      if (y < minY) minY = y
      if (y > maxY) maxY = y
    }
  }
  return { minX, maxX, minY, maxY }
}

// s = i/(N-1): 0=fern/snowflake column, 1=sierpinski/dragon column
// t = j/(N-1): 0=fern/sierpinski row,   1=snowflake/dragon row
function interpolateIFS(s, t) {
  const w00 = (1 - s) * (1 - t) // fern
  const w10 = s * (1 - t)        // sierpinski
  const w01 = (1 - s) * t        // snowflake
  const w11 = s * t               // dragon

  const f_  = cornerFns.value.fern
  const si_ = cornerFns.value.sierpinski
  const sn_ = cornerFns.value.snowflake
  const dr_ = cornerFns.value.dragon

  const result = []
  for (let k = 0; k < 4; k++) {
    const f = f_[k], si = si_[k], sn = sn_[k], dr = dr_[k]
    result.push({
      a:  w00*f.a  + w10*si.a  + w01*sn.a  + w11*dr.a,
      b:  w00*f.b  + w10*si.b  + w01*sn.b  + w11*dr.b,
      c:  w00*f.c  + w10*si.c  + w01*sn.c  + w11*dr.c,
      d:  w00*f.d  + w10*si.d  + w01*sn.d  + w11*dr.d,
      tx: w00*f.tx + w10*si.tx + w01*sn.tx + w11*dr.tx,
      ty: w00*f.ty + w10*si.ty + w01*sn.ty + w11*dr.ty,
      p:  w00*f.p  + w10*si.p  + w01*sn.p  + w11*dr.p,
    })
  }
  const pSum = result.reduce((acc, fn) => acc + fn.p, 0)
  if (pSum > 0) result.forEach(fn => fn.p /= pSum)
  return result
}

function parseColor(hex) {
  const c = hex.replace('#', '')
  return {
    r: parseInt(c.substring(0, 2), 16),
    g: parseInt(c.substring(2, 4), 16),
    b: parseInt(c.substring(4, 6), 16),
  }
}

function renderIFS(ctx, fns, iters, pointColor, bgColor, width, height, transparentBg = false) {
  if (transparentBg) {
    ctx.clearRect(0, 0, width, height)
  } else {
    ctx.fillStyle = bgColor
    ctx.fillRect(0, 0, width, height)
  }

  const bbox = computeBBox(fns, 5000)
  const bw = bbox.maxX - bbox.minX || 1
  const bh = bbox.maxY - bbox.minY || 1

  const imageData = ctx.getImageData(0, 0, width, height)
  const data = imageData.data
  const col = parseColor(pointColor)
  const pCum = buildCumProbs(fns)

  let x = 0, y = 0
  for (let i = 0; i < iters + 50; i++) {
    const fn = pickFn(fns, pCum)
    const nx = fn.a * x + fn.b * y + fn.tx
    const ny = fn.c * x + fn.d * y + fn.ty
    x = nx; y = ny
    if (i < 50) continue

    const nx01 = (x - bbox.minX) / bw
    const ny01 = (y - bbox.minY) / bh
    const px = Math.floor(nx01 * (width - 1))
    const py = Math.floor((1 - ny01) * (height - 1))

    if (px >= 0 && px < width && py >= 0 && py < height) {
      const idx = (py * width + px) * 4
      data[idx]     = col.r
      data[idx + 1] = col.g
      data[idx + 2] = col.b
      data[idx + 3] = 255
    }
  }
  ctx.putImageData(imageData, 0, 0)
}

function circleOffsets(size) {
  if (size <= 1) return [[0, 0]]
  const r = (size - 1) / 2
  const offsets = []
  for (let dy = -Math.ceil(r); dy <= Math.ceil(r); dy++) {
    for (let dx = -Math.ceil(r); dx <= Math.ceil(r); dx++) {
      if (dx * dx + dy * dy <= r * r + r * 0.5) offsets.push([dx, dy])
    }
  }
  return offsets
}

async function renderIFSAsync(ctx, fns, iters, pointColor, bgColor, width, height, transparentBg = false, onProgress = null, cancelToken = null, additive = false, pointSize = 1) {
  if (transparentBg) {
    ctx.clearRect(0, 0, width, height)
  } else {
    ctx.fillStyle = bgColor
    ctx.fillRect(0, 0, width, height)
  }

  const bbox = computeBBox(fns, 5000)
  const bw = bbox.maxX - bbox.minX || 1
  const bh = bbox.maxY - bbox.minY || 1

  const imageData = ctx.getImageData(0, 0, width, height)
  const data = imageData.data
  const col = parseColor(pointColor)
  const pCum = buildCumProbs(fns)

  const CHUNK = 50000
  const offsets = circleOffsets(pointSize)
  let x = 0, y = 0
  let plotted = 0

  for (let i = 0; i < iters + 50; i++) {
    const fn = pickFn(fns, pCum)
    const nx = fn.a * x + fn.b * y + fn.tx
    const ny = fn.c * x + fn.d * y + fn.ty
    x = nx; y = ny
    if (i < 50) continue

    const nx01 = (x - bbox.minX) / bw
    const ny01 = (y - bbox.minY) / bh
    const cx = Math.floor(nx01 * (width - 1))
    const cy = Math.floor((1 - ny01) * (height - 1))

    for (const [dx, dy] of offsets) {
      const px = cx + dx
      const py = cy + dy
      if (px >= 0 && px < width && py >= 0 && py < height) {
        const idx = (py * width + px) * 4
        if (additive) {
          data[idx]     = Math.round(data[idx]     * 0.9 + col.r * 0.1)
          data[idx + 1] = Math.round(data[idx + 1] * 0.9 + col.g * 0.1)
          data[idx + 2] = Math.round(data[idx + 2] * 0.9 + col.b * 0.1)
          data[idx + 3] = Math.min(255, data[idx + 3] + 26)
        } else {
          data[idx]     = col.r
          data[idx + 1] = col.g
          data[idx + 2] = col.b
          data[idx + 3] = 255
        }
      }
    }

    plotted++
    if (plotted % CHUNK === 0) {
      if (cancelToken && cancelToken.cancelled) {
        ctx.putImageData(imageData, 0, 0)
        if (onProgress) onProgress(plotted)
        return
      }
      if (onProgress) onProgress(plotted)
      ctx.putImageData(imageData, 0, 0)
      await new Promise(r => setTimeout(r, 0))
    }
  }

  ctx.putImageData(imageData, 0, 0)
  if (onProgress) onProgress(plotted)
}

// --- PNG export helpers ---
function makeCrcTable() {
  const table = new Uint32Array(256)
  for (let i = 0; i < 256; i++) {
    let c = i
    for (let j = 0; j < 8; j++) c = (c & 1) ? (0xEDB88320 ^ (c >>> 1)) : (c >>> 1)
    table[i] = c
  }
  return table
}
const CRC_TABLE = makeCrcTable()

function crc32(data) {
  let crc = 0xFFFFFFFF
  for (let i = 0; i < data.length; i++) crc = (crc >>> 8) ^ CRC_TABLE[(crc ^ data[i]) & 0xFF]
  return (crc ^ 0xFFFFFFFF) >>> 0
}

function injectPhys(dataURL, ppm) {
  const b64 = dataURL.split(',')[1]
  const binary = atob(b64)
  const bytes = new Uint8Array(binary.length)
  for (let i = 0; i < binary.length; i++) bytes[i] = binary.charCodeAt(i)

  const insertAt = 33
  const physData = new Uint8Array(9)
  const dv = new DataView(physData.buffer)
  dv.setUint32(0, ppm)
  dv.setUint32(4, ppm)
  physData[8] = 1

  const typeBytes = new Uint8Array([0x70, 0x48, 0x59, 0x73]) // 'pHYs'
  const combined = new Uint8Array([...typeBytes, ...physData])
  const checksum = crc32(combined)

  const chunk = new Uint8Array(4 + 4 + 9 + 4)
  const cv = new DataView(chunk.buffer)
  cv.setUint32(0, 9)
  chunk.set(typeBytes, 4)
  chunk.set(physData, 8)
  cv.setUint32(17, checksum)

  const out = new Uint8Array(bytes.length + chunk.length)
  out.set(bytes.slice(0, insertAt))
  out.set(chunk, insertAt)
  out.set(bytes.slice(insertAt), insertAt + chunk.length)

  return new Blob([out], { type: 'image/png' })
}

// --- IFS paste parser ---
function parseIFSPaste(text) {
  const lines = text.split('\n')
  const rows = []

  for (const raw of lines) {
    const line = raw.trim()
    if (!line || line.startsWith('#')) continue

    let cleaned = line
      .replace(/^[-*•]\s*/, '')
      .replace(/^Slot\s*\d+\s*:\s*/i, '')
      .replace(/[\[\]]/g, '')
      .trim()

    const nums = cleaned.match(/-?\d+(\.\d+)?([eE][+-]?\d+)?/g)
    if (!nums || nums.length === 0) continue
    if (nums.length !== 7) throw `Expected 7 numbers per line, got ${nums.length} on: "${raw.trim()}"`

    rows.push(nums.map(Number))
  }

  if (rows.length !== 4) throw `Expected exactly 4 functions, found ${rows.length}`

  for (const [i, row] of rows.entries()) {
    if (row[6] < 0) throw `Function ${i + 1}: p value (${row[6]}) must be non-negative`
  }

  const pSum = rows.reduce((s, r) => s + r[6], 0)
  let warning = null
  if (Math.abs(pSum - 1) > 0.001) {
    warning = `p values sum to ${pSum.toFixed(4)}, auto-renormalizing`
  }

  const fns = rows.map(([a, b, c, d, tx, ty, p]) => {
    const pNorm = pSum > 0 ? p / pSum : 0.25
    return { a, b, c, d, tx, ty, p: pNorm }
  })

  return { fns, warning }
}

// --- Mode toggle ---
const singleMode = ref(false)

// --- Grid mode state ---
const N = ref(5)
const iterationsIndex = ref(4) // 10^4 = 10,000
const iterations = computed(() => Math.pow(10, iterationsIndex.value))
const pointColor = ref('#ffffff')
const bgColor = ref('#0a0a0f')
const invertColors = ref(false)
const exportInches = ref(12)
const status = ref('')

const effectivePointColor = computed(() => invertColors.value ? '#000000' : pointColor.value)
const effectiveBgColor    = computed(() => invertColors.value ? '#ffffff' : bgColor.value)
const iterationsLabel = computed(() => iterations.value.toLocaleString())

// --- Single mode state ---
const SINGLE_IFS_OPTIONS = [
  { key: 'fern',       label: 'Fern' },
  { key: 'sierpinski', label: 'Sierpinski' },
  { key: 'snowflake',  label: 'Snowflake' },
  { key: 'dragon',     label: 'Dragon' },
]
const singleSelectedIFS = ref('fern')
const singleIterationsIndex = ref(4) // 10^4 = 10,000
const singleIterations = computed(() => Math.pow(10, singleIterationsIndex.value))
const singleIterationsLabel = computed(() => singleIterations.value.toLocaleString())
const singleColorMode = ref('white-on-black') // 'white-on-black' | 'black-on-white'
const singleTransparentBg = ref(false)
const singleAdditive = ref(false)
const singlePointSize = ref(1)
const singleExportInches = ref(12)
const singleCanvasRef = ref(null)
const singleProgress = ref(0)
const singleRendering = ref(false)
let singleCancelToken = null

const singlePointColor = computed(() => singleColorMode.value === 'white-on-black' ? '#ffffff' : '#000000')
const singleBgColor    = computed(() => singleColorMode.value === 'white-on-black' ? '#000000' : '#ffffff')
const singleCanvasSize = computed(() => Math.max(200, Math.floor(Math.min(window.innerWidth - 280, window.innerHeight - 40))))

// --- Override panel state ---
const CORNER_OPTIONS = [
  { key: 'fern',       label: 'Fern (0,0)' },
  { key: 'sierpinski', label: 'Sierpinski (1,0)' },
  { key: 'snowflake',  label: 'Snowflake (0,1)' },
  { key: 'dragon',     label: 'Dragon (1,1)' },
]
const overrideCorner = ref('fern')
const overrideText = ref('')
const overrideStatus = ref(null)

function applyOverride() {
  try {
    const { fns, warning } = parseIFSPaste(overrideText.value)
    cornerFns.value[overrideCorner.value] = fns
    overridden.value[overrideCorner.value] = true
    overrideStatus.value = {
      ok: true,
      message: warning ? `4 functions loaded (${warning})` : '4 functions loaded',
    }
    renderAll()
  } catch (err) {
    overrideStatus.value = { ok: false, message: String(err) }
  }
}

function resetCorner() {
  const key = overrideCorner.value
  cornerFns.value[key] = DEFAULTS[key].map(f => ({ ...f }))
  overridden.value[key] = false
  overrideStatus.value = null
  overrideText.value = ''
  renderAll()
}

// --- Canvas refs & grid ---
const canvasRefs = ref({})

function setCanvasRef(el, key) {
  if (el) canvasRefs.value[key] = el
  else delete canvasRefs.value[key]
}

const gridCells = computed(() => {
  const cells = []
  for (let j = N.value - 1; j >= 0; j--) {
    for (let i = 0; i < N.value; i++) {
      cells.push({ i, j, key: `${i}-${j}` })
    }
  }
  return cells
})

const cellSize = computed(() => {
  const maxPx = Math.min(window.innerWidth - 260, window.innerHeight - 40)
  return Math.max(32, Math.floor(maxPx / N.value))
})

// --- Grid render/export ---
async function renderAll() {
  const total = N.value * N.value
  let done = 0
  for (const cell of gridCells.value) {
    status.value = `Rendering ${done + 1}/${total}...`
    await nextTick()
    await new Promise(r => setTimeout(r, 0))

    const canvas = canvasRefs.value[cell.key]
    if (!canvas) continue

    const sz = cellSize.value
    canvas.width = sz
    canvas.height = sz
    const ctx = canvas.getContext('2d')

    const s = N.value > 1 ? cell.i / (N.value - 1) : 0.5
    const t = N.value > 1 ? cell.j / (N.value - 1) : 0.5
    const fns = interpolateIFS(s, t)

    renderIFS(ctx, fns, iterations.value, effectivePointColor.value, effectiveBgColor.value, sz, sz)
    done++
  }
  status.value = 'Done.'
  setTimeout(() => { status.value = '' }, 2000)
}

async function exportGrid() {
  const totalPx = Math.round(300 * exportInches.value)
  const cellPx = Math.floor(totalPx / N.value)
  const offscreen = document.createElement('canvas')
  offscreen.width = totalPx
  offscreen.height = totalPx
  const octx = offscreen.getContext('2d')

  const total = N.value * N.value
  let done = 0
  for (const cell of gridCells.value) {
    status.value = `Exporting ${done + 1}/${total}...`
    await new Promise(r => setTimeout(r, 0))

    const s = N.value > 1 ? cell.i / (N.value - 1) : 0.5
    const t = N.value > 1 ? cell.j / (N.value - 1) : 0.5
    const fns = interpolateIFS(s, t)

    const tmp = document.createElement('canvas')
    tmp.width = cellPx
    tmp.height = cellPx
    renderIFS(tmp.getContext('2d'), fns, iterations.value, effectivePointColor.value, effectiveBgColor.value, cellPx, cellPx, invertColors.value)

    const col = cell.i
    const row = (N.value - 1) - cell.j
    octx.drawImage(tmp, col * cellPx, row * cellPx)
    done++
  }

  status.value = 'Encoding PNG...'
  await new Promise(r => setTimeout(r, 0))

  const dataURL = offscreen.toDataURL('image/png')
  const blob = injectPhys(dataURL, 2953)

  const a = document.createElement('a')
  a.href = URL.createObjectURL(blob)
  a.download = 'ifs-grid.png'
  a.click()
  URL.revokeObjectURL(a.href)
  status.value = 'Exported!'
  setTimeout(() => { status.value = '' }, 2000)
}

// --- Single mode render/export ---
async function renderSingle() {
  const canvas = singleCanvasRef.value
  if (!canvas) return
  const sz = singleCanvasSize.value
  canvas.width = sz
  canvas.height = sz
  const ctx = canvas.getContext('2d')
  const fns = cornerFns.value[singleSelectedIFS.value]
  if (singleCancelToken) singleCancelToken.cancelled = true
  singleCancelToken = { cancelled: false }
  const token = singleCancelToken
  singleProgress.value = 0
  singleRendering.value = true
  status.value = 'Rendering...'
  await nextTick()
  await renderIFSAsync(ctx, fns, singleIterations.value, singlePointColor.value, singleBgColor.value, sz, sz, singleTransparentBg.value, (n) => { singleProgress.value = n }, token, singleAdditive.value, singlePointSize.value)
  singleRendering.value = false
  status.value = token.cancelled ? 'Stopped.' : 'Done.'
  setTimeout(() => { status.value = ''; singleProgress.value = 0 }, 2000)
}

function stopSingle() {
  if (singleCancelToken) singleCancelToken.cancelled = true
}

async function exportSingle() {
  const totalPx = Math.round(300 * singleExportInches.value)
  const offscreen = document.createElement('canvas')
  offscreen.width = totalPx
  offscreen.height = totalPx
  const ctx = offscreen.getContext('2d')
  const fns = cornerFns.value[singleSelectedIFS.value]
  status.value = 'Rendering export...'
  await new Promise(r => setTimeout(r, 0))
  renderIFS(ctx, fns, singleIterations.value, singlePointColor.value, singleBgColor.value, totalPx, totalPx, singleTransparentBg.value)
  status.value = 'Encoding PNG...'
  await new Promise(r => setTimeout(r, 0))
  const dataURL = offscreen.toDataURL('image/png')
  const blob = injectPhys(dataURL, 11811) // 300 DPI = 11811 px/m
  const a = document.createElement('a')
  a.href = URL.createObjectURL(blob)
  a.download = `ifs-${singleSelectedIFS.value}.png`
  a.click()
  URL.revokeObjectURL(a.href)
  status.value = 'Exported!'
  setTimeout(() => { status.value = '' }, 2000)
}

// --- Debounced rendering ---
let renderDebounceTimer = null
function debouncedRender() {
  clearTimeout(renderDebounceTimer)
  renderDebounceTimer = setTimeout(() => renderAll(), 1000)
}

let singleRenderDebounceTimer = null
function debouncedRenderSingle() {
  clearTimeout(singleRenderDebounceTimer)
  singleRenderDebounceTimer = setTimeout(() => renderSingle(), 1000)
}

watch([N, iterationsIndex], debouncedRender)
watch([singleSelectedIFS, singleIterationsIndex, singleColorMode, singleTransparentBg, singleAdditive, singlePointSize], () => {
  if (singleMode.value) debouncedRenderSingle()
})
watch(singleMode, (val) => {
  if (val) nextTick(() => renderSingle())
})

onMounted(() => {
  renderAll()
})
</script>

<template>
  <div class="app">
    <!-- Canvas area -->
    <div class="canvas-area">
      <!-- Grid mode -->
      <div v-if="!singleMode" class="grid-area">
        <div class="grid-wrapper">
          <span class="clabel clabel-tl">
            Snowflake<span v-if="overridden.snowflake" class="override-dot" title="Overridden">*</span>
          </span>
          <span class="clabel clabel-tr">
            Dragon<span v-if="overridden.dragon" class="override-dot" title="Overridden">*</span>
          </span>
          <div
            class="grid"
            :style="{
              gridTemplateColumns: `repeat(${N}, ${cellSize}px)`,
              gridTemplateRows: `repeat(${N}, ${cellSize}px)`,
            }"
          >
            <div
              v-for="cell in gridCells"
              :key="cell.key"
              class="cell"
              :style="{ width: cellSize + 'px', height: cellSize + 'px' }"
            >
              <canvas
                :ref="el => setCanvasRef(el, cell.key)"
                :width="cellSize"
                :height="cellSize"
              />
            </div>
          </div>
          <span class="clabel clabel-bl">
            Fern<span v-if="overridden.fern" class="override-dot" title="Overridden">*</span>
          </span>
          <span class="clabel clabel-br">
            Sierpinski<span v-if="overridden.sierpinski" class="override-dot" title="Overridden">*</span>
          </span>
        </div>
      </div>

      <!-- Single mode -->
      <div v-else class="single-area" :class="singleTransparentBg ? 'checkerboard' : ''" :style="{ background: singleTransparentBg ? undefined : singleBgColor }">
        <canvas ref="singleCanvasRef" :width="singleCanvasSize" :height="singleCanvasSize" class="single-canvas" />
      </div>
    </div>

    <!-- Controls sidebar -->
    <div class="controls">
      <!-- Mode toggle -->
      <button class="mode-toggle" :class="{ active: singleMode }" @click="singleMode = !singleMode">
        {{ singleMode ? 'Single IFS' : 'IFS Grid' }}
      </button>

      <!-- Single mode controls -->
      <template v-if="singleMode">
        <h2>Single IFS</h2>

        <label>
          IFS system
          <select v-model="singleSelectedIFS">
            <option v-for="opt in SINGLE_IFS_OPTIONS" :key="opt.key" :value="opt.key">
              {{ opt.label }}{{ overridden[opt.key] ? ' *' : '' }}
            </option>
          </select>
        </label>

        <label>
          Iterations: <strong>{{ singleIterationsLabel }}</strong>
          <span v-if="singleProgress > 0" class="progress-count">
            {{ singleProgress.toLocaleString() }} drawn
            <button v-if="singleRendering" class="btn-stop" @click="stopSingle">Stop</button>
          </span>
          <input type="range" min="0" max="7" step="1" v-model.number="singleIterationsIndex" />
        </label>

        <label>
          Colors
          <select v-model="singleColorMode">
            <option value="white-on-black">White on Black</option>
            <option value="black-on-white">Black on White</option>
          </select>
        </label>

        <label class="label-row">
          <input type="checkbox" v-model="singleTransparentBg" />
          Transparent background
        </label>

        <label class="label-row">
          <input type="checkbox" v-model="singleAdditive" />
          Additive blending
        </label>

        <label>
          Point size: <strong>{{ singlePointSize }}</strong>
          <input type="range" min="1" max="10" step="1" v-model.number="singlePointSize" />
        </label>

        <label>
          Export size (inches)
          <input type="number" min="1" max="60" step="1" v-model.number="singleExportInches" />
          <span class="export-px">{{ (300 * singleExportInches).toLocaleString() }} × {{ (300 * singleExportInches).toLocaleString() }} px</span>
        </label>

        <div v-if="status" class="status">{{ status }}</div>

        <div class="button-row">
          <button @click="renderSingle">Re-render</button>
          <button @click="exportSingle">Export PNG</button>
        </div>
      </template>

      <!-- Grid mode controls -->
      <template v-else>
        <h2>IFS Grid</h2>

        <label>
          Grid size (N): <strong>{{ N }}</strong>
          <input type="range" min="2" max="30" v-model.number="N" />
        </label>

        <label>
          Iterations: <strong>{{ iterationsLabel }}</strong>
          <input type="range" min="0" max="6" step="1" v-model.number="iterationsIndex" />
        </label>

        <label>
          Point color
          <input type="color" v-model="pointColor" />
        </label>

        <label>
          Background color
          <input type="color" v-model="bgColor" />
        </label>

        <label class="label-row">
          <input type="checkbox" v-model="invertColors" />
          Invert colors
          <span class="invert-hint">(export: transparent bg)</span>
        </label>

        <label>
          Export size (inches)
          <input type="number" min="1" max="60" step="1" v-model.number="exportInches" />
          <span class="export-px">{{ (300 * exportInches).toLocaleString() }} × {{ (300 * exportInches).toLocaleString() }} px</span>
        </label>

        <div v-if="status" class="status">{{ status }}</div>

        <div class="button-row">
          <button @click="renderAll">Re-render</button>
          <button @click="exportGrid">Export PNG</button>
        </div>

        <!-- Override panel -->
        <div class="divider" />
        <h3>Override Corner</h3>

        <label>
          Corner
          <select v-model="overrideCorner" @change="overrideStatus = null">
            <option v-for="opt in CORNER_OPTIONS" :key="opt.key" :value="opt.key">
              {{ opt.label }}{{ overridden[opt.key] ? ' *' : '' }}
            </option>
          </select>
        </label>

        <label>
          Paste functions
          <textarea
            v-model="overrideText"
            placeholder="Paste 4 IFS functions here&#10;&#10;Accepted formats:&#10;- Slot 1: [a, b, c, d, tx, ty, p]&#10;[a, b, c, d, tx, ty, p]&#10;a b c d tx ty p"
            rows="8"
            spellcheck="false"
          />
        </label>

        <div v-if="overrideStatus" class="override-status" :class="overrideStatus.ok ? 'ok' : 'err'">
          {{ overrideStatus.ok ? '✓' : '✗' }} {{ overrideStatus.message }}
        </div>

        <div class="button-row">
          <button @click="applyOverride">Apply</button>
          <button class="btn-reset" @click="resetCorner">Reset</button>
        </div>
      </template>
    </div>
  </div>
</template>

<style>
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
html, body, #app { width: 100%; height: 100%; background: #0a0a0f; color: #ccc; font-family: system-ui, sans-serif; }
</style>

<style scoped>
.app {
  display: flex;
  height: 100vh;
  overflow: hidden;
}

.canvas-area {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: auto;
  padding: 16px;
}

.grid-area {
  display: flex;
  align-items: center;
  justify-content: center;
}

.single-area {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100%;
  height: 100%;
}

.single-area.checkerboard {
  background-image: repeating-conic-gradient(#444 0% 25%, #222 0% 50%);
  background-size: 20px 20px;
}

.single-canvas {
  display: block;
  max-width: 100%;
  max-height: 100%;
}

.grid {
  display: grid;
  gap: 2px;
  line-height: 0;
}

.cell {
  position: relative;
  overflow: hidden;
  background: #111;
}

.cell canvas {
  display: block;
  width: 100%;
  height: 100%;
}

.grid-wrapper {
  display: inline-grid;
  grid-template-areas:
    "tl . tr"
    ". grid ."
    "bl . br";
  grid-template-columns: max-content 1fr max-content;
  grid-template-rows: max-content 1fr max-content;
  gap: 6px;
  align-items: center;
}

.grid { grid-area: grid; }

.clabel {
  font-size: 11px;
  color: rgba(255,255,255,0.6);
  white-space: nowrap;
}
.clabel-tl { grid-area: tl; align-self: end; justify-self: start; }
.clabel-tr { grid-area: tr; align-self: end; justify-self: end; }
.clabel-bl { grid-area: bl; align-self: start; justify-self: start; }
.clabel-br { grid-area: br; align-self: start; justify-self: end; }

.override-dot {
  color: #f59e0b;
  margin-left: 2px;
  font-weight: bold;
}

.controls {
  width: 260px;
  min-width: 260px;
  padding: 16px;
  background: #111118;
  border-left: 1px solid #222;
  display: flex;
  flex-direction: column;
  gap: 14px;
  overflow-y: auto;
}

.mode-toggle {
  flex: none;
  background: rgba(255, 255, 255, 0.06);
  color: #aaa;
  border: 1px solid #333;
  padding: 8px 12px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 13px;
  font-weight: 600;
  transition: background 0.15s, color 0.15s;
  text-align: center;
}

.mode-toggle.active {
  background: rgba(0, 229, 255, 0.12);
  color: #00e5ff;
  border-color: rgba(0, 229, 255, 0.4);
}

.mode-toggle:hover {
  background: rgba(0, 229, 255, 0.1);
  color: #00e5ff;
}

.controls h2 {
  font-size: 16px;
  color: #fff;
  letter-spacing: 0.05em;
}

.controls h3 {
  font-size: 13px;
  color: #aaa;
  font-weight: 600;
  letter-spacing: 0.04em;
  text-transform: uppercase;
}

.divider {
  border-top: 1px solid #222;
  margin: 2px 0;
}

label {
  display: flex;
  flex-direction: column;
  gap: 5px;
  font-size: 13px;
  color: #999;
}

label strong {
  color: #eee;
}

input[type="range"] {
  width: 100%;
  accent-color: #00e5ff;
}

input[type="color"] {
  width: 48px;
  height: 32px;
  border: none;
  background: none;
  cursor: pointer;
  padding: 0;
  border-radius: 4px;
}

input[type="number"] {
  background: #1a1a24;
  color: #ccc;
  border: 1px solid #333;
  padding: 5px 8px;
  border-radius: 4px;
  font-size: 13px;
  width: 80px;
}

select {
  background: #1a1a24;
  color: #ccc;
  border: 1px solid #333;
  padding: 5px 8px;
  border-radius: 4px;
  font-size: 13px;
}

textarea {
  background: #1a1a24;
  color: #ccc;
  border: 1px solid #333;
  padding: 7px 8px;
  border-radius: 4px;
  font-size: 11px;
  font-family: 'Menlo', 'Consolas', monospace;
  resize: vertical;
  line-height: 1.5;
}

textarea:focus {
  outline: none;
  border-color: #00e5ff44;
}

.override-status {
  font-size: 11px;
  padding: 4px 6px;
  border-radius: 4px;
  line-height: 1.4;
}
.override-status.ok {
  color: #4ade80;
  background: rgba(74, 222, 128, 0.08);
}
.override-status.err {
  color: #f87171;
  background: rgba(248, 113, 113, 0.08);
}

.button-row {
  display: flex;
  gap: 8px;
}

.export-px {
  font-size: 11px;
  color: #666;
}

.label-row {
  flex-direction: row;
  align-items: center;
  gap: 7px;
  color: #ccc;
}

.invert-hint {
  font-size: 11px;
  color: #555;
}

button {
  flex: 1;
  background: rgba(0, 229, 255, 0.08);
  color: #00e5ff;
  border: 1px solid rgba(0, 229, 255, 0.3);
  padding: 9px 12px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 13px;
  transition: background 0.15s;
}

button:hover {
  background: rgba(0, 229, 255, 0.18);
}

.btn-reset {
  background: rgba(255, 255, 255, 0.05);
  color: #888;
  border-color: #333;
}

.btn-reset:hover {
  background: rgba(255, 255, 255, 0.1);
  color: #aaa;
}

.status {
  font-size: 12px;
  color: #00e5ff;
  opacity: 0.85;
}

.progress-count {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 11px;
  color: #00e5ff;
  opacity: 0.7;
}

.btn-stop {
  flex: none;
  padding: 2px 7px;
  font-size: 11px;
  background: rgba(255, 80, 80, 0.12);
  color: #f87171;
  border: 1px solid rgba(255, 80, 80, 0.35);
  border-radius: 4px;
}

.btn-stop:hover {
  background: rgba(255, 80, 80, 0.25);
}
</style>
