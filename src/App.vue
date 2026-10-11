<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue'

const OUTPUT = { width: 1092, height: 1456 }
const WINDOW = [
  { x: 238, y: 322 },
  { x: 879, y: 323 },
  { x: 857, y: 960 },
  { x: 210, y: 960 },
]
const BOX = { x: 210, y: 322, width: 669, height: 638 }

const canvas = ref(null)
const fileInput = ref(null)
const zoom = ref(1)
const photo = ref(null)
const frame = new Image()
const frameReady = ref(false)
const dragging = ref(false)
const dragStart = { x: 0, y: 0, panX: 0, panY: 0 }
const pan = { x: 0, y: 0 }
let activePointer = null
let objectUrl = null

const hasPhoto = computed(() => Boolean(photo.value))

function windowPath(ctx) {
  ctx.moveTo(WINDOW[0].x, WINDOW[0].y)
  WINDOW.slice(1).forEach(({ x, y }) => ctx.lineTo(x, y))
  ctx.closePath()
}

function coverScale(img) {
  return Math.max(BOX.width / img.naturalWidth, BOX.height / img.naturalHeight)
}

function clampPan() {
  if (!photo.value) return
  const scale = coverScale(photo.value) * zoom.value
  const excessX = Math.max(0, (photo.value.naturalWidth * scale - BOX.width) / 2)
  const excessY = Math.max(0, (photo.value.naturalHeight * scale - BOX.height) / 2)
  pan.x = Math.max(-excessX, Math.min(excessX, pan.x))
  pan.y = Math.max(-excessY, Math.min(excessY, pan.y))
}

function drawPhoto(ctx) {
  if (!photo.value) return
  const img = photo.value
  const scale = coverScale(img) * zoom.value
  const width = img.naturalWidth * scale
  const height = img.naturalHeight * scale
  const centerX = BOX.x + BOX.width / 2 + pan.x
  const centerY = BOX.y + BOX.height / 2 + pan.y

  ctx.save()
  ctx.beginPath()
  windowPath(ctx)
  ctx.clip()
  ctx.drawImage(img, centerX - width / 2, centerY - height / 2, width, height)
  ctx.restore()
}

function drawFrameWithoutWindow(ctx) {
  ctx.save()
  ctx.beginPath()
  ctx.rect(0, 0, OUTPUT.width, OUTPUT.height)
  windowPath(ctx)
  ctx.clip('evenodd')
  ctx.drawImage(frame, 0, 0, OUTPUT.width, OUTPUT.height)
  ctx.restore()
}

function drawInfoCard(ctx) {
  ctx.save()
  ctx.translate(970, 1080)
  ctx.rotate(-0.025)
  ctx.shadowColor = 'rgba(65, 31, 13, .28)'
  ctx.shadowBlur = 10
  ctx.fillStyle = 'rgba(239, 206, 153, .96)'
  ctx.beginPath()
  ctx.roundRect(-111, -112, 220, 330, 6)
  ctx.fill()
  ctx.shadowColor = 'transparent'

  ctx.fillStyle = '#5d2d18'
  ctx.textAlign = 'center'
  ctx.font = '700 22px "Microsoft JhengHei", sans-serif'
  ctx.fillText('料理鼠亡', 0, -76)
  ctx.font = '600 15px "Microsoft JhengHei", sans-serif'
  ctx.fillText('2026/9/17～2026/10/08', 0, -47)

  ctx.strokeStyle = 'rgba(93, 45, 24, .45)'
  ctx.lineWidth = 1.5
  ctx.beginPath()
  ctx.moveTo(-76, -27)
  ctx.lineTo(76, -27)
  ctx.stroke()

  ctx.textAlign = 'left'
  ctx.font = '600 16px "Microsoft JhengHei", sans-serif'
  const names = ['Janice 尖尖', 'Cathy 阿栗栗', '🐳蔡澤', '布奇', 'Sophia LUO']
  names.forEach((name, index) => ctx.fillText(name, -79, 8 + index * 36))
  ctx.restore()
}

function draw() {
  const el = canvas.value
  if (!el || !frameReady.value) return
  const ctx = el.getContext('2d')
  ctx.clearRect(0, 0, OUTPUT.width, OUTPUT.height)

  ctx.fillStyle = '#e9d8bd'
  ctx.fillRect(0, 0, OUTPUT.width, OUTPUT.height)
  drawPhoto(ctx)
  drawFrameWithoutWindow(ctx)
  drawInfoCard(ctx)

  if (!photo.value) {
    ctx.save()
    ctx.beginPath()
    windowPath(ctx)
    ctx.clip()
    ctx.fillStyle = 'rgba(255, 249, 235, .5)'
    ctx.fillRect(BOX.x, BOX.y, BOX.width, BOX.height)
    ctx.fillStyle = '#77452c'
    ctx.textAlign = 'center'
    ctx.font = '700 30px "Microsoft JhengHei", sans-serif'
    ctx.fillText('上傳今天的料理照片', BOX.x + BOX.width / 2, BOX.y + BOX.height / 2 - 8)
    ctx.font = '500 19px "Microsoft JhengHei", sans-serif'
    ctx.fillText('上傳後可拖曳與縮放', BOX.x + BOX.width / 2, BOX.y + BOX.height / 2 + 34)
    ctx.restore()
  }
}

