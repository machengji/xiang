<script lang="ts" setup>
import { onMounted, onUnmounted, ref } from 'vue'

defineOptions({
  name: 'Home',
})
definePage({
  type: 'home',
  style: {
    navigationBarTitleText: '诚心上香',
    navigationBarBackgroundColor: '#ffffff',
    navigationBarTextStyle: 'black',
  },
})

// 图片数据
const imageData = ref([
  {
    id: 1,
    src: '/static/guangong.jpg',
    title: '关公',
    description: '忠义千秋',
  },
  {
    id: 2,
    src: '/static/caisheng.jpg',
    title: '财神',
    description: '财源广进',
  },
  {
    id: 3,
    src: '/static/pudu.jpg',
    title: '观音',
    description: '普度众生',
  },
])

// 当前选中的图片索引
const currentImageIndex = ref(0)

// 燃烧状态
const isBurning = ref(false)
const showBlessing = ref(false)
const buttonText = ref('上香')

// 标题和描述的可编辑性
const currentTitle = ref('')
const currentDescription = ref('')

// 初始化和监听图片切换
onMounted(() => {
  currentTitle.value = imageData.value[currentImageIndex.value].title
  currentDescription.value = imageData.value[currentImageIndex.value].description
})

function updateImageData() {
  imageData.value[currentImageIndex.value].title = currentTitle.value
  imageData.value[currentImageIndex.value].description = currentDescription.value
}

// 香烟高度（燃烧变短）
const stickHeights = ref([140, 150, 140]) // 左中右三根香的高度
const initialHeights = [140, 150, 140]

// Canvas 相关
const smokeCanvas = ref<HTMLCanvasElement | null>(null)
const canvasCtx = ref<CanvasRenderingContext2D | null>(null)
const animationId = ref<number | null>(null)
const burnInterval = ref<NodeJS.Timeout | null>(null)

// 烟雾粒子数组
interface SmokeParticle {
  x: number
  y: number
  vx: number
  vy: number
  size: number
  life: number
  decay: number
}

const smokeParticles = ref<SmokeParticle[]>([])

// 创建烟雾粒子
function createParticle(x: number, y: number): SmokeParticle {
  return {
    x: x + (Math.random() - 0.5) * 20,
    y,
    vx: (Math.random() - 0.5) * 0.5,
    vy: -Math.random() * 1.5 - 0.5,
    size: Math.random() * 16 + 8, // 增大粒子尺寸
    life: 1,
    decay: Math.random() * 0.003 + 0.002, // 减慢衰减速度
  }
}

// 动画循环
function animate() {
  const ctx = canvasCtx.value
  const canvas = smokeCanvas.value
  if (!ctx || !canvas)
    return

  ctx.clearRect(0, 0, canvas.width, canvas.height)

  // 添加新粒子
  if (isBurning.value) {
    for (let i = 0; i < 3; i++) {
      if (Math.random() < 0.4) {
        const offsetX = (i - 1) * 15
        // 根据当前香的高度计算烟雾位置
        // 注意：Canvas 内部尺寸是 400x600，显示尺寸是 200x300
        const stickHeight = stickHeights.value[i]
        const scaleY = 2 // 缩放比例
        smokeParticles.value.push(createParticle(
          canvas.width / 2 + offsetX * scaleY,
          canvas.height - stickHeight * scaleY,
        ))
      }
    }
  }

  // 更新和绘制粒子
  smokeParticles.value = smokeParticles.value.filter((particle) => {
    // 更新位置
    const wind = Math.sin(Date.now() * 0.001 + particle.y * 0.01) * 0.2
    particle.x += particle.vx + wind
    particle.y += particle.vy
    particle.size += 0.1
    particle.life -= particle.decay

    // 绘制粒子
    ctx.save()
    ctx.globalAlpha = particle.life * 0.5 // 增加透明度
    const gradient = ctx.createRadialGradient(
      particle.x,
      particle.y,
      0,
      particle.x,
      particle.y,
      particle.size,
    )
    gradient.addColorStop(0, 'rgba(220, 220, 220, 0.9)')
    gradient.addColorStop(0.4, 'rgba(180, 180, 180, 0.5)')
    gradient.addColorStop(1, 'rgba(150, 150, 150, 0)')
    ctx.fillStyle = gradient
    ctx.beginPath()
    ctx.arc(particle.x, particle.y, particle.size, 0, Math.PI * 2)
    ctx.fill()
    ctx.restore()

    return particle.life > 0
  })

  if (isBurning.value || smokeParticles.value.length > 0) {
    animationId.value = requestAnimationFrame(animate)
  }
}

