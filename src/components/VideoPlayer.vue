<script setup lang="ts">
defineOptions({
  name: 'VideoPlayer',
})

// 硬编码的数据 - 从 public/source 目录扫描得到
const sourceData = [
  { id: '19480503', video: '/source/19480503/generated.mp4', generated: '/source/19480503/generated.jpg', original: '/source/19480503/original.jpg' },
  { id: '19480554', video: '/source/19480554/generated.mp4', generated: '/source/19480554/generated.jpeg', original: '/source/19480554/original.jpeg' },
  { id: '19480632', video: '/source/19480632/generated.mp4', generated: '/source/19480632/generated.jpg', original: '/source/19480632/original.jpg' },
  { id: '19480695', video: '/source/19480695/generated.mp4', generated: '/source/19480695/generated.jpg', original: '/source/19480695/original.jpg' },
  { id: '19480875', video: '/source/19480875/generated.mp4', generated: '/source/19480875/generated.jpg', original: '/source/19480875/original.jpg' },
  { id: '19480952', video: '/source/19480952/generated.mp4', generated: '/source/19480952/generated.jpg', original: '/source/19480952/original.jpg' },
  { id: '19488487', video: '/source/19488487/generated.mp4', generated: '/source/19488487/generated.jpg', original: '/source/19488487/original.jpg' },
  { id: '19488757', video: '/source/19488757/generated.mp4', generated: '/source/19488757/generated.jpg', original: '/source/19488757/original.jpg' },
  { id: '19489106', video: '/source/19489106/generated.mp4', generated: '/source/19489106/generated.jpeg', original: '/source/19489106/original.jpeg' },
  { id: '19489850', video: '/source/19489850/generated.mp4', generated: '/source/19489850/generated.jpg', original: '/source/19489850/original.jpg' },
  { id: '19489938', video: '/source/19489938/generated.mp4', generated: '/source/19489938/generated.jpg', original: '/source/19489938/original.jpg' },
  { id: '19489986', video: '/source/19489986/generated.mp4', generated: '/source/19489986/generated.jpg', original: '/source/19489986/original.jpg' },
]

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
  rotateX: number
  rotateY: number
}

const CONFIG = {
  SEEK_THROTTLE: 10,
  AUTO_PLAY_SEEK_THROTTLE: 10,
  AUTO_PLAY_DURATION: 8000,
  MIN_DIFF: { auto: 0.001, drag: 0.15 },
}

const currentIndex = ref(0)
const videoRef = ref<HTMLVideoElement | null>(null)
const wrapperRef = ref<HTMLElement | null>(null)
const state = ref<VideoState>({
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
  rotateX: 0,
  rotateY: 0,
})

const currentItem = computed(() => sourceData[currentIndex.value])

function getPercentage(video: HTMLVideoElement) {
  if (!video.duration || Number.isNaN(video.duration))
    return 0
  return video.currentTime / video.duration
}

function startAutoPlay(direction: 1 | -1) {
  const video = videoRef.value
  if (!video)
    return

  if (state.value.rafId) {
    cancelAnimationFrame(state.value.rafId)
    state.value.rafId = null
  }
  state.value.isAutoPlaying = true
  state.value.playDirection = direction
  state.value.autoPlayStartPercentage = getPercentage(video)
  state.value.autoPlayStartTime = Date.now()
  const remaining = direction === 1
    ? (1 - state.value.autoPlayStartPercentage)
    : state.value.autoPlayStartPercentage
  state.value.autoPlayDuration = remaining * CONFIG.AUTO_PLAY_DURATION
  state.value.lastSeekTime = 0
  video.pause()
  updateVideoTime()
}

function calculateTargetTime(clientX: number) {
  const wrapper = wrapperRef.value
  const video = videoRef.value
  if (!wrapper || !video)
    return 0

  const rect = wrapper.getBoundingClientRect()
  let p = (clientX - rect.left) / rect.width
  p = p < 0 ? 1 - (Math.abs(p) % 1) : p > 1 ? p % 1 : p
  p = Math.max(0, Math.min(1, p))
  return Math.min(video.duration - 0.01, p * video.duration)
}

