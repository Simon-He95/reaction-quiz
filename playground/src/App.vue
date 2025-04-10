<script setup lang="ts">
import {
  collisionDetection,
  createElement,
  findElement,
  insertElement,
  removeElement,
  useEventListener,
  useIntersectionObserver,
  useRaf,
} from 'lazy-js-utils'
import { computed, onMounted, ref } from 'vue'
import { blocks } from './ball'

const gameState = ref('start') // 'start', 'game', 'ranking', 'gameover'
const playerName = ref('')
const rankings = ref<{ name: string, time: number, date: string, difficulty: string }[]>([])
const currentRankingTab = ref('simple') // 当前排行榜标签页
const isHighScore = ref(false) // 是否为新最高分

const filteredRankings = computed(() => {
  return rankings.value.filter(rank => rank.difficulty === currentRankingTab.value)
})

const top = ref(0)
const left = ref(0)
let over: () => void
const status = ref(true)
const times = ref(0)
const blood = ref(100)
const speed = ref(8)
const isHit = ref(false)
const difficulty = ref('simple') // 'simple' 或 'difficult'
const isMobile = ref(false) // 添加到顶部的ref声明

const target = ref<HTMLElement>()
let start = false
let timer: any = null

// 添加这个函数在组件加载后立即设置初始位置
onMounted(() => {
  // 获取当前鼠标位置，或者使用屏幕中心作为默认位置
  const initialX = window.innerWidth / 2
  const initialY = window.innerHeight / 2

  // 使用setTimeout确保target元素已渲染
  setTimeout(() => {
    if (target.value) {
      top.value = initialY - target.value.offsetWidth / 2
      left.value = initialX - target.value.offsetHeight / 2
    }
  }, 0)

  // 加载上次使用的用户名
  const savedName = localStorage.getItem('reaction-quiz-username')
  if (savedName) {
    playerName.value = savedName
  }

  // 加载排名
  loadRankings()

  // 设置触摸事件
  setupTouchEvents()

  // 添加设备检测并显示合适的游戏说明
  const isMobileDevice = /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent)
  isMobile.value = isMobileDevice
})

function mousemove(e: MouseEvent) {
  if (!target.value)
    return
  top.value = e.y - target.value.offsetWidth / 2
  left.value = e.x - target.value.offsetHeight / 2

  if (start)
    return

  startGameLogic()
}
useEventListener(document, 'mousemove', mousemove)
const w = window.innerWidth
const h = window.innerHeight

function chooseNearestEdge() {
  // 计算鼠标到四个边的距离
  const distances = [
    top.value, // 到上边的距离
    h - top.value, // 到下边的距离
    left.value, // 到左边的距离
    w - left.value, // 到右边的距离
  ]

  // 找出最近的边缘
  const minDistance = Math.min(...distances)
  const edgeIndex = distances.indexOf(minDistance)

  return edgeIndex // 0=上, 1=下, 2=左, 3=右
}