// 燃烧变短效果
function startBurningEffect() {
  // 每100毫秒减少一点高度
  burnInterval.value = setInterval(() => {
    let allBurned = true
    stickHeights.value = stickHeights.value.map((height, index) => {
      if (height > 20) {
        allBurned = false
        return height - 0.2 // 减慢燃烧速度，约60秒内烧完
      }
      return height
    })

    // 如果全部烧完了，停止燃烧
    if (allBurned) {
      stopBurning()
    }
  }, 100)
}

// 停止燃烧
function stopBurning() {
  if (burnInterval.value) {
    clearInterval(burnInterval.value)
    burnInterval.value = null
  }
  showBlessing.value = false
  // 5秒后允许重新上香
  setTimeout(() => {
    isBurning.value = false
    buttonText.value = '再次上香'
    // 重置香的高度
    stickHeights.value = [...initialHeights]
  }, 5000)
}

// 切换到上一张图片
function previousImage() {
  currentImageIndex.value = (currentImageIndex.value - 1 + imageData.value.length) % imageData.value.length
}

// 切换到下一张图片
function nextImage() {
  currentImageIndex.value = (currentImageIndex.value + 1) % imageData.value.length
}

// 切换图片（顺序切换）
function handleSwitch() {
  currentImageIndex.value = (currentImageIndex.value + 1) % imageData.value.length
  syncContent()
}

function syncContent() {
  currentTitle.value = imageData.value[currentImageIndex.value].title
  currentDescription.value = imageData.value[currentImageIndex.value].description
}

// 上传图片
function handleUpload() {
  uni.chooseImage({
    count: 1,
    sizeType: ['compressed'],
    sourceType: ['album', 'camera'],
    success: (res) => {
      const tempFilePath = res.tempFilePaths[0]
      const newItem = {
        id: Date.now(),
        src: tempFilePath,
        title: '自定义',
        description: '诚心祈福',
      }
      imageData.value.push(newItem)
      currentImageIndex.value = imageData.value.length - 1
      syncContent()
    },
  })
}

// 点击图片切换（随机切换）
function switchImage() {
  let newIndex
  if (imageData.value.length <= 1)
    return
  do {
    newIndex = Math.floor(Math.random() * imageData.value.length)
  } while (newIndex === currentImageIndex.value)

  currentImageIndex.value = newIndex
  syncContent()
}

// 开始上香
function startBurning() {
  // 先清理之前的定时器，避免重复点击导致多个定时器同时运行
  if (burnInterval.value) {
    clearInterval(burnInterval.value)
    burnInterval.value = null
  }
  if (animationId.value) {
    cancelAnimationFrame(animationId.value)
    animationId.value = null
  }

  // 重置高度
  stickHeights.value = [...initialHeights]

  isBurning.value = true
  buttonText.value = '燃烧中...'

  // 开始动画
  animate()

  // 开始燃烧变短效果
  startBurningEffect()

  // 显示祈福文字
  setTimeout(() => {
    showBlessing.value = true
  }, 500)
}

onMounted(() => {
  // 在 uni-app 中，对于 H5 平台我们可以直接操作 DOM
  // 检查是否是 H5 平台（通过判断是否有 document 对象）
  if (typeof document !== 'undefined') {
    // 动态创建 Canvas 元素
    const canvasContainer = document.querySelector('[ref="canvasContainer"]') || document.createElement('div')
    const canvasElement = document.createElement('canvas')
    canvasElement.id = 'smokeCanvas'
    canvasElement.width = 400
    canvasElement.height = 600
    canvasElement.style.position = 'absolute'
    canvasElement.style.left = '50%'
    canvasElement.style.bottom = '0'
    canvasElement.style.transform = 'translateX(-50%)'
    canvasElement.style.width = '200px'
    canvasElement.style.height = '300px'
    canvasElement.style.pointerEvents = 'none'
    canvasElement.style.zIndex = '10'

    // 添加到 DOM
    if (canvasContainer) {
      canvasContainer.appendChild(canvasElement)
    }

    canvasCtx.value = canvasElement.getContext('2d')
    console.log('Canvas context created:', canvasCtx.value)
  }
  else {
    // 对于其他平台，我们可能需要使用 uni.createCanvasContext
    // 这里可以添加其他平台的 Canvas 初始化代码
  }
})