function updateVideoTime() {
  const video = videoRef.value
  if (!video)
    return

  if (!video.duration || Number.isNaN(video.duration)) {
    state.value.rafId = null
    return
  }

  if (state.value.isSeeking && !state.value.isAutoPlaying) {
    state.value.rafId = requestAnimationFrame(updateVideoTime)
    return
  }

  let targetTime: number

  if (state.value.isHovering) {
    targetTime = calculateTargetTime(state.value.mouseX)
  }
  else if (state.value.isAutoPlaying) {
    const elapsed = Date.now() - state.value.autoPlayStartTime

    if (elapsed >= state.value.autoPlayDuration) {
      if (state.value.playDirection === 1) {
        video.currentTime = video.duration - 0.01
        state.value.playDirection = -1
        state.value.autoPlayStartPercentage = 1
      }
      else {
        video.currentTime = 0
        state.value.playDirection = 1
        state.value.autoPlayStartPercentage = 0
      }
      state.value.autoPlayStartTime = Date.now()
      state.value.autoPlayDuration = CONFIG.AUTO_PLAY_DURATION
      state.value.rafId = requestAnimationFrame(updateVideoTime)
      return
    }

    const progress = Math.min(1, elapsed / state.value.autoPlayDuration)

    if (state.value.playDirection === 1) {
      const startTime = state.value.autoPlayStartPercentage * video.duration
      const endTime = video.duration - 0.01
      targetTime = startTime + progress * (endTime - startTime)
      targetTime = Math.min(endTime, targetTime)
    }
    else {
      const startTime = state.value.autoPlayStartPercentage * video.duration
      const endTime = 0
      targetTime = startTime - progress * (startTime - endTime)
      targetTime = Math.max(endTime, targetTime)
    }

    const p = targetTime / video.duration
    if (state.value.playDirection === 1 && p >= 0.995) {
      state.value.playDirection = -1
      state.value.autoPlayStartPercentage = 1
      state.value.autoPlayStartTime = Date.now()
      state.value.autoPlayDuration = CONFIG.AUTO_PLAY_DURATION
      targetTime = video.duration - 0.01
    }
    else if (state.value.playDirection === -1 && p <= 0.005) {
      state.value.playDirection = 1
      state.value.autoPlayStartPercentage = 0
      state.value.autoPlayStartTime = Date.now()
      state.value.autoPlayDuration = CONFIG.AUTO_PLAY_DURATION
      targetTime = 0
    }
  }
  else {
    state.value.rafId = null
    return
  }

  const timeDiff = Math.abs(video.currentTime - targetTime)
  const now = Date.now()
  const throttle = state.value.isAutoPlaying
    ? CONFIG.AUTO_PLAY_SEEK_THROTTLE
    : CONFIG.SEEK_THROTTLE
  const minDiff = state.value.isAutoPlaying ? CONFIG.MIN_DIFF.auto : CONFIG.MIN_DIFF.drag

  if (state.value.isAutoPlaying && !state.value.isSeeking) {
    if (now - state.value.lastSeekTime >= throttle) {
      try {
        video.currentTime = targetTime
        state.value.lastSeekTime = now
      }
      catch (e) {
        console.error('Error setting video time:', e)
      }
    }
  }
  else if (state.value.isHovering && timeDiff > minDiff && !state.value.isSeeking) {
    if (now - state.value.lastSeekTime >= throttle) {
      try {
        video.currentTime = targetTime
        state.value.lastSeekTime = now
      }
      catch (e) {
        console.error('Error setting video time:', e)
      }
    }
  }

  state.value.rafId = requestAnimationFrame(updateVideoTime)
}

function goTo(index: number) {
  if (index === currentIndex.value)
    return

  // 停止当前动画
  if (state.value.rafId) {
    cancelAnimationFrame(state.value.rafId)
    state.value.rafId = null
  }

  // 重置状态
  state.value.isHovering = false
  state.value.isAutoPlaying = false
  state.value.isSeeking = false
  state.value.rotateX = 0
  state.value.rotateY = 0

  currentIndex.value = index

  // 等待视频加载后开始播放
  nextTick(() => {
    const video = videoRef.value
    if (video) {
      video.currentTime = 0
      if (video.readyState >= 1) {
        startAutoPlay(1)
      }
    }
  })
}

function prev() {
  const newIndex = currentIndex.value === 0 ? sourceData.length - 1 : currentIndex.value - 1
  goTo(newIndex)
}

function next() {
  const newIndex = currentIndex.value === sourceData.length - 1 ? 0 : currentIndex.value + 1
  goTo(newIndex)
}

let cleanupFunction: (() => void) | null = null

