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
          'card-faded': expandedId && expandedId !== id,
          'card-expanded': expandedId === id
        }"
        @toggle-select="(e) => toggleSelect(id, e)"
        @update-position="(x, y) => updateCardPosition(id, x, y)"
        @dblclick="() => handleDblClick(id)"
      />
    </div>

    <!-- Expanded overlay -->
    <div v-if="expandedId" class="expanded-overlay"
      @touchstart.passive="onExpandTouchStart"
      @touchmove.passive="onExpandTouchMove"
      @touchend="onExpandTouchEnd"
    >
      <div class="expanded-card" :style="expandStyle">
        <BlockCard
          v-for="[id, card] in cards"
          :key="'exp-' + id"
          v-show="id === expandedId"
          :card="card"
          style="position:relative;width:100%;max-width:100%;transform:none;opacity:1;filter:none;"
        />
      </div>
      <div class="swipe-hint" :style="{ opacity: swipeFraction }">↑ 上滑关闭</div>
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

// Canvas height — enough to scroll through all cards
const canvasHeight = computed(() => {
  let maxBottom = window.innerHeight
  for (const [, card] of cards.value) {
    const bottom = (card.y || 0) + (card.height || 200) + 40
    if (bottom > maxBottom) maxBottom = bottom
  }
  return maxBottom
})

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
  if (e.target.closest('.v-block') || e.target.closest('.input-bar')) return
  clearSelection()
  emit('click-canvas')
}

// Double-click to expand
function handleDblClick(id) {
  emit('expand-card', id)
}

// Expanded card swipe-to-close
const swipeY = ref(0)
const swipeFraction = computed(() => Math.min(swipeY.value / 200, 1))
const expandStyle = computed(() => {
  if (swipeY.value <= 0) return {}
  const scale = 1 - swipeY.value / 1000
  const ty = -swipeY.value * 0.5
  return {
    transform: `scale(${Math.max(scale, 0.7)}) translateY(${ty}px)`,
    borderRadius: `${Math.min(swipeY.value / 5, 20)}px`
  }
})

let expandTouchStartY = 0
function onExpandTouchStart(e) {
  expandTouchStartY = e.touches[0].clientY
  swipeY.value = 0
}
function onExpandTouchMove(e) {
  const dy = expandTouchStartY - e.touches[0].clientY
  swipeY.value = Math.max(dy, 0)
}
function onExpandTouchEnd() {
  if (swipeY.value > 100) {
    emit('collapse-card')
  }
  swipeY.value = 0
}

// No desktop parallax — mobile-first

onMounted(() => {})
onUnmounted(() => {})
</script>