function generateObject() {
  const random = chooseNearestEdge()
  const div = createElement('div', {
    class: 'blocks',
  })
  const ball = blocks[Math.floor(Math.random() * blocks.length)]
  div.innerHTML = ball

  // 基础尺寸
  let width = 32
  let height = 32

  // 困难模式下随机增大球体尺寸
  let scaleFactor = 1
  if (difficulty.value === 'difficult') {
    // 40%的几率生成更大的球
    if (Math.random() > 0.6) {
      // 根据游戏时间动态调整最大缩放倍数
      // 初始最大倍数为2.5，随着时间增加到最大5倍
      const maxScale = Math.min(2.5 + (times.value / 60) * 2.5, 5.0)

      // 计算当前的缩放因子，范围在1到maxScale之间
      scaleFactor = 1 + Math.random() * (maxScale - 1)
      width *= scaleFactor
      height *= scaleFactor
    }
  }

  const randomWidth = getRandomWidth() + width / 2
  const randomHeight = getRandomHeight() + height / 2
  if (times.value > 60)
    speed.value++

  let [lat1, lng1] = getStartPosition(random)
  const initialStatus = `left:${lat1}px;top:${lng1}px;position:absolute;${scaleFactor !== 1 ? `transform:scale(${scaleFactor});` : ''}`
  div.setAttribute('style', initialStatus)

  let flag = false
  let flag1 = false
  useIntersectionObserver(
    div,
    ([{ isIntersecting }, observeable]) => {
      if (isIntersecting)
        flag = true
      if (observeable)
        flag1 = true
      if (flag && !isIntersecting)
        removeElement(div)
      if (flag1 && !observeable)
        removeElement(div)
    },
    {},
  )

  const [directionX, directionY] = getDirection()
  const stop = useRaf(() => {
    if (!status.value)
      return stop()
    gameover()
    lat1 += speed.value * directionX
    lng1 += speed.value * directionY
    div.style.left = `${lat1}px`
    div.style.top = `${lng1}px`
  }, {
    delta: 0.01,
  })
  useEventListener(div, 'mousemove', gameover)

  insertElement('main', div, null)

  function getRandomWidth(): number {
    const result = Math.random() * (w + 2 * width) - width
    if (result < 100)
      return getRandomWidth()
    return result
  }
  function getRandomHeight(): number {
    const result = Math.random() * (h + 2 * height) - h
    if (result < 100)
      return getRandomWidth() // 修正递归调用，原代码错误地调用了getRandomWidth
    return result
  }

  function getStartPosition(random: number): number[] {
    const start: Record<number, number[]> = {
      0: [randomWidth, -height],
      1: [randomWidth, height + h],
      2: [-width, randomHeight],
      3: [width + w, randomHeight],
    }
    return start[random]
  }
  function sqrt(x: number, y: number) {
    return Math.sqrt(x * x + y * y)
  }
  function getDirection() {
    const symbolX
        = random === 0 || random === 1
          ? randomWidth - left.value > 0
            ? -1
            : 1
          : random === 2
            ? 1
            : -1

    const symbolY
        = random === 0 ? 1 : random === 1 ? -1 : randomHeight - top.value > 0 ? -1 : 1
    if (random === 0) {
      return [
        symbolX
        * Math.abs(
          (randomWidth - left.value)
          / sqrt(randomWidth - left.value, top.value + height / 2),
        ),
        symbolY
        * Math.abs(
          (top.value + height / 2)
          / sqrt(left.value - randomWidth, top.value + height / 2),
        ),
      ]
    }
    if (random === 1) {
      return [
        symbolX
        * Math.abs(
          (randomWidth - left.value)
          / sqrt(randomWidth - left.value, h - top.value + height / 2),
        ),
        symbolY
        * Math.abs(
          (h - top.value + height / 2)
          / sqrt(left.value - randomWidth, h - top.value + height / 2),
        ),
      ]
    }
    if (random === 2) {
      return [
        symbolX
        * Math.abs(
          (left.value + width / 2)
          / sqrt(left.value + width / 2, top.value - randomHeight),
        ),
        symbolY
        * Math.abs(
          (top.value - randomHeight)
          / sqrt(left.value + width / 2, top.value - randomHeight),
        ),
      ]
    }
    return [
      symbolX
      * Math.abs(
        (w - left.value + width / 2)
        / sqrt(w - left.value + width / 2, top.value - randomHeight),
      ),
      symbolY
      * Math.abs(
        (top.value - randomHeight)
        / sqrt(w - left.value + width / 2, top.value - randomHeight),
      ),
    ]
  }
  function gameover() {
    if (!status.value)
      return
    if (collisionDetection(div, '.target')) {
      blood.value -= 20
      removeElement(div)

      // 触发受击特效
      isHit.value = true
      setTimeout(() => {
        isHit.value = false
      }, 300) // 特效持续300毫秒

      // 修改游戏结束处理
      if (blood.value <= 0) {
        status.value = false
        clearInterval(timer)
        setTimeout(() => {
          // 保存分数并检查是否为最高分
          isHighScore.value = saveScore(times.value)
          // 显示游戏结束对话框
          gameState.value = 'gameover'
        })
      }
    }
  }
}

function resetGame() {
  times.value = 0
  blood.value = 100
  speed.value = 8
  status.value = true

  // 清除所有球体
  const blocks = document.querySelectorAll('.blocks')
  blocks.forEach(block => removeElement(block as HTMLElement))

  // 重置其他状态
  start = false
}