function setupVideoListeners() {
  const video = videoRef.value
  const wrapper = wrapperRef.value
  if (!video || !wrapper)
    return

  // 清理之前的监听器
  if (cleanupFunction) {
    cleanupFunction()
    cleanupFunction = null
  }

  const handleLoadedMetadata = () => {
    video.currentTime = 0
    startAutoPlay(1)
  }

  const handleEnded = () => {
    if (!state.value.isHovering && !state.value.isAutoPlaying) {
      startAutoPlay(-1)
    }
  }

  const handleSeeking = () => {
    state.value.isSeeking = true
  }

  const handleSeeked = () => {
    state.value.isSeeking = false
  }

  const handleMouseEnter = () => {
    state.value.isHovering = true
    state.value.isAutoPlaying = false
    video.pause()
    if (!state.value.rafId) {
      state.value.rafId = requestAnimationFrame(updateVideoTime)
    }
  }

  const handleMouseLeave = () => {
    state.value.isHovering = false
    state.value.isSeeking = false
    if (state.value.rafId) {
      cancelAnimationFrame(state.value.rafId)
      state.value.rafId = null
    }
    state.value.isAutoPlaying = false
    state.value.lastSeekTime = 0
    state.value.rotateX = 0
    state.value.rotateY = 0
    const direction = getPercentage(video) < 0.5 ? -1 : 1
    startAutoPlay(direction)
  }

  const handleMouseMove = (e: MouseEvent) => {
    if (state.value.isHovering) {
      state.value.mouseX = e.clientX
      if (!state.value.rafId) {
        state.value.rafId = requestAnimationFrame(updateVideoTime)
      }

      const rect = wrapper.getBoundingClientRect()
      const centerX = rect.left + rect.width / 2
      const centerY = rect.top + rect.height / 2

      const percentX = (e.clientX - centerX) / (rect.width / 2)
      const percentY = (e.clientY - centerY) / (rect.height / 2)

      const maxRotate = 15

      state.value.rotateX = percentY * maxRotate
      state.value.rotateY = -percentX * maxRotate
    }
  }

  video.addEventListener('loadedmetadata', handleLoadedMetadata)
  video.addEventListener('ended', handleEnded)
  video.addEventListener('seeking', handleSeeking)
  video.addEventListener('seeked', handleSeeked)
  wrapper.addEventListener('mouseenter', handleMouseEnter)
  wrapper.addEventListener('mouseleave', handleMouseLeave)
  wrapper.addEventListener('mousemove', handleMouseMove)

  cleanupFunction = () => {
    video.removeEventListener('loadedmetadata', handleLoadedMetadata)
    video.removeEventListener('ended', handleEnded)
    video.removeEventListener('seeking', handleSeeking)
    video.removeEventListener('seeked', handleSeeked)
    wrapper.removeEventListener('mouseenter', handleMouseEnter)
    wrapper.removeEventListener('mouseleave', handleMouseLeave)
    wrapper.removeEventListener('mousemove', handleMouseMove)
    if (state.value.rafId) {
      cancelAnimationFrame(state.value.rafId)
      state.value.rafId = null
    }
  }
}

onMounted(() => {
  nextTick(() => {
    setupVideoListeners()
  })
})

watch(currentIndex, () => {
  nextTick(() => {
    setupVideoListeners()
  })
})

onUnmounted(() => {
  if (cleanupFunction) {
    cleanupFunction()
    cleanupFunction = null
  }
})
</script>

<template>
  <div class="carousel-container">
    <!-- 左侧：视频播放区域 -->
    <div class="left-panel">
      <div class="video-section">
        <div ref="wrapperRef" class="video-wrapper">
          <video
            :key="currentItem.id"
            ref="videoRef"
            :src="currentItem.video"
            muted
            playsinline
            preload="auto"
            :style="{
              transform: `rotateX(${state.rotateX}deg) rotateY(${state.rotateY}deg)`,
            }"
          />
        </div>
      </div>

      <div class="indicator">
        <button class="nav-arrow nav-prev" @click="prev">
          <span>‹</span>
        </button>
        <span class="current">{{ currentIndex + 1 }}</span>
        <span class="separator">/</span>
        <span class="total">{{ sourceData.length }}</span>
        <button class="nav-arrow nav-next" @click="next">
          <span>›</span>
        </button>
      </div>
      <div class="tips">
        1. 鼠标移入的时候左右滑动鼠标可以旋转<br>
        2. 鼠标移除从当前位置继续自动旋转
      </div>
    </div>

    <!-- 右侧：导航和原图 -->
    <div class="right-panel">
      <div class="original-section">
        <h3 class="section-title">
          原图 Original
        </h3>
        <div class="original-image-wrapper">
          <img :key="`original-${currentItem.id}`" :src="currentItem.original" :alt="`Original ${currentItem.id}`" class="original-image">
        </div>
      </div>

      <div class="thumbnails-section">
        <h3 class="section-title">
          导航 Navigation
        </h3>
        <div class="thumbnails-grid">
          <div
            v-for="(item, index) in sourceData"
            :key="item.id"
            class="thumbnail-item"
            :class="{ active: index === currentIndex }"
            @click="goTo(index)"
          >
            <img :src="item.generated" :alt="`Thumbnail ${item.id}`">
            <span class="thumbnail-index">{{ index + 1 }}</span>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.carousel-container {
  display: flex;
  width: 100%;
  min-height: 100vh;
  background: linear-gradient(135deg, #1a1a2e 0%, #16213e 100%);
}

.left-panel {
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  padding: 40px;
  background: rgba(0, 0, 0, 0.2);
}

.video-section {
  display: flex;
  align-items: center;
  gap: 20px;
}

.nav-arrow {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  border: 2px solid rgba(255, 255, 255, 0.3);
  background: rgba(255, 255, 255, 0.1);
  color: white;
  font-size: 20px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s ease;
  backdrop-filter: blur(10px);
}

.nav-arrow:hover {
  background: rgba(255, 255, 255, 0.2);
  border-color: rgba(255, 255, 255, 0.5);
  transform: scale(1.1);
}

.nav-arrow span {
  line-height: 1;
  margin-top: -2px;
}

.video-wrapper {
  position: relative;
  width: 750px;
  max-width: 80vw;
  cursor: pointer;
  overflow: visible;
  border-radius: 12px;
  user-select: none;
  -webkit-user-select: none;
  perspective: 1000px;
  box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.5);
  background-color: #fff;
  overflow: hidden;
}