onUnmounted(() => {
  if (animationId.value) {
    cancelAnimationFrame(animationId.value)
  }
  if (burnInterval.value) {
    clearInterval(burnInterval.value)
  }
})
</script>

<template>
  <view class="min-h-screen flex flex-col items-center bg-white px-4">
    <!-- 1. 顶部标题和按钮区域 - 支持编辑 -->
    <view class="z-10 mb-2 mt-1 w-full flex flex-col items-center text-center">
      <input
        v-model="currentTitle"
        class="w-full border-none bg-transparent text-center text-2xl text-red-900 font-bold tracking-widest focus:outline-none"
        placeholder="请输入标题"
        @blur="updateImageData"
      >
      <input
        v-model="currentDescription"
        class="mt-0.5 w-full border-none bg-transparent text-center text-sm text-gray-600 italic focus:outline-none"
        placeholder="请输入描述"
        @blur="updateImageData"
      >
    </view>

    <!-- 2. 内容核心区 -->
    <view class="relative w-full flex flex-col items-center">
      <!-- 神像图片 - 改为相对定位，自然排在文字下方 -->
      <view class="guangong-container">
        <image
          :src="imageData[currentImageIndex].src"
          :alt="imageData[currentImageIndex].title"
          class="guangong-main-img"
          mode="aspectFit"
          @click="switchImage"
        />
      </view>

      <!-- 香烟容器 - 放在神像和桌子之间 -->
      <view v-show="isBurning" class="incense-overlay">
        <view class="incense-container">
          <view ref="canvasContainer" />
          <view class="incense-stick stick-left" :style="{ height: `${stickHeights[0]}px` }" />
          <view class="incense-stick stick-center" :style="{ height: `${stickHeights[1]}px` }" />
          <view class="incense-stick stick-right" :style="{ height: `${stickHeights[2]}px` }" />
        </view>
      </view>

      <!-- 桌子图片 - 使用负 margin 向上移动，遮挡神像底部 -->
      <image src="/static/desk.png" alt="desk" class="desk-display" mode="widthFix" />
    </view>

    <!-- 上香按钮 - 靠下位置 -->
    <view class="button-wrapper">
      <button
        class="burn-btn flex items-center justify-center"
        :class="{ burning: isBurning }"
        @click="startBurning"
      >
        <view v-if="!isBurning" class="i-carbon-fire text-3xl" />
        <view v-else class="i-carbon-hourglass text-3xl" />
      </button>
    </view>

    <!-- 底部浮动工具栏 -->
    <view class="floating-toolbar">
      <view class="toolbar-btn" @click="handleSwitch">
        <view class="i-carbon-rotate text-lg" />
        <text class="toolbar-label">切换</text>
      </view>
      <view class="toolbar-divider" />
      <view class="toolbar-btn" @click="handleUpload">
        <view class="i-carbon-upload text-lg" />
        <text class="toolbar-label">上传</text>
      </view>
    </view>

    <!-- 底部留白 -->
    <view class="h-10" />
  </view>
</template>

<style scoped>
/* 图片标题样式 */
.text-center {
  text-align: center;
}

.text-3xl {
  font-size: 30px;
  line-height: 1.2;
}

.text-red-900 {
  color: #7f1d1d;
}

.text-gray-600 {
  color: #6b7280;
  font-size: 14px;
}

.mt-1 {
  margin-top: 4px;
}

.mt-6 {
  margin-top: 24px;
}

.mb-6 {
  margin-bottom: 24px;
}

.tracking-widest {
  letter-spacing: 0.1em;
}

.italic {
  font-style: italic;
}