async function loadPhoto(event) {
  const file = event.target.files?.[0]
  if (!file) return
  if (!file.type.startsWith('image/')) return
  if (objectUrl) URL.revokeObjectURL(objectUrl)
  objectUrl = URL.createObjectURL(file)
  const img = new Image()
  img.onload = () => {
    photo.value = img
    zoom.value = 1
    pan.x = 0
    pan.y = 0
    draw()
  }
  img.src = objectUrl
}

function pickPhoto() {
  fileInput.value?.click()
}

function canvasPoint(event) {
  const rect = canvas.value.getBoundingClientRect()
  return {
    x: (event.clientX - rect.left) * OUTPUT.width / rect.width,
    y: (event.clientY - rect.top) * OUTPUT.height / rect.height,
  }
}

function inWindow(point) {
  return point.x >= BOX.x && point.x <= BOX.x + BOX.width && point.y >= BOX.y && point.y <= BOX.y + BOX.height
}

function pointerDown(event) {
  if (!photo.value || !inWindow(canvasPoint(event))) return
  activePointer = event.pointerId
  canvas.value.setPointerCapture(event.pointerId)
  const point = canvasPoint(event)
  dragStart.x = point.x
  dragStart.y = point.y
  dragStart.panX = pan.x
  dragStart.panY = pan.y
  dragging.value = true
}

function pointerMove(event) {
  if (!dragging.value || event.pointerId !== activePointer) return
  const point = canvasPoint(event)
  pan.x = dragStart.panX + point.x - dragStart.x
  pan.y = dragStart.panY + point.y - dragStart.y
  clampPan()
  draw()
}

function pointerUp(event) {
  if (event.pointerId !== activePointer) return
  dragging.value = false
  activePointer = null
}

function wheelZoom(event) {
  if (!photo.value || !inWindow(canvasPoint(event))) return
  event.preventDefault()
  zoom.value = Math.max(1, Math.min(3, zoom.value + (event.deltaY < 0 ? 0.08 : -0.08)))
}

function resetPosition() {
  zoom.value = 1
  pan.x = 0
  pan.y = 0
  draw()
}

function download() {
  draw()
  canvas.value.toBlob((blob) => {
    if (!blob) return
    const url = URL.createObjectURL(blob)
    const link = document.createElement('a')
    link.download = `料理鼠亡-${new Date().toISOString().slice(0, 10)}.png`
    link.href = url
    link.click()
    window.setTimeout(() => URL.revokeObjectURL(url), 1000)
  }, 'image/png')
}

watch(zoom, () => {
  clampPan()
  draw()
})

onMounted(async () => {
  frame.onload = () => {
    frameReady.value = true
    draw()
  }
  frame.src = `${import.meta.env.BASE_URL}frame.png`
  await nextTick()
  canvas.value?.addEventListener('wheel', wheelZoom, { passive: false })
})

onBeforeUnmount(() => {
  canvas.value?.removeEventListener('wheel', wheelZoom)
  if (objectUrl) URL.revokeObjectURL(objectUrl)
})
</script>

<template>
  <main class="page-shell">
    <section class="intro">
      <p class="eyebrow">COOK · LEARN · SHARE</p>
      <h1>料理鼠亡<br><span>料理拍立得</span></h1>
      <p class="lead">把今天的料理，裝進我們的共學回憶。</p>

      <div class="steps" aria-label="使用步驟">
        <div><b>1</b><span>上傳料理照片</span></div>
        <div><b>2</b><span>拖曳、縮放取景</span></div>
        <div><b>3</b><span>下載完整 PNG</span></div>
      </div>

      <div class="controls">
        <input ref="fileInput" class="visually-hidden" type="file" accept="image/*" @change="loadPhoto">
        <button class="button primary" type="button" @click="pickPhoto">
          {{ hasPhoto ? '換一張照片' : '上傳料理照片' }}
        </button>

        <div class="zoom-control" :class="{ disabled: !hasPhoto }">
          <label for="zoom">照片縮放 <output>{{ Math.round(zoom * 100) }}%</output></label>
          <input id="zoom" v-model.number="zoom" type="range" min="1" max="3" step="0.01" :disabled="!hasPhoto">
        </div>

        <div class="secondary-actions">
          <button class="button ghost" type="button" :disabled="!hasPhoto" @click="resetPosition">重新置中</button>
          <button class="button download" type="button" :disabled="!hasPhoto" @click="download">下載 PNG</button>
        </div>
      </div>

      <p class="privacy">照片只在你的瀏覽器裡處理，不會上傳到伺服器。</p>
    </section>

    <section class="stage" aria-label="拍立得即時預覽">
      <div class="preview-heading">
        <span>即時預覽</span>
        <small>直式 · 1092 × 1456</small>
      </div>
      <div class="canvas-wrap" :class="{ dragging }">
        <canvas
          ref="canvas"
          :width="OUTPUT.width"
          :height="OUTPUT.height"
          aria-label="料理鼠亡拍立得預覽；上傳照片後可在中央拖曳"
          @pointerdown="pointerDown"
          @pointermove="pointerMove"
          @pointerup="pointerUp"
          @pointercancel="pointerUp"
        />
      </div>
      <p class="hint">在中央照片上拖曳調整位置，也可用滑鼠滾輪縮放。</p>
    </section>
  </main>
</template>