.video-wrapper video {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
  pointer-events: none;
  border-radius: 12px;
  transform-style: preserve-3d;
  transition:
    transform 0.1s ease-out,
    box-shadow 0.3s ease;
  will-change: transform;
}

.video-wrapper:hover video {
  transition:
    transform 0.05s ease-out,
    box-shadow 0.3s ease;
}

.indicator {
  margin-top: 30px;
  font-size: 18px;
  color: rgba(255, 255, 255, 0.7);
  font-weight: 500;
  display: flex;
  align-items: center;
  gap: 16px;
}

.indicator .current {
  color: #fff;
  font-size: 24px;
  font-weight: 700;
}

.indicator .separator {
  margin: 0 -10px;
  color: rgba(255, 255, 255, 0.4);
}

.right-panel {
  flex: 1;
  display: flex;
  flex-direction: column;
  padding: 40px;
  overflow-y: auto;
  background: rgba(255, 255, 255, 0.02);
}

.section-title {
  color: rgba(255, 255, 255, 0.9);
  font-size: 16px;
  font-weight: 600;
  margin-bottom: 16px;
  letter-spacing: 0.5px;
  text-transform: uppercase;
}

.original-section {
  margin-bottom: 40px;
}

.original-image-wrapper {
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
  background: rgba(0, 0, 0, 0.3);
}

.original-image {
  width: 100%;
  max-height: 300px;
  object-fit: contain;
  display: block;
  transition: transform 0.3s ease;
}

.thumbnails-section {
  flex: 1;
}

.thumbnails-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(100px, 1fr));
  gap: 12px;
}

.thumbnail-item {
  position: relative;
  aspect-ratio: 1;
  border-radius: 8px;
  overflow: hidden;
  cursor: pointer;
  transition: all 0.3s ease;
  border: 2px solid transparent;
  background: rgba(0, 0, 0, 0.3);
}

.thumbnail-item img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  transition: transform 0.3s ease;
}

.thumbnail-item:hover {
  transform: translateY(-4px);
  box-shadow: 0 8px 25px rgba(0, 0, 0, 0.4);
}

.thumbnail-item:hover img {
  transform: scale(1.1);
}

.thumbnail-item.active {
  border-color: #4fc3f7;
  box-shadow: 0 0 20px rgba(79, 195, 247, 0.4);
}

.thumbnail-index {
  position: absolute;
  bottom: 6px;
  right: 6px;
  background: rgba(0, 0, 0, 0.7);
  color: white;
  font-size: 12px;
  font-weight: 600;
  padding: 2px 8px;
  border-radius: 4px;
  backdrop-filter: blur(4px);
}

.tips {
  margin-top: 20px;
  font-size: 14px;
  color: rgba(255, 255, 255, 0.7);
  font-weight: 500;
}

.thumbnail-item.active .thumbnail-index {
  background: #4fc3f7;
  color: #1a1a2e;
}

/* 响应式设计 */
@media (max-width: 1024px) {
  .carousel-container {
    flex-direction: column;
  }

  .left-panel,
  .right-panel {
    flex: none;
    width: 100%;
  }

  .left-panel {
    min-height: 60vh;
  }

  .right-panel {
    min-height: 40vh;
  }

  .video-wrapper {
    width: 400px;
  }
}

@media (max-width: 640px) {
  .video-wrapper {
    width: 280px;
  }

  .nav-arrow {
    width: 40px;
    height: 40px;
    font-size: 22px;
  }

  .thumbnails-grid {
    grid-template-columns: repeat(auto-fill, minmax(70px, 1fr));
  }
}
</style>