function startGame() {
  resetGame()
  gameState.value = 'game'
}

const bloodColor = computed(() => {
  if (blood.value >= 80)
    return 'bg-green'
  if (blood.value >= 60)
    return 'bg-yellow'
  return 'bg-red'
})

function saveScore(time: number) {
  try {
    // 读取现有排名
    const savedRankings = localStorage.getItem('reaction-quiz-rankings')
    let rankingsList: any[] = savedRankings ? JSON.parse(savedRankings) : []

    // 添加新成绩
    const newScore = {
      name: playerName.value || '匿名玩家',
      time,
      date: new Date().toLocaleDateString(),
      difficulty: difficulty.value, // 添加难度信息
    }

    // 保存用户名供下次使用
    if (playerName.value) {
      localStorage.setItem('reaction-quiz-username', playerName.value)
    }

    rankingsList.push(newScore)

    // 检查是否为当前难度下的最高分
    const sameTypeRankings = rankingsList.filter(r => r.difficulty === difficulty.value)
    const isNewHighScore = sameTypeRankings.length === 1
      || time > Math.max(...sameTypeRankings.filter(r => r !== newScore).map(r => r.time))

    // 按时间排序并只保留每个难度的前10名
    const simpleRankings = rankingsList
      .filter(r => r.difficulty === 'simple')
      .sort((a, b) => b.time - a.time)
      .slice(0, 10)

    const difficultRankings = rankingsList
      .filter(r => r.difficulty === 'difficult')
      .sort((a, b) => b.time - a.time)
      .slice(0, 10)

    // 合并两个难度的排名
    rankingsList = [...simpleRankings, ...difficultRankings]

    // 保存回本地存储
    localStorage.setItem('reaction-quiz-rankings', JSON.stringify(rankingsList))

    // 更新当前显示的排名
    rankings.value = rankingsList

    return isNewHighScore // 返回是否为新最高分
  }
  catch (error) {
    console.error('Error saving score:', error)
    return false
  }
}

function loadRankings() {
  try {
    const savedRankings = localStorage.getItem('reaction-quiz-rankings')
    if (savedRankings) {
      rankings.value = JSON.parse(savedRankings)
    }
  }
  catch (error) {
    console.error('Error loading rankings:', error)
  }
}

// 添加触摸事件支持
function setupTouchEvents() {
  // 触摸开始时记录位置并设置状态
  useEventListener('main', 'touchstart', (e) => {
    // 防止页面滚动
    if (!target.value)
      return
    const touch = e.touches[0]
    top.value = touch.clientY - target.value.offsetWidth / 2
    left.value = touch.clientX - target.value.offsetHeight / 2

    // 如果游戏还没开始，则开始游戏
    if (!start) {
      startGameLogic()
    }
  })

  // 触摸移动时更新位置
  useEventListener('main', 'touchmove', (e) => {
    // 防止页面滚动
    if (!target.value)
      return
    const touch = e.touches[0]
    top.value = touch.clientY - target.value.offsetWidth / 2
    left.value = touch.clientX - target.value.offsetHeight / 2
  })
}

// 抽取游戏开始逻辑为单独函数，以便在鼠标和触摸事件中重用
function startGameLogic() {
  start = true
  over = useRaf(() => {
    const len = (findElement('.blocks', true) as NodeListOf<Element>).length
    if (len <= 10) {
      // Generate more balls as time increases
      const objectsToGenerate = Math.min(Math.floor(times.value / 15) + 1, 5)
      for (let i = 0; i < objectsToGenerate; i++)
        generateObject()
    }
  }, {
    delta: 2000,
  })
  timer = setInterval(() => {
    times.value++

    // 每秒回复1%血量，但不超过100%
    if (blood.value < 100)
      blood.value += 1
  }, 1000)
}
</script>

