<template>
  <div class="canvas" ref="canvasRef"
    @click="handleBgClick"
    @touchstart.passive="onTouchStart"
    @touchend="onTouchEnd"
    @touchcancel="onTouchEnd"
    @touchmove.passive="onTouchMove"
  >
    <!-- Eye tracking debug indicator -->
    <div v-if="showDebug" class="eye-debug">
      <div class="eye-dot" :style="gazeDotStyle"></div>
      <div class="eye-info">
        {{ isTracking ? `👁 tracking (${confidence.toFixed(2)})` : '👁 no face' }}
        <br>head: {{ headX.toFixed(2) }}, {{ headY.toFixed(2) }}
        <br>gaze: {{ gazeX.toFixed(2) }}, {{ gazeY.toFixed(2) }}
      </div>
    </div>

    <div class="canvas-space" ref="spaceRef">
      <BlockCard
        v-for="[id, card] in cards"
        :key="id"
        :card="card"
        :class="{
          'card-dimmed': expandedId && expandedId !== id,
          'card-zoomed': expandedId === id
        }"
        :style="getCardParallaxStyle(card)"
        @toggle-select="(e) => toggleSelect(id, e)"
        @update-position="(x, y) => updateCardPosition(id, x, y)"
        @dblclick="() => handleDblClick(id)"
        @touchend="(e) => handleCardTap(id, e)"
      />
    </div>

    <!-- Gaze highlight — subtle glow where user is looking -->
    <div v-if="isTracking" class="gaze-glow" :style="gazeGlowStyle"></div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { storeToRefs } from 'pinia'
import { useCanvasStore } from '../stores/canvas.js'
import { useEyeTracking } from '../composables/useEyeTracking.js'
import BlockCard from './BlockCard.vue'

const props = defineProps({ expandedCardId: { type: String, default: null } })
const emit = defineEmits(['click-canvas', 'longpress-canvas', 'longpress-end', 'expand-card', 'collapse-card'])

const canvas = useCanvasStore()
const { cards, greetingVisible } = storeToRefs(canvas)
const { toggleSelect, clearSelection, updateCardPosition } = canvas

const spaceRef = ref(null)
const canvasRef = ref(null)
const expandedId = computed(() => props.expandedCardId)
const showDebug = ref(true) // Toggle with 'd' key

// Eye tracking
const { headX, headY, gazeX, gazeY, isTracking, confidence } = useEyeTracking({
  smoothing: 0.12,
  updateRate: 30,
})

// Parallax: each card shifts based on headX/headY * its z-depth
// Higher z = more parallax = feels closer to viewer
const PARALLAX_FACTOR = 25 // max px shift for z=50 cards

function getCardParallaxStyle(card) {
  const z = card.z || 0
  const depth = z / 50 // normalize z to 0-1 range
  const shiftX = headX.value * depth * PARALLAX_FACTOR
  const shiftY = headY.value * depth * PARALLAX_FACTOR * 0.6

  const style = {
    transform: `translate3d(${shiftX}px, ${shiftY}px, 0)`,
    transition: 'transform 0.05s linear',
  }

  if (expandedId.value === card.id) {
    style.transform = `scale(1.25) translate3d(${shiftX * 0.3}px, ${shiftY * 0.3}px, 0)`
    style.zIndex = 9999
    style.filter = 'none'
    style.opacity = 1
    style.width = '90%'
    style.transition = 'transform 0.4s cubic-bezier(.16,1,.3,1), width 0.4s cubic-bezier(.16,1,.3,1), opacity 0.3s, filter 0.3s'
  }

  return style
}

// Gaze glow — subtle light where user looks
const gazeGlowStyle = computed(() => ({
  left: `${gazeX.value * 100}%`,
  top: `${gazeY.value * 100}%`,
  opacity: isTracking.value ? 0.15 : 0,
}))

// Debug gaze dot
const gazeDotStyle = computed(() => ({
  left: `${gazeX.value * 100}%`,
  top: `${gazeY.value * 100}%`,
}))

// Long press detection
let longPressTimer = null
let longPressActive = false
let touchStartPos = null

function onTouchStart(e) {
  if (e.target.closest('.v-block')) return
  touchStartPos = { x: e.touches[0].clientX, y: e.touches[0].clientY }
  longPressTimer = setTimeout(() => {
    longPressActive = true
    emit('longpress-canvas')
  }, 400)
}

function onTouchMove(e) {
  if (!touchStartPos) return
  const dx = e.touches[0].clientX - touchStartPos.x
  const dy = e.touches[0].clientY - touchStartPos.y
  if (Math.abs(dx) > 10 || Math.abs(dy) > 10) {
    clearTimeout(longPressTimer)
    longPressTimer = null
  }
}

function onTouchEnd() {
  if (longPressTimer) { clearTimeout(longPressTimer); longPressTimer = null }
  if (longPressActive) {
    longPressActive = false
    emit('longpress-end')
  }
  touchStartPos = null
}

function handleBgClick(e) {
  if (e.target.closest('.v-block')) return
  if (expandedId.value) {
    emit('collapse-card')
    return
  }
  clearSelection()
  emit('click-canvas')
}

function handleDblClick(id) {
  if (expandedId.value === id) {
    emit('collapse-card')
  } else {
    emit('expand-card', id)
  }
}

let lastTapTime = 0
let lastTapId = null
function handleCardTap(id, e) {
  const now = Date.now()
  if (lastTapId === id && now - lastTapTime < 350) {
    e.preventDefault()
    e.stopPropagation()
    lastTapTime = 0
    lastTapId = null
    if (expandedId.value === id) {
      emit('collapse-card')
    } else {
      emit('expand-card', id)
    }
  } else {
    lastTapTime = now
    lastTapId = id
  }
}

// Debug toggle
function onKeyDown(e) {
  if (e.key === 'd' && !e.target.matches('input, textarea')) {
    showDebug.value = !showDebug.value
  }
}

onMounted(() => {
  window.addEventListener('keydown', onKeyDown)
})
onUnmounted(() => {
  window.removeEventListener('keydown', onKeyDown)
})
</script>

<style scoped>
.eye-debug {
  position: fixed;
  top: 0; left: 0; right: 0; bottom: 0;
  pointer-events: none;
  z-index: 10000;
}
.eye-dot {
  position: absolute;
  width: 12px; height: 12px;
  border-radius: 50%;
  background: rgba(0, 200, 255, 0.8);
  box-shadow: 0 0 12px rgba(0, 200, 255, 0.6);
  transform: translate(-50%, -50%);
  transition: left 0.05s, top 0.05s;
}
.eye-info {
  position: absolute;
  top: 8px; right: 8px;
  font-size: 10px;
  color: rgba(255,255,255,0.5);
  font-family: monospace;
  text-align: right;
  line-height: 1.6;
}
.gaze-glow {
  position: fixed;
  width: 200px; height: 200px;
  border-radius: 50%;
  background: radial-gradient(circle, rgba(100,150,255,0.2) 0%, transparent 70%);
  transform: translate(-50%, -50%);
  pointer-events: none;
  z-index: 1;
  transition: left 0.1s, top 0.1s, opacity 0.5s;
}
</style>