/* 底部浮动工具栏 */
.floating-toolbar {
  position: fixed;
  bottom: 70px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  align-items: center;
  background: rgba(255, 255, 255, 0.75);
  backdrop-filter: blur(12px);
  border-radius: 20px;
  padding: 6px 16px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.08);
  z-index: 100;
}

.toolbar-btn {
  display: flex;
  align-items: center;
  gap: 4px;
  padding: 6px 12px;
  color: #7f1d1d;
  cursor: pointer;
  transition: opacity 0.2s;
}

.toolbar-btn:active {
  opacity: 0.5;
}

.toolbar-label {
  font-size: 12px;
  color: #7f1d1d;
}

.toolbar-divider {
  width: 1px;
  height: 16px;
  background: rgba(127, 29, 29, 0.15);
}

/* 神像容器 - 不再使用绝对定位漂浮 */
.guangong-container {
  width: 240px;
  height: 300px;
  z-index: 0;
  display: flex;
  align-items: center;
  justify-content: center;
}

.guangong-main-img {
  width: 100%;
  height: 100%;
  object-fit: contain;
  cursor: pointer;
}

/* 香烟层 - 绝对定位在桌子上方 */
.incense-overlay {
  position: absolute;
  left: 50%;
  bottom: 150px; /* 继续向上移动香烟位置 */
  transform: translateX(-50%);
  z-index: 1;
}

/* 桌子显示 - 向上移动覆盖图片底部 */
.desk-display {
  width: 100%;
  height: auto;
  position: relative;
  z-index: 2;
  margin-top: -20px; /* 显著减小负边距，使其向下移动 */
}

/* 香烟容器 */
.incense-container {
  position: relative;
  width: 60px;
  height: 200px;
  pointer-events: none;
}

/* 单根香烟 */
.incense-stick {
  position: absolute;
  bottom: 0;
  width: 3px;
  background: linear-gradient(to bottom, #d4a574 0%, #c4956a 70%, #b8956a 100%);
  border-radius: 1px;
  transform-origin: bottom center;
  transition: height 0.1s linear;
}

.incense-stick::before {
  content: '';
  position: absolute;
  top: -4px;
  left: 50%;
  transform: translateX(-50%);
  width: 6px;
  height: 8px;
  background: radial-gradient(ellipse at center, #ff6b35 0%, #ff4500 50%, #8b0000 100%);
  border-radius: 50%;
  box-shadow:
    0 0 10px #ff4500,
    0 0 20px #ff6b35;
  animation: glow 1.5s ease-in-out infinite alternate;
}

.stick-left {
  left: 50%;
  transform: translateX(-50%) rotate(-5deg);
}

.stick-center {
  left: 50%;
  transform: translateX(-50%);
}

.stick-right {
  left: 50%;
  transform: translateX(-50%) rotate(5deg);
}

@keyframes glow {
  0% {
    box-shadow:
      0 0 5px #ff4500,
      0 0 10px #ff6b35;
  }
  100% {
    box-shadow:
      0 0 15px #ff4500,
      0 0 30px #ff6b35,
      0 0 40px #ff8c00;
  }
}

/* 烟雾 Canvas */
#smokeCanvas {
  position: absolute;
  left: 50%;
  bottom: 0;
  transform: translateX(-50%);
  width: 200px;
  height: 300px;
  pointer-events: none;
  z-index: 10;
}

/* 按钮包装器 - 靠下 */
.button-wrapper {
  margin-top: 20px;
  margin-bottom: 20px;
  width: 100%;
  display: flex;
  justify-content: center;
}

/* 上香按钮 - 圆形大按钮 */
.burn-btn {
  width: 60px;
  height: 60px;
  color: #fff;
  background: linear-gradient(135deg, #e11d48 0%, #9f1239 100%);
  border: none;
  border-radius: 50%;
  box-shadow:
    0 10px 20px -5px rgba(159, 18, 57, 0.4),
    inset 0 -4px 0 rgba(0, 0, 0, 0.1);
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  justify-content: center;
}

.burn-btn:active {
  transform: scale(0.96) translateY(2px);
  box-shadow: 0 5px 10px -3px rgba(159, 18, 57, 0.5);
}

.burn-btn.burning {
  background: linear-gradient(135deg, #4b5563 0%, #1f2937 100%);
  box-shadow: none;
  opacity: 0.9;
}
</style>