<template>
  <main class="game-container" font-sans h-full overflow-hidden relative>
    <!-- 开始界面 -->
    <div v-if="gameState === 'start'" class="start-screen">
      <div class="start-content">
        <h1 class="game-title">
          反应力测试
        </h1>
        <p class="game-subtitle">
          测试你的反应速度！避开所有的球体！ <br>
          <span v-if="isMobile" class="device-tip">按住屏幕并移动以控制角色</span>
          <span v-else class="device-tip">使用鼠标移动来控制角色</span>
        </p>
        <div class="player-input">
          <input v-model="playerName" type="text" placeholder="输入你的名字" class="name-input">
        </div>
        <!-- 添加难度选择 -->
        <div class="difficulty-selection">
          <p class="difficulty-label">
            选择难度:
          </p>
          <div class="difficulty-options">
            <button
              :class="{ selected: difficulty === 'simple' }" class="difficulty-button"
              @click="difficulty = 'simple'"
            >
              简单
            </button>
            <button
              :class="{ selected: difficulty === 'difficult' }" class="difficulty-button"
              @click="difficulty = 'difficult'"
            >
              困难
            </button>
          </div>
        </div>
        <div class="action-buttons">
          <button class="start-button" @click="startGame">
            开始游戏
          </button>
          <button class="ranking-button" @click="loadRankings(); gameState = 'ranking'">
            历史排名
          </button>
        </div>
      </div>
    </div>
    <!-- 排行榜界面 -->
    <div v-else-if="gameState === 'ranking'" class="ranking-screen">
      <div class="ranking-content">
        <h2 class="ranking-title">
          历史最佳成绩
        </h2>
        <!-- 添加难度选择标签页 -->
        <div class="ranking-tabs">
          <button
            :class="{ active: currentRankingTab === 'simple' }" class="ranking-tab"
            @click="currentRankingTab = 'simple'"
          >
            简单模式
          </button>
          <button
            :class="{ active: currentRankingTab === 'difficult' }" class="ranking-tab"
            @click="currentRankingTab = 'difficult'"
          >
            困难模式
          </button>
        </div>
        <div class="ranking-list">
          <table class="ranking-table">
            <thead>
              <tr>
                <th>排名</th>
                <th>玩家</th>
                <th>坚持时间</th>
                <th>日期</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(rank, index) in filteredRankings" :key="index">
                <td>{{ index + 1 }}</td>
                <td>{{ rank.name }}</td>
                <td>{{ rank.time }}秒</td>
                <td>{{ rank.date }}</td>
              </tr>
              <tr v-if="filteredRankings.length === 0">
                <td colspan="4">
                  暂无记录
                </td>
              </tr>
            </tbody>
          </table>
        </div>
        <button class="back-button" @click="gameState = 'start'">
          返回主菜单
        </button>
      </div>
    </div>
    <!-- 游戏结束界面 -->
    <div v-else-if="gameState === 'gameover'" class="gameover-screen">
      <div class="gameover-content" :class="{ 'high-score': isHighScore }">
        <h2 class="gameover-title">
          游戏结束
        </h2>
        <p class="gameover-score">
          你坚持了 <span>{{ times }}</span> 秒!
        </p>
        <!-- 当破纪录时显示 -->
        <div v-if="isHighScore" class="high-score-banner">
          <div class="trophy">
            🏆
          </div> 新纪录！
        </div>
        <div class="action-buttons">
          <button class="restart-button" @click="startGame">
            再来一次
          </button>
          <button class="menu-button" @click="gameState = 'start'">
            回到主菜单
          </button>
        </div>
      </div>
    </div>
    <!-- 游戏界面 - 只有在gameState为'game'时显示 -->
    <template v-else-if="gameState === 'game'">
      <!-- 顶部游戏信息面板 -->
      <div class="game-header" p="x-4 y-3" flex="~ col gap-2" w-full>
        <!-- 血量条 -->
        <div class="health-bar-container" rounded-full overflow-hidden border="2 solid" border-opacity-30>
          <div
            :class="[bloodColor]" :style="{ width: `${blood <= 0 ? 0 : blood}%` }" class="health-bar" h-6
            transition-all duration-300 flex items-center
          >
            <div class="health-text" font-bold text-white px-3 text-shadow="1px 1px 2px rgba(0,0,0,0.5)">
              {{ blood }}%
            </div>
          </div>
        </div>
        <!-- 时间和信息 -->
        <div flex="~ gap-4" items-center justify-between>
          <div class="time-display" bg="gray-100 dark:gray-800" p="x-4 y-2" rounded-lg text="xl" font-bold shadow="sm">
            <span class="time-value">{{ times }}</span>
            <span class="time-unit" text="sm" opacity-70>秒</span>
          </div>
          <div class="game-info" flex="~ gap-2">
            <div v-if="times > 30" class="level-indicator" text="sm" bg="orange-500" text-white px-3 py-1 rounded-full>
              难度提升中
            </div>
            <div
              class="difficulty-indicator" text="sm"
              :class="difficulty === 'difficult' ? 'bg-red-500' : 'bg-green-500'" text-white px-3 py-1 rounded-full
            >
              {{
                difficulty === 'difficult' ? '困难' : '简单' }}
            </div>
          </div>
        </div>
      </div>
      <div v-if="isHit" class="screen-hit-effect" />
      <!-- 游戏目标/鼠标指针 -->
      <div
        ref="target" class="target" :class="{ 'hit-effect': isHit }" absolute
        :style="{ top: `${top}px`, left: `${left}px` }"
      >
        <div class="target-inner" animate-pulse>
          <svg width="40" height="40" viewBox="0 0 256 256" class="target-svg">
            <path
              fill="currentColor"
              d="m214.8 136.9l-39.2-50.4a5.2 5.2 0 0 0-1-1.1A31.5 31.5 0 0 0 152 76a40 40 0 1 0-48 0a31.5 31.5 0 0 0-22.6 9.4a5.2 5.2 0 0 0-1 1.1l-39.2 50.4a24 24 0 0 0 33.9 33.9l3.8-2.9L63 218.1a23.5 23.5 0 0 0-.4 17.5a24 24 0 0 0 43.9 2.7l21.5-33.8l21.5 33.8a23.9 23.9 0 0 0 13.2 11.6a23.3 23.3 0 0 0 8.2 1.5a24 24 0 0 0 22.1-33.3l-15.9-50.2l3.8 2.9a24 24 0 0 0 33.9-33.9ZM128 28a16 16 0 1 1-16 16a16 16 0 0 1 16-16Zm68.1 124.3l-34.7-27.1a12 12 0 0 0-18.8 13.1l27.7 87.6l.6 1.5a10 10 0 0 0-.8-1.4l-32-50.4a12 12 0 0 0-20.2 0l-32 50.4a10 10 0 0 0-.8 1.4l.6-1.5l27.7-87.6a11.9 11.9 0 0 0-4.6-13.4a11.3 11.3 0 0 0-6.8-2.2a12.7 12.7 0 0 0-7.4 2.5l-34.7 27.1l-1.2 1l1-1.2l39.1-50.2a8.1 8.1 0 0 1 5.2-1.9h48a8.1 8.1 0 0 1 5.2 1.9l39.1 50.2l1 1.2Z"
            />
          </svg>
        </div>
        <div class="target-ring" />
      </div>
    </template>
  </main>
