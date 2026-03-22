<template>
  <!-- Single face element that animates between positions -->
  <div class="face-icon" :class="{ 'face-idle': idle, 'face-active': !idle }"
    @click="idle ? wakeUp() : null"
    @touchstart.passive="idle ? wakeUp() : null"
    :style="faceParallaxStyle"
  >
    <svg viewBox="0 0 48 48" fill="none" xmlns="http://www.w3.org/2000/svg">
      <circle cx="24" cy="24" r="20" stroke="rgba(255,255,255,0.3)" stroke-width="1.5"/>
      <!-- Eyes follow the user's head position -->
      <circle :cx="18 + eyeOffsetX" :cy="20 + eyeOffsetY" r="1.5" fill="rgba(255,255,255,0.4)"/>
      <circle :cx="30 + eyeOffsetX" :cy="20 + eyeOffsetY" r="1.5" fill="rgba(255,255,255,0.4)"/>
      <path :d="`M${18 + eyeOffsetX * 0.5} 29c2 2.5 4 3.5 6 3.5s4-1 6-3.5`" stroke="rgba(255,255,255,0.3)" stroke-width="1.5" stroke-linecap="round"/>
    </svg>
  </div>

  <!-- Idle overlay (text only, face is separate) -->
  <div v-if="idle" class="idle-screen" @click="wakeUp" @touchstart.passive="wakeUp">
    <div class="idle-bottom">
      <div class="idle-time">{{ currentTime }}</div>
      <div class="idle-greeting">Nice to meet you</div>
    </div>
  </div>

  <!-- Active state -->
  <template v-if="!idle">
    <!-- Top bubble (below face) -->
    <div class="top-bubble-wrap">
      <div class="top-bubble" :class="{ visible: bubbleVisible || isScrollingTimeline }">
        {{ isScrollingTimeline ? timelineBubbleText : bubbleText }}
      </div>
    </div>

    <CanvasSpace
      @click-canvas="handleCanvasClick"
      @longpress-canvas="startRecording"
      @longpress-end="stopRecording"
      :expanded-card-id="expandedCardId"
      @expand-card="expandCard"
      @collapse-card="collapseCard"
    />
    <ThinkingDots :visible="isThinking" />

    <!-- Bottom floating bar -->
    <div class="bottom-bar" :class="{ 'input-open': inputOpen }">
      <div v-if="!inputOpen" class="bottom-hint"
        @click="inputOpen = true"
        @touchstart.passive="onBottomTouchStart"
        @touchend="onBottomTouchEnd"
        @touchcancel="onBottomTouchEnd"
      >
        <div class="bottom-line"></div>
        <span v-if="sttRecording" class="bottom-stt-text">{{ sttText || '说话中...' }}</span>
      </div>
      <div v-else class="bottom-input-wrap">
        <input
          ref="bottomInput"
          v-model="inputText"
          placeholder="说点什么..."
          @keydown.enter="sendText"
          @blur="maybeCloseInput"
        />
        <button class="send-btn" @click="sendText" :disabled="!inputText.trim()">↑</button>
      </div>
    </div>

    <!-- Recording pulse overlay -->
    <div class="recording-pulse" :class="{ active: sttRecording }"></div>
    <div v-if="sttRecording && sttText" class="stt-live-text">{{ sttText }}</div>
  </template>

  <button class="gear-btn" @click="configOpen = true">⚙</button>
  <ConfigPanel v-model:open="configOpen" />
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, watch, nextTick } from 'vue'
import CanvasSpace from './components/CanvasSpace.vue'
import ThinkingDots from './components/ThinkingDots.vue'
import ConfigPanel from './components/ConfigPanel.vue'
import { useSend } from './composables/useSend.js'
import { useTTS } from './composables/useTTS.js'
import { useSTT } from './composables/useSTT.js'
import { useTimeline } from './composables/useTimeline.js'
import { useEyeTracking } from './composables/useEyeTracking.js'
import { useConfigStore } from './stores/config.js'
import { useTimelineStore } from './stores/timeline.js'

const idle = ref(true)
const configOpen = ref(false)
const inputOpen = ref(false)
const inputText = ref('')
const bottomInput = ref(null)
const expandedCardId = ref(null)
const sttText = ref('')
const configStore = useConfigStore()
const timeline = useTimelineStore()
let lastInputWasVoice = false
let bottomLongPressTimer = null

