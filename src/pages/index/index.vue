<script lang="ts" setup>
import { onMounted, onUnmounted, ref } from 'vue'

defineOptions({
  name: 'Home',
})
definePage({
  type: 'home',
  style: {
    navigationStyle: 'custom',
    navigationBarTitleText: '首页',
  },
})

// 燃烧状态
const isBurning = ref(false)
const showBlessing = ref(false)
const buttonText = ref('上香')

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
    size: Math.random() * 8 + 4,
    life: 1,
    decay: Math.random() * 0.005 + 0.003,
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
        const stickHeight = stickHeights.value[i]
        smokeParticles.value.push(createParticle(canvas.width / 2 + offsetX, canvas.height - stickHeight))
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
    ctx.globalAlpha = particle.life * 0.3
    const gradient = ctx.createRadialGradient(
      particle.x,
      particle.y,
      0,
      particle.x,
      particle.y,
      particle.size,
    )
    gradient.addColorStop(0, 'rgba(200, 200, 200, 0.8)')
    gradient.addColorStop(0.4, 'rgba(150, 150, 150, 0.4)')
    gradient.addColorStop(1, 'rgba(100, 100, 100, 0)')
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
        return height - 0.3 // 慢慢燃烧变短
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
  isBurning.value = false
  buttonText.value = '再次上香'
  showBlessing.value = false
  // 重置香的高度
  setTimeout(() => {
    stickHeights.value = [...initialHeights]
  }, 1000)
}

// 开始上香
function startBurning() {
  if (isBurning.value)
    return

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
  // 初始化 Canvas
  if (smokeCanvas.value) {
    smokeCanvas.value.width = 200
    smokeCanvas.value.height = 300
    canvasCtx.value = smokeCanvas.value.getContext('2d')
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
  <view class="min-h-screen flex flex-col items-center bg-white px-4 pt-safe">
    <!-- 顶部空白区域 -->
    <view class="flex-1" />

    <!-- 图片和香烟区域 -->
    <view class="relative mb-4 max-w-400 w-full">
      <!-- 关公图片 - 在香烟上方 -->
      <view class="guangong-wrapper">
        <image
          src="/static/guangong.jpg"
          alt="关公"
          class="guangong-image"
          mode="aspectFit"
        />
      </view>

      <!-- 香烟容器 - 在图片后方 -->
      <view v-show="isBurning" class="incense-wrapper">
        <view class="incense-container">
          <!-- 烟雾 Canvas -->
          <canvas id="smokeCanvas" ref="smokeCanvas" />
          <!-- 三根香烟 -->
          <view
            class="incense-stick stick-left"
            :style="{ height: `${stickHeights[0]}px` }"
          />
          <view
            class="incense-stick stick-center"
            :style="{ height: `${stickHeights[1]}px` }"
          />
          <view
            class="incense-stick stick-right"
            :style="{ height: `${stickHeights[2]}px` }"
          />
        </view>
      </view>

      <!-- 图片 - 在香烟前方 -->
      <image src="/static/desk.png" alt="desk" class="desk-image h-auto w-full" mode="widthFix" />
    </view>

    <!-- 上香按钮 - 靠下位置 -->
    <view class="button-wrapper">
      <button
        class="burn-btn"
        :class="{ burning: isBurning }"
        :disabled="isBurning"
        @click="startBurning"
      >
        {{ buttonText }}
      </button>
    </view>

    <!-- 底部留白 -->
    <view class="h-10" />
  </view>
</template>

<style scoped>
/* 关公图片包装器 */
.guangong-wrapper {
  position: absolute;
  left: 50%;
  bottom: 65%;
  transform: translateX(-50%);
  z-index: 0;
  width: 200px;
  height: 250px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.guangong-image {
  width: 100%;
  height: 100%;
  object-fit: contain;
}

/* 香烟包装器 - 在图片后方 */
.incense-wrapper {
  position: absolute;
  left: 50%;
  bottom: 55%;
  transform: translateX(-50%);
  z-index: 1;
}

/* 桌子图片 - 在香烟前方 */
.desk-image {
  position: relative;
  z-index: 2;
  margin-top: 40px;
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
  bottom: 100%;
  transform: translateX(-50%);
  width: 200px;
  height: 300px;
  pointer-events: none;
}

/* 按钮包装器 - 靠下 */
.button-wrapper {
  margin-top: 30px;
  margin-bottom: 10px;
}

/* 上香按钮 - 变小 */
.burn-btn {
  padding: 12px 40px;
  font-size: 18px;
  font-weight: bold;
  color: #fff;
  background: linear-gradient(135deg, #c41e3a 0%, #8b0000 100%);
  border: none;
  border-radius: 25px;
  box-shadow: 0 4px 15px rgba(139, 0, 0, 0.4);
  transition: all 0.3s ease;
  font-family: 'Microsoft YaHei', 'SimSun', serif;
  letter-spacing: 4px;
  line-height: 1.5;
}

.burn-btn:active {
  transform: scale(0.95);
}

.burn-btn.burning {
  background: linear-gradient(135deg, #666 0%, #333 100%);
  opacity: 0.8;
}
</style>