</template>

<style scoped>
  /* 基础样式保持不变 */

  /* 优化目标指针动画 */
  .target {
    z-index: 100;
    pointer-events: none;
    will-change: transform;
    /* 启用GPU加速 */
    transform: translateZ(0);
    /* 强制GPU渲染 */
  }

  .target-inner {
    position: relative;
    transform: scale(1);
    transition: transform 0.2s cubic-bezier(0.34, 1.56, 0.64, 1);
    /* 改用弹性曲线 */
    color: #3b82f6;
    filter: drop-shadow(0 0 8px rgba(59, 130, 246, 0.5));
  }

  :global(.dark) .target-inner {
    color: #60a5fa;
    filter: drop-shadow(0 0 8px rgba(96, 165, 250, 0.7));
  }

  .target-ring {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    width: 60px;
    height: 60px;
    border-radius: 50%;
    border: 2px solid #3b82f6;
    opacity: 0.7;
    box-shadow: 0 0 15px rgba(59, 130, 246, 0.5);
    animation: pulse 2s cubic-bezier(0.4, 0, 0.6, 1) infinite;
    /* 更平滑的曲线 */
    will-change: transform, opacity;
    /* 指定变化属性提高性能 */
  }

  @keyframes pulse {
    0% {
      transform: translate(-50%, -50%) scale(0.8);
      opacity: 0.8;
    }

    50% {
      transform: translate(-50%, -50%) scale(1.1);
      opacity: 0.2;
    }

    100% {
      transform: translate(-50%, -50%) scale(0.8);
      opacity: 0.8;
    }
  }

  /* 优化血量条动画 */
  .health-bar {
    position: relative;
    transition: width 0.4s cubic-bezier(0.25, 1, 0.5, 1);
    /* 更平滑的过渡 */
    will-change: width;
    /* 提升性能 */
  }

  /* 优化游戏对象 */
  .blocks {
    filter: drop-shadow(0 4px 3px rgba(0, 0, 0, 0.2));
    transition: transform 0.15s cubic-bezier(0.34, 1.56, 0.64, 1);
    /* 更快更有弹性 */
    will-change: transform;
    /* 硬件加速 */
    backface-visibility: hidden;
    /* 减少闪烁 */
  }

  .blocks:hover {
    transform: scale(1.1);
  }

  /* 优化受击效果 */
  .hit-effect {
    animation: hitAnimation 0.3s cubic-bezier(0.22, 1, 0.36, 1);
    /* 使用更有弹性的曲线 */
  }

  .hit-effect .target-inner {
    color: #ef4444 !important;
    filter: drop-shadow(0 0 12px rgba(239, 68, 68, 0.8)) !important;
    transition: color 0.1s ease, filter 0.1s ease;
    /* 添加颜色过渡 */
  }

  .hit-effect .target-ring {
    border-color: #ef4444 !important;
    box-shadow: 0 0 20px rgba(239, 68, 68, 0.8) !important;
    transition: border-color 0.1s ease, box-shadow 0.1s ease;
    /* 添加过渡 */
  }

  @keyframes hitAnimation {
    0% {
      transform: scale(1);
    }

    40% {
      transform: scale(1.25);
      /* 略微增大缩放比例 */
    }

    70% {
      transform: scale(0.95);
      /* 添加反弹效果 */
    }

    100% {
      transform: scale(1);
    }
  }

  /* 优化屏幕受击效果 */
  .screen-hit-effect {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-color: rgba(239, 68, 68, 0.15);
    pointer-events: none;
    z-index: 50;
    animation: fadeOut 0.3s cubic-bezier(0.4, 0, 0.2, 1) forwards;
    /* 平滑过渡曲线 */
    will-change: opacity;
    /* 提升淡出性能 */
  }

  @keyframes fadeOut {
    from {
      opacity: 1;
    }

    to {
      opacity: 0;
    }
  }

  /* 添加难度提示动画 */
  .level-indicator {
    animation: pulseAttention 2s ease infinite;
  }

  @keyframes pulseAttention {

    0%,
    100% {
      transform: scale(1);
      opacity: 1;
    }

    50% {
      transform: scale(1.05);
      opacity: 0.9;
    }
  }

  /* 优化移动性能 */
  .game-container {
    background: radial-gradient(circle, rgba(240, 240, 245, 1) 0%, rgba(220, 225, 235, 1) 100%);
    color: #334155;
    transform: translateZ(0);
    /* 强制整个容器使用GPU */
  }

  /* 开始界面样式 */
  .start-screen,
  .ranking-screen,
  .gameover-screen {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    display: flex;
    justify-content: center;
    align-items: center;
    background: radial-gradient(circle, rgba(30, 41, 59, 0.9) 0%, rgba(15, 23, 42, 0.95) 100%);
    z-index: 1000;
  }

  .start-content,
  .ranking-content,
  .gameover-content {
    background-color: rgba(255, 255, 255, 0.95);
    border-radius: 16px;
    padding: 2rem;
    width: 90%;
    max-width: 500px;
    text-align: center;
    box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.25);
  }

  .game-title {
    font-size: 2.5rem;
    font-weight: 800;
    margin-bottom: 1rem;
    color: #3b82f6;
    text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.1);
  }

  .game-subtitle {
    color: #4b5563;
    margin-bottom: 2rem;
    font-size: 1.1rem;
  }

  .player-input {
    margin-bottom: 2rem;
  }

  .name-input {
    padding: 0.75rem 1rem;
    border: 2px solid #cbd5e1;
    border-radius: 8px;
    width: 100%;
    font-size: 1rem;
    transition: border 0.2s ease;
  }

  .name-input:focus {
    border-color: #3b82f6;
    outline: none;
  }

  .difficulty-selection {
    margin-bottom: 2rem;
  }

  .difficulty-label {
    margin-bottom: 0.5rem;
    color: #4b5563;
    font-weight: 500;
  }

  .difficulty-options {
    display: flex;
    justify-content: center;
    gap: 1rem;
  }

  .difficulty-button {
    padding: 0.5rem 1.5rem;
    border-radius: 8px;
    font-weight: 600;
    background-color: #e5e7eb;
    color: #4b5563;
    border: 2px solid transparent;
    transition: all 0.2s ease;
    cursor: pointer;
  }

  .difficulty-button:hover {
    background-color: #d1d5db;
    transform: translateY(-1px);
  }

  .difficulty-button.selected {
    background-color: #3b82f6;
    color: white;
    border-color: #2563eb;
  }

  .action-buttons {
    display: flex;
    gap: 1rem;
    justify-content: center;
  }

  .start-button,
  .ranking-button,
  .back-button,
  .restart-button,
  .menu-button {
    padding: 0.75rem 1.5rem;
    border-radius: 8px;
    font-weight: 600;
    font-size: 1rem;
    cursor: pointer;
    transition: all 0.2s ease;
    border: none;
  }

  .start-button {
    background-color: #3b82f6;
    color: white;
  }

  .start-button:hover {
    background-color: #2563eb;
    transform: translateY(-2px);
  }

  .ranking-button,
  .back-button,
  .menu-button {
    background-color: #e5e7eb;
    color: #4b5563;
  }

  .ranking-button:hover,
  .back-button:hover,
  .menu-button:hover {
    background-color: #d1d5db;
    transform: translateY(-2px);
  }

  .restart-button {
    background-color: #10b981;
    color: white;
  }

  .restart-button:hover {
    background-color: #059669;
    transform: translateY(-2px);
  }

  /* 排行榜样式 */
  .ranking-title {
    color: #3b82f6;
    font-size: 1.8rem;
    margin-bottom: 1.5rem;
  }

  .ranking-tabs {
    display: flex;
    justify-content: center;
    gap: 1rem;
    margin-bottom: 1.5rem;
  }

  .ranking-tab {
    padding: 0.5rem 1.5rem;
    border-radius: 8px;
    font-weight: 600;
    background-color: #e5e7eb;
    color: #4b5563;
    border: 2px solid transparent;
    transition: all 0.2s ease;
    cursor: pointer;
  }

  .ranking-tab:hover {
    background-color: #d1d5db;
    transform: translateY(-1px);
  }

  .ranking-tab.active {
    background-color: #3b82f6;
    color: white;
    border-color: #2563eb;
  }

  .ranking-list {
    margin-bottom: 1.5rem;
    max-height: 300px;
    overflow-y: auto;
  }

  .ranking-table {
    width: 100%;
    border-collapse: collapse;
  }

  .ranking-table th,
  .ranking-table td {
    padding: 0.75rem;
    text-align: center;
    border-bottom: 1px solid #e5e7eb;
  }

  .ranking-table th {
    background-color: #f3f4f6;
    font-weight: 600;
    color: #4b5563;
  }

  .ranking-table tr:nth-child(even) {
    background-color: #f9fafb;
  }

  .ranking-table tr:hover {
    background-color: #f3f4f6;
  }

  /* 游戏结束样式 */
  .gameover-title {
    color: #ef4444;
    font-size: 2rem;
    margin-bottom: 1rem;
  }

  .gameover-score {
    font-size: 1.25rem;
    margin-bottom: 2rem;
  }

  .gameover-score span {
    color: #3b82f6;
    font-weight: 700;
    font-size: 1.5rem;
  }

  .high-score-banner {
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 1.5rem;
    font-weight: bold;
    color: #10b981;
    margin-bottom: 1rem;
  }

  .high-score-banner .trophy {
    font-size: 2rem;
    margin-right: 0.5rem;
  }

  .device-tip {
    display: block;
    font-size: 0.9rem;
    margin-top: 0.5rem;
    color: #6b7280;
    font-style: italic;
  }
</style>