// Eye tracking — face follows user, parallax on idle
const { headX, headY } = useEyeTracking({ smoothing: 0.12, updateRate: 30 })

// Face eyes follow user's head
const eyeOffsetX = computed(() => headX.value * 2.5)
const eyeOffsetY = computed(() => headY.value * 1.5)

// Idle face parallax — slight drift with head
const faceParallaxStyle = computed(() => {
  if (!idle.value) return {}
  return {
    transform: `translate(calc(-50% + ${headX.value * 8}px), calc(-50% + ${headY.value * 5}px))`,
  }
})

// Current time for idle screen
const currentTime = ref('')
function updateTime() {
  const now = new Date()
  currentTime.value = now.getHours().toString().padStart(2, '0') + ' : ' + now.getMinutes().toString().padStart(2, '0')
}
updateTime()
let timeInterval = null

// Wake up from idle
function wakeUp() {
  idle.value = false
}

// TTS
const tts = useTTS()

// Send
const { send, isThinking, bubbleText, bubbleVisible, toolLogs, showBubble, dismissBubble } = useSend({ tts })

tts.onPlaybackEnd(() => { dismissBubble(3000) })

async function handleSend(text) {
  if (!configStore.apiKey) { configOpen.value = true; return }
  tts.unlockAudio()
  send(text)
}

// STT
const { isRecording: sttRecording, startRecording: rawStartRecording, stopRecording: rawStopRecording } = useSTT({
  tts,
  onResult: (text) => {
    lastInputWasVoice = true
    sttText.value = ''
    handleSend(text)
  },
  onError: (msg) => { showBubble(msg, 3000) },
  onStart: (label) => {
    dismissBubble(0)
    showBubble(label || '说话中...')
  },
  onStop: () => { bubbleVisible.value = false },
  onThinkingStart: () => { isThinking.value = true },
  onThinkingEnd: () => { isThinking.value = false },
  onPartialResult: (text) => { sttText.value = text },
})

function startRecording() {
  bubbleVisible.value = false
  rawStartRecording()
}
function stopRecording() {
  bubbleVisible.value = false
  rawStopRecording()
}

// Bottom bar long press
function onBottomTouchStart() {
  bottomLongPressTimer = setTimeout(() => { startRecording() }, 300)
}
function onBottomTouchEnd() {
  if (bottomLongPressTimer) { clearTimeout(bottomLongPressTimer); bottomLongPressTimer = null }
  if (sttRecording.value) stopRecording()
}

// Canvas click
function handleCanvasClick() {
  if (inputOpen.value) { inputOpen.value = false }
}

// Send text from input
function sendText() {
  const t = inputText.value.trim()
  if (!t) return
  inputText.value = ''
  inputOpen.value = false
  handleSend(t)
}

function maybeCloseInput() {
  setTimeout(() => { if (!inputText.value.trim()) inputOpen.value = false }, 200)
}

// Expand/collapse cards
function expandCard(id) { expandedCardId.value = id }
function collapseCard() { expandedCardId.value = null }

// Timeline
const { isScrollingTimeline } = useTimeline()
const timelineBubbleText = computed(() => {
  if (!isScrollingTimeline.value) return ''
  const info = timeline.getBubbleInfo()
  return info ? info.text : ''
})

// TTS → bubble
watch(bubbleText, async (text) => {
  if (text && !isScrollingTimeline.value && !sttRecording.value) {
    const played = await tts.playTTS(text)
    if (!played) dismissBubble(3000)
  }
})

// Focus input when opened
watch(inputOpen, (open) => {
  if (open) nextTick(() => bottomInput.value?.focus())
})

// Spacebar push-to-talk
const spaceDown = ref(false)
function handleKeyDown(e) {
  if (e.key !== ' ' || e.repeat) return
  if (document.activeElement?.matches('input, textarea, select')) return
  if (configOpen.value || idle.value) return
  e.preventDefault()
  spaceDown.value = true
  startRecording()
}
function handleKeyUp(e) {
  if (e.key !== ' ' || !spaceDown.value) return
  e.preventDefault()
  spaceDown.value = false
  stopRecording()
}

onMounted(() => {
  timeInterval = setInterval(updateTime, 10000)
  window.addEventListener('keydown', handleKeyDown)
  window.addEventListener('keyup', handleKeyUp)
})
onUnmounted(() => {
  clearInterval(timeInterval)
  window.removeEventListener('keydown', handleKeyDown)
  window.removeEventListener('keyup', handleKeyUp)
})
</script>
