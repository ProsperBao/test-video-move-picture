<script setup lang="ts">
defineOptions({
  name: 'VideoPlayer',
})

interface VideoState {
  isHovering: boolean
  isAutoPlaying: boolean
  isSeeking: boolean
  rafId: number | null
  mouseX: number
  lastSeekTime: number
  playDirection: 1 | -1
  autoPlayStartTime: number
  autoPlayStartPercentage: number
  autoPlayDuration: number
}

const CONFIG = {
  SEEK_THROTTLE: 10,
  AUTO_PLAY_SEEK_THROTTLE: 10,
  AUTO_PLAY_DURATION: 8000,
  MIN_DIFF: { auto: 0.001, drag: 0.15 },
}

const videoRefs = ref<HTMLVideoElement[]>([])
const wrapperRefs = ref<HTMLElement[]>([])
const states = ref<VideoState[]>([])

function setVideoRef(el: HTMLVideoElement | null, index: number) {
  if (el) {
    videoRefs.value[index] = el
  }
}

function getPercentage(video: HTMLVideoElement) {
  if (!video.duration || Number.isNaN(video.duration))
    return 0
  return video.currentTime / video.duration
}

function startAutoPlay(index: number, direction: 1 | -1) {
  const video = videoRefs.value[index]
  const state = states.value[index]
  if (!video || !state)
    return

  if (state.rafId) {
    cancelAnimationFrame(state.rafId)
    state.rafId = null
  }
  state.isAutoPlaying = true
  state.playDirection = direction
  state.autoPlayStartPercentage = getPercentage(video)
  state.autoPlayStartTime = Date.now()
  const remaining = direction === 1
    ? (1 - state.autoPlayStartPercentage)
    : state.autoPlayStartPercentage
  state.autoPlayDuration = remaining * CONFIG.AUTO_PLAY_DURATION
  state.lastSeekTime = 0
  video.pause()
  updateVideoTime(index)
}

function calculateTargetTime(index: number, clientX: number) {
  const wrapper = wrapperRefs.value[index]
  const video = videoRefs.value[index]
  if (!wrapper || !video)
    return 0

  const rect = wrapper.getBoundingClientRect()
  let p = (clientX - rect.left) / rect.width
  p = p < 0 ? 1 - (Math.abs(p) % 1) : p > 1 ? p % 1 : p
  p = Math.max(0, Math.min(1, p))
  return Math.min(video.duration - 0.01, p * video.duration)
}

function updateVideoTime(index: number) {
  const video = videoRefs.value[index]
  const state = states.value[index]
  if (!video || !state)
    return

  if (!video.duration || Number.isNaN(video.duration)) {
    state.rafId = null
    return
  }

  if (state.isSeeking && !state.isAutoPlaying) {
    state.rafId = requestAnimationFrame(() => updateVideoTime(index))
    return
  }

  let targetTime: number

  if (state.isHovering) {
    targetTime = calculateTargetTime(index, state.mouseX)
  }
  else if (state.isAutoPlaying) {
    const elapsed = Date.now() - state.autoPlayStartTime

    if (elapsed >= state.autoPlayDuration) {
      if (state.playDirection === 1) {
        video.currentTime = video.duration - 0.01
        state.playDirection = -1
        state.autoPlayStartPercentage = 1
      }
      else {
        video.currentTime = 0
        state.playDirection = 1
        state.autoPlayStartPercentage = 0
      }
      state.autoPlayStartTime = Date.now()
      state.autoPlayDuration = CONFIG.AUTO_PLAY_DURATION
      state.rafId = requestAnimationFrame(() => updateVideoTime(index))
      return
    }

    const progress = Math.min(1, elapsed / state.autoPlayDuration)

    if (state.playDirection === 1) {
      const startTime = state.autoPlayStartPercentage * video.duration
      const endTime = video.duration - 0.01
      targetTime = startTime + progress * (endTime - startTime)
      targetTime = Math.min(endTime, targetTime)
    }
    else {
      const startTime = state.autoPlayStartPercentage * video.duration
      const endTime = 0
      targetTime = startTime - progress * (startTime - endTime)
      targetTime = Math.max(endTime, targetTime)
    }

    const p = targetTime / video.duration
    if (state.playDirection === 1 && p >= 0.995) {
      state.playDirection = -1
      state.autoPlayStartPercentage = 1
      state.autoPlayStartTime = Date.now()
      state.autoPlayDuration = CONFIG.AUTO_PLAY_DURATION
      targetTime = video.duration - 0.01
    }
    else if (state.playDirection === -1 && p <= 0.005) {
      state.playDirection = 1
      state.autoPlayStartPercentage = 0
      state.autoPlayStartTime = Date.now()
      state.autoPlayDuration = CONFIG.AUTO_PLAY_DURATION
      targetTime = 0
    }
  }
  else {
    state.rafId = null
    return
  }

  const timeDiff = Math.abs(video.currentTime - targetTime)
  const now = Date.now()
  const throttle = state.isAutoPlaying
    ? CONFIG.AUTO_PLAY_SEEK_THROTTLE
    : CONFIG.SEEK_THROTTLE
  const minDiff = state.isAutoPlaying ? CONFIG.MIN_DIFF.auto : CONFIG.MIN_DIFF.drag

  if (state.isAutoPlaying && !state.isSeeking) {
    if (now - state.lastSeekTime >= throttle) {
      try {
        video.currentTime = targetTime
        state.lastSeekTime = now
      }
      catch (e) {
        console.error('Error setting video time:', e)
      }
    }
  }
  else if (state.isHovering && timeDiff > minDiff && !state.isSeeking) {
    if (now - state.lastSeekTime >= throttle) {
      try {
        video.currentTime = targetTime
        state.lastSeekTime = now
      }
      catch (e) {
        console.error('Error setting video time:', e)
      }
    }
  }

  state.rafId = requestAnimationFrame(() => updateVideoTime(index))
}

