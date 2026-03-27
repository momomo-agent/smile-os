<template>
  <div class="canvas" ref="canvasRef"
    @click="handleBgClick"
    @touchstart.passive="onTouchStart"
    @touchend="onTouchEnd"
    @touchcancel="onTouchEnd"
    @touchmove.passive="onTouchMove"
  >
    <div class="canvas-space" ref="spaceRef">
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
      <div v-if="cards.size === 0" class="greeting" :class="{ hidden: !greetingVisible }">smile os</div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { storeToRefs } from 'pinia'
import { useCanvasStore } from '../stores/canvas.js'
import BlockCard from './BlockCard.vue'

const props = defineProps({ expandedCardId: { type: String, default: null } })
const emit = defineEmits(['click-canvas', 'longpress-canvas', 'longpress-end', 'expand-card', 'collapse-card', 'navigate'])

const canvas = useCanvasStore()
const { cards, greetingVisible } = storeToRefs(canvas)
const { toggleSelect, clearSelection, updateCardPosition } = canvas

const spaceRef = ref(null)
const canvasRef = ref(null)
const expandedId = computed(() => props.expandedCardId)

// Zoomed card style
const zoomedStyle = computed(() => ({
  transform: 'scale(1.25)',
  zIndex: 9999,
  filter: 'none',
  opacity: 1,
  width: '90%',
  transition: 'transform 0.4s cubic-bezier(.16,1,.3,1), z-index 0s, width 0.4s cubic-bezier(.16,1,.3,1), opacity 0.3s, filter 0.3s',
}))

// Touch handling: long press for voice, horizontal swipe for timeline navigation
let longPressTimer = null
let longPressActive = false
let touchStartPos = null
let touchStartTime = 0
let isHorizontalSwipe = false

function onTouchStart(e) {
  touchStartPos = { x: e.touches[0].clientX, y: e.touches[0].clientY }
  touchStartTime = Date.now()
  isHorizontalSwipe = false

  // Only start long press if not on a card
  if (!e.target.closest('.v-block')) {
    longPressTimer = setTimeout(() => {
      longPressActive = true
      emit('longpress-canvas')
    }, 400)
  }
}

function onTouchMove(e) {
  if (!touchStartPos) return
  const dx = e.touches[0].clientX - touchStartPos.x
  const dy = e.touches[0].clientY - touchStartPos.y

  // Cancel long press on any movement
  if (Math.abs(dx) > 10 || Math.abs(dy) > 10) {
    clearTimeout(longPressTimer)
    longPressTimer = null
  }

  // Detect horizontal swipe (for timeline navigation)
  if (Math.abs(dx) > Math.abs(dy) * 1.5 && Math.abs(dx) > 30) {
    isHorizontalSwipe = true
  }
}

function onTouchEnd(e) {
  if (longPressTimer) { clearTimeout(longPressTimer); longPressTimer = null }
  if (longPressActive) {
    longPressActive = false
    emit('longpress-end')
    touchStartPos = null
    return
  }

  // Check for horizontal swipe → timeline navigation
  if (touchStartPos && isHorizontalSwipe) {
    const dx = (e.changedTouches?.[0]?.clientX || 0) - touchStartPos.x
    const elapsed = Date.now() - touchStartTime
    if (Math.abs(dx) > 60 && elapsed < 500) {
      // Swipe left = go forward (down in timeline), swipe right = go back (up)
      emit('navigate', dx > 0 ? 'up' : 'down')
    }
  }

  touchStartPos = null
  isHorizontalSwipe = false
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

// Double-click/tap to expand
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

onMounted(() => {})
onUnmounted(() => {})
</script>
