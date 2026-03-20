<template>
  <div class="canvas" ref="canvasRef"
    @click="handleBgClick"
    @touchstart.passive="onTouchStart"
    @touchend="onTouchEnd"
    @touchcancel="onTouchEnd"
    @touchmove.passive="onTouchMove"
  >
    <div class="canvas-space" ref="spaceRef" :style="{ minHeight: canvasHeight + 'px' }">
      <BlockCard
        v-for="[id, card] in cards"
        :key="id"
        :card="card"
        :class="{
          'card-dimmed': expandedId && expandedId !== id,
          'card-zoomed': expandedId === id
        }"
        :style="expandedId === id ? zoomedStyle : {}"
        @toggle-select="(e) => toggleSelect(id, e)"
        @update-position="(x, y) => updateCardPosition(id, x, y)"
        @dblclick="() => handleDblClick(id)"
        @touchend="(e) => handleCardTap(id, e)"
      />
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { storeToRefs } from 'pinia'
import { useCanvasStore } from '../stores/canvas.js'
import BlockCard from './BlockCard.vue'

const props = defineProps({ expandedCardId: { type: String, default: null } })
const emit = defineEmits(['click-canvas', 'longpress-canvas', 'longpress-end', 'expand-card', 'collapse-card'])

const canvas = useCanvasStore()
const { cards, greetingVisible } = storeToRefs(canvas)
const { toggleSelect, clearSelection, updateCardPosition } = canvas

const spaceRef = ref(null)
const canvasRef = ref(null)
const expandedId = computed(() => props.expandedCardId)

const canvasHeight = computed(() => {
  let maxBottom = window.innerHeight
  for (const [, card] of cards.value) {
    const bottom = (card.y || 0) + (card.height || 200) + 40
    if (bottom > maxBottom) maxBottom = bottom
  }
  return maxBottom
})

// Zoomed card style — scale up in place, highest z-index
const zoomedStyle = computed(() => ({
  transform: 'scale(1.35) translateZ(200px)',
  zIndex: 9999,
  filter: 'none',
  opacity: 1,
  width: '85%',
  transition: 'transform 0.4s cubic-bezier(.16,1,.3,1), z-index 0s, width 0.4s cubic-bezier(.16,1,.3,1), opacity 0.3s, filter 0.3s',
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
  // If a card is expanded, collapse it
  if (expandedId.value) {
    emit('collapse-card')
    return
  }
  clearSelection()
  emit('click-canvas')
}

// Double-click to expand (desktop)
function handleDblClick(id) {
  if (expandedId.value === id) {
    emit('collapse-card')
  } else {
    emit('expand-card', id)
  }
}

// Touch double-tap detection (mobile)
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

onMounted(() => {})
onUnmounted(() => {})
</script>