const cleanupFunctions: Array<() => void> = []

onMounted(() => {
  // 等待 DOM 更新
  nextTick(() => {
    // 初始化状态
    states.value = Array.from({ length: 2 }, () => ({
      isHovering: false,
      isAutoPlaying: false,
      isSeeking: false,
      rafId: null,
      mouseX: 0,
      lastSeekTime: 0,
      playDirection: 1,
      autoPlayStartTime: 0,
      autoPlayStartPercentage: 0,
      autoPlayDuration: CONFIG.AUTO_PLAY_DURATION,
    }))

    // 为每个视频设置事件监听
    videoRefs.value.forEach((video, index) => {
      const wrapper = wrapperRefs.value[index]
      const state = states.value[index]
      if (!video || !wrapper || !state)
        return

      const getPercentage = () => video.currentTime / video.duration

      const handleLoadedMetadata = () => {
        video.currentTime = 0
        startAutoPlay(index, 1)
      }

      const handleEnded = () => {
        if (!state.isHovering && !state.isAutoPlaying) {
          startAutoPlay(index, -1)
        }
      }

      const handleSeeking = () => {
        state.isSeeking = true
      }

      const handleSeeked = () => {
        state.isSeeking = false
      }

      const handleMouseEnter = () => {
        state.isHovering = true
        state.isAutoPlaying = false
        video.pause()
        if (!state.rafId) {
          state.rafId = requestAnimationFrame(() => updateVideoTime(index))
        }
      }

      const handleMouseLeave = () => {
        state.isHovering = false
        state.isSeeking = false
        if (state.rafId) {
          cancelAnimationFrame(state.rafId)
          state.rafId = null
        }
        state.isAutoPlaying = false
        state.lastSeekTime = 0
        const direction = getPercentage() < 0.5 ? -1 : 1
        startAutoPlay(index, direction)
      }

      const handleMouseMove = (e: MouseEvent) => {
        if (state.isHovering) {
          state.mouseX = e.clientX
          if (!state.rafId) {
            state.rafId = requestAnimationFrame(() => updateVideoTime(index))
          }
        }
      }

      video.addEventListener('loadedmetadata', handleLoadedMetadata)
      video.addEventListener('ended', handleEnded)
      video.addEventListener('seeking', handleSeeking)
      video.addEventListener('seeked', handleSeeked)
      wrapper.addEventListener('mouseenter', handleMouseEnter)
      wrapper.addEventListener('mouseleave', handleMouseLeave)
      wrapper.addEventListener('mousemove', handleMouseMove)

      // 保存清理函数
      cleanupFunctions.push(() => {
        video.removeEventListener('loadedmetadata', handleLoadedMetadata)
        video.removeEventListener('ended', handleEnded)
        video.removeEventListener('seeking', handleSeeking)
        video.removeEventListener('seeked', handleSeeked)
        wrapper.removeEventListener('mouseenter', handleMouseEnter)
        wrapper.removeEventListener('mouseleave', handleMouseLeave)
        wrapper.removeEventListener('mousemove', handleMouseMove)
        if (state.rafId) {
          cancelAnimationFrame(state.rafId)
          state.rafId = null
        }
      })
    })
  })
})

onUnmounted(() => {
  // 清理所有事件监听器和动画帧
  cleanupFunctions.forEach(cleanup => cleanup())
  cleanupFunctions.length = 0
})
</script>

<template>
  <div class="container">
    <div
      v-for="index in 2"
      :key="index"
      ref="wrapperRefs"
      class="video-wrapper"
    >
      <video
        :ref="(el) => setVideoRef(el as HTMLVideoElement, index - 1)"
        :src="`/${index}.mp4`"
        muted
        playsinline
        preload="auto"
      />
    </div>
  </div>
</template>

<style scoped>
.container {
  display: flex;
  gap: 20px;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  flex-wrap: wrap;
  padding: 20px;
}

.video-wrapper {
  position: relative;
  width: 400px;
  cursor: pointer;
  overflow: hidden;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  background: #000;
  user-select: none;
  -webkit-user-select: none;
}

.video-wrapper video {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
  pointer-events: none;
}

.video-wrapper:hover {
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.2);
}
</style>
