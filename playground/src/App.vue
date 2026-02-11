<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'

type GameState = 'menu' | 'ranking' | 'playing' | 'paused' | 'gameover'
type Difficulty = 'casual' | 'pro'
type EnemyKind = 'meteor' | 'dart' | 'tank'
type PickupKind = 'focus' | 'heal'
type AudioStyle = 'arcade' | 'minimal'

interface DifficultyConfig {
  spawnBase: number
  speedMultiplier: number
  damageMultiplier: number
  nearMissWindow: number
  focusDrain: number
  stageRamp: number
}

interface Enemy {
  id: number
  x: number
  y: number
  vx: number
  vy: number
  radius: number
  damage: number
  rotation: number
  spin: number
  color: string
  kind: EnemyKind
  nearMissed: boolean
  value: number
}

interface Pickup {
  id: number
  x: number
  y: number
  vx: number
  vy: number
  radius: number
  value: number
  kind: PickupKind
  rotation: number
  spin: number
}

interface Floater {
  id: number
  x: number
  y: number
  text: string
  color: string
  ttl: number
  vy: number
  size: number
}

interface TrailPoint {
  x: number
  y: number
  alpha: number
}

interface Star {
  x: number
  y: number
  size: number
  twinkle: number
  speed: number
}

interface RankingEntry {
  name: string
  score: number
  time: number
  stage: number
  nearMiss: number
  pickup: number
  difficulty: Difficulty
  date: string
}

interface ToneOptions {
  from: number
  to?: number
  duration: number
  type?: OscillatorType
  volume?: number
  at?: number
}

type WebkitAudioWindow = Window & { webkitAudioContext?: typeof AudioContext }

interface AudioStyleConfig {
  masterVolume: number
  toneScale: number
  durationScale: number
  hapticsScale: number
}

const difficultyConfig: Record<Difficulty, DifficultyConfig> = {
  casual: {
    spawnBase: 0.9,
    speedMultiplier: 0.95,
    damageMultiplier: 0.9,
    nearMissWindow: 20,
    focusDrain: 26,
    stageRamp: 0.035,
  },
  pro: {
    spawnBase: 0.78,
    speedMultiplier: 1.12,
    damageMultiplier: 1.2,
    nearMissWindow: 16,
    focusDrain: 34,
    stageRamp: 0.05,
  },
}

const STORAGE_KEY = 'reaction-arena-rankings-v2'
const NAME_KEY = 'reaction-arena-player-v2'
const FEEDBACK_KEY = 'reaction-arena-feedback-v1'
const LEADERBOARD_LIMIT = 12

const audioStyleConfig: Record<AudioStyle, AudioStyleConfig> = {
  arcade: {
    masterVolume: 0.2,
    toneScale: 1,
    durationScale: 1,
    hapticsScale: 1,
  },
  minimal: {
    masterVolume: 0.12,
    toneScale: 0.68,
    durationScale: 0.76,
    hapticsScale: 0.72,
  },
}

const gameState = ref<GameState>('menu')
const difficulty = ref<Difficulty>('casual')
const rankingTab = ref<Difficulty>('casual')
const playerName = ref('')
const rankings = ref<RankingEntry[]>([])
const isNewRecord = ref(false)
const isTouchDevice = ref(false)
const soundEnabled = ref(true)
const hapticsEnabled = ref(true)
const audioStyle = ref<AudioStyle>('arcade')

const score = ref(0)
const elapsed = ref(0)
const stage = ref(1)
const health = ref(100)
const focus = ref(58)
const nearMissCount = ref(0)
const pickupCount = ref(0)
const hitFlash = ref(0)

const slowMoRequested = ref(false)
const slowMoActive = ref(false)

const latestScore = ref<RankingEntry | null>(null)

const canvasRef = ref<HTMLCanvasElement>()
let ctx: CanvasRenderingContext2D | null = null
let arenaWidth = 1280
let arenaHeight = 720

let rafId = 0
let lastTick = 0
let spawnClock = 0
let pickupClock = 0
let idSeed = 0
let activeConfig: DifficultyConfig = difficultyConfig.casual
let audioContext: AudioContext | null = null
let masterGain: GainNode | null = null
let lastNearMissSfxAt = 0
let lastVibrateAt = 0

const enemies: Enemy[] = []
const pickups: Pickup[] = []
const floaters: Floater[] = []
const stars: Star[] = []

const player = {
  x: 0,
  y: 0,
  targetX: 0,
  targetY: 0,
  radius: 17,
  trail: [] as TrailPoint[],
}

const hpPercent = computed(() => clamp(health.value, 0, 100))
const focusPercent = computed(() => clamp(focus.value, 0, 100))
const scoreDisplay = computed(() => Math.round(score.value))
const timeDisplay = computed(() => elapsed.value.toFixed(1))

const leaderboardRows = computed(() => {
  return rankings.value
    .filter(item => item.difficulty === rankingTab.value)
    .sort((a, b) => b.score - a.score)
    .slice(0, LEADERBOARD_LIMIT)
})

const bestByDifficulty = computed(() => {
  const items = rankings.value.filter(item => item.difficulty === difficulty.value)
  if (!items.length)
    return null
  return items.sort((a, b) => b.score - a.score)[0]
})

const stageLabel = computed(() => {
  if (stage.value >= 10)
    return '超限区域'
  if (stage.value >= 7)
    return '高压区域'
  if (stage.value >= 4)
    return '危险区域'
  return '校准区域'
})

const audioStyleLabel = computed(() => (audioStyle.value === 'arcade' ? '爽快声场' : '克制声场'))

const feedbackStatusText = computed(() => {
  const soundText = soundEnabled.value ? '音效开' : '音效关'
  const hapticsText = hapticsEnabled.value ? '震动开' : '震动关'
  return `${soundText} · ${hapticsText} · ${audioStyleLabel.value}`
})

function clamp(value: number, min: number, max: number) {
  return Math.min(max, Math.max(min, value))
}

function randomBetween(min: number, max: number) {
  return min + Math.random() * (max - min)
}

function distance(x1: number, y1: number, x2: number, y2: number) {
  const dx = x1 - x2
  const dy = y1 - y2
  return Math.sqrt(dx * dx + dy * dy)
}

function getAudioProfile() {
  return audioStyleConfig[audioStyle.value]
}

function ensureAudioReady() {
  const AudioCtor = window.AudioContext || (window as WebkitAudioWindow).webkitAudioContext
  if (!AudioCtor)
    return false

  if (!audioContext) {
    audioContext = new AudioCtor()
    masterGain = audioContext.createGain()
    masterGain.gain.value = 0.0001
    masterGain.connect(audioContext.destination)
  }

  if (audioContext.state === 'suspended')
    audioContext.resume().catch(() => {})

  applyAudioMix(true)

  return !!masterGain
}

function applyAudioMix(immediate = false) {
  if (!masterGain || !audioContext)
    return

  const profile = getAudioProfile()
  const target = soundEnabled.value ? profile.masterVolume : 0.0001
  if (immediate) {
    masterGain.gain.setValueAtTime(target, audioContext.currentTime)
    return
  }

  masterGain.gain.cancelScheduledValues(audioContext.currentTime)
  masterGain.gain.setTargetAtTime(target, audioContext.currentTime, 0.05)
}

function playTone(options: ToneOptions) {
  if (!soundEnabled.value)
    return

  if (!ensureAudioReady() || !audioContext || !masterGain)
    return

  const profile = getAudioProfile()
  const now = audioContext.currentTime + (options.at || 0)
  const end = now + Math.max(0.04, options.duration * profile.durationScale)
  const from = Math.max(30, options.from)
  const to = Math.max(30, options.to ?? options.from)
  const volume = clamp((options.volume ?? 0.08) * profile.toneScale, 0.01, 0.35)

  const oscillator = audioContext.createOscillator()
  const gain = audioContext.createGain()
  const filter = audioContext.createBiquadFilter()
  filter.type = 'lowpass'
  filter.frequency.value = 2400

  oscillator.type = options.type ?? 'sine'
  oscillator.frequency.setValueAtTime(from, now)
  if (to !== from)
    oscillator.frequency.exponentialRampToValueAtTime(to, end)

  gain.gain.setValueAtTime(0.0001, now)
  gain.gain.exponentialRampToValueAtTime(volume, now + 0.01)
  gain.gain.exponentialRampToValueAtTime(0.0001, end)

  oscillator.connect(filter)
  filter.connect(gain)
  gain.connect(masterGain)

  oscillator.start(now)
  oscillator.stop(end + 0.02)
  oscillator.onended = () => {
    oscillator.disconnect()
    filter.disconnect()
    gain.disconnect()
  }
}

function vibrate(pattern: number | number[], cooldown = 0) {
  if (!hapticsEnabled.value || typeof navigator.vibrate !== 'function')
    return

  const now = performance.now()
  if (cooldown > 0 && now - lastVibrateAt < cooldown)
    return

  lastVibrateAt = now
  const scale = getAudioProfile().hapticsScale
  if (Array.isArray(pattern))
    navigator.vibrate(pattern.map(ms => Math.max(1, Math.round(ms * scale))))
  else
    navigator.vibrate(Math.max(1, Math.round(pattern * scale)))
}

function playStartSfx() {
  if (audioStyle.value === 'minimal') {
    playTone({ from: 300, to: 460, duration: 0.08, type: 'sine', volume: 0.08 })
    return
  }

  playTone({ from: 250, to: 390, duration: 0.08, type: 'triangle', volume: 0.08 })
  playTone({ from: 400, to: 570, duration: 0.1, type: 'triangle', volume: 0.08, at: 0.09 })
}

function playNearMissSfx() {
  const now = performance.now()
  if (now - lastNearMissSfxAt < 80)
    return

  lastNearMissSfxAt = now
  if (audioStyle.value === 'minimal') {
    playTone({ from: 600, to: 760, duration: 0.05, type: 'sine', volume: 0.05 })
    return
  }

  playTone({ from: 520, to: 780, duration: 0.07, type: 'triangle', volume: 0.06 })
}

function playDamageSfx(damage: number) {
  const intensity = clamp(damage / 22, 0.45, 1.2)
  if (audioStyle.value === 'minimal') {
    playTone({
      from: 180,
      to: 72,
      duration: 0.14,
      type: 'triangle',
      volume: 0.1 * intensity,
    })
    return
  }

  playTone({
    from: 190,
    to: 70,
    duration: 0.15,
    type: 'sawtooth',
    volume: 0.12 * intensity,
  })
  playTone({
    from: 95,
    to: 50,
    duration: 0.19,
    type: 'square',
    volume: 0.08 * intensity,
    at: 0.02,
  })
}

function playPickupSfx(kind: PickupKind) {
  if (audioStyle.value === 'minimal') {
    if (kind === 'focus')
      playTone({ from: 400, to: 590, duration: 0.07, type: 'sine', volume: 0.075 })
    else
      playTone({ from: 430, to: 610, duration: 0.08, type: 'sine', volume: 0.075 })
    return
  }

  if (kind === 'focus') {
    playTone({ from: 360, to: 540, duration: 0.08, type: 'triangle', volume: 0.08 })
    playTone({ from: 550, to: 760, duration: 0.1, type: 'triangle', volume: 0.08, at: 0.07 })
    return
  }

  playTone({ from: 430, to: 470, duration: 0.08, type: 'sine', volume: 0.08 })
  playTone({ from: 470, to: 620, duration: 0.1, type: 'sine', volume: 0.08, at: 0.08 })
}

function playSlowMoSfx(active: boolean) {
  if (audioStyle.value === 'minimal') {
    if (active)
      playTone({ from: 260, to: 350, duration: 0.08, type: 'sine', volume: 0.058 })
    else
      playTone({ from: 280, to: 190, duration: 0.08, type: 'sine', volume: 0.052 })
    return
  }

  if (active) {
    playTone({ from: 230, to: 310, duration: 0.09, type: 'sine', volume: 0.065 })
    playTone({ from: 320, to: 420, duration: 0.12, type: 'triangle', volume: 0.065, at: 0.06 })
    return
  }

  playTone({ from: 300, to: 170, duration: 0.1, type: 'sine', volume: 0.06 })
}

function playGameOverSfx() {
  if (audioStyle.value === 'minimal') {
    playTone({ from: 240, to: 110, duration: 0.18, type: 'triangle', volume: 0.08 })
    return
  }

  playTone({ from: 260, to: 170, duration: 0.14, type: 'triangle', volume: 0.085 })
  playTone({ from: 170, to: 90, duration: 0.24, type: 'sawtooth', volume: 0.09, at: 0.12 })
}

function playNewRecordSfx() {
  if (audioStyle.value === 'minimal') {
    playTone({ from: 460, to: 700, duration: 0.09, type: 'sine', volume: 0.088 })
    playTone({ from: 700, to: 980, duration: 0.12, type: 'sine', volume: 0.092, at: 0.1 })
    return
  }

  playTone({ from: 420, to: 620, duration: 0.09, type: 'triangle', volume: 0.09 })
  playTone({ from: 620, to: 840, duration: 0.1, type: 'triangle', volume: 0.09, at: 0.08 })
  playTone({ from: 760, to: 1100, duration: 0.14, type: 'triangle', volume: 0.1, at: 0.18 })
}

function releaseAudio() {
  if (!audioContext)
    return

  audioContext.close().catch(() => {})
  audioContext = null
  masterGain = null
}

function saveFeedbackSettings() {
  try {
    const value = {
      soundEnabled: soundEnabled.value,
      hapticsEnabled: hapticsEnabled.value,
      audioStyle: audioStyle.value,
    }
    localStorage.setItem(FEEDBACK_KEY, JSON.stringify(value))
  }
  catch (error) {
    console.error('保存反馈设置失败:', error)
  }
}

function loadFeedbackSettings() {
  try {
    const raw = localStorage.getItem(FEEDBACK_KEY)
    if (!raw)
      return

    const parsed = JSON.parse(raw) as Partial<{ soundEnabled: boolean, hapticsEnabled: boolean, audioStyle: AudioStyle }>
    if (typeof parsed.soundEnabled === 'boolean')
      soundEnabled.value = parsed.soundEnabled
    if (typeof parsed.hapticsEnabled === 'boolean')
      hapticsEnabled.value = parsed.hapticsEnabled
    if (parsed.audioStyle === 'arcade' || parsed.audioStyle === 'minimal')
      audioStyle.value = parsed.audioStyle
  }
  catch (error) {
    console.error('读取反馈设置失败:', error)
  }
}

function setSoundEnabled(value: boolean) {
  soundEnabled.value = value
  if (value) {
    ensureAudioReady()
    playTone({ from: 460, to: 620, duration: 0.08, type: 'triangle', volume: 0.06 })
  }
  applyAudioMix(true)
  saveFeedbackSettings()
}

function setHapticsEnabled(value: boolean) {
  hapticsEnabled.value = value
  if (value)
    vibrate(12, 10)
  saveFeedbackSettings()
}

function setAudioStyle(value: AudioStyle) {
  audioStyle.value = value
  applyAudioMix(true)
  saveFeedbackSettings()

  if (soundEnabled.value) {
    if (value === 'arcade')
      playTone({ from: 350, to: 600, duration: 0.09, type: 'triangle', volume: 0.075 })
    else
      playTone({ from: 320, to: 520, duration: 0.08, type: 'sine', volume: 0.058 })
  }
}

function clearRoundData() {
  enemies.length = 0
  pickups.length = 0
  floaters.length = 0
  player.trail.length = 0

  score.value = 0
  elapsed.value = 0
  stage.value = 1
  health.value = 100
  focus.value = 58
  nearMissCount.value = 0
  pickupCount.value = 0
  hitFlash.value = 0

  slowMoRequested.value = false
  slowMoActive.value = false

  spawnClock = 0.65
  pickupClock = 3.8
  idSeed = 0
}

function resetPlayerToCenter() {
  player.x = arenaWidth / 2
  player.y = arenaHeight / 2
  player.targetX = player.x
  player.targetY = player.y
}

function updatePointer(clientX: number, clientY: number) {
  const canvas = canvasRef.value
  if (!canvas)
    return

  const rect = canvas.getBoundingClientRect()
  const x = clientX - rect.left
  const y = clientY - rect.top

  player.targetX = clamp(x, player.radius, rect.width - player.radius)
  player.targetY = clamp(y, player.radius, rect.height - player.radius)
}

function onArenaPointerMove(event: PointerEvent) {
  updatePointer(event.clientX, event.clientY)
}

function onArenaPointerDown(event: PointerEvent) {
  if (soundEnabled.value)
    ensureAudioReady()
  updatePointer(event.clientX, event.clientY)
}

function onFocusPress() {
  if (gameState.value === 'playing') {
    ensureAudioReady()
    slowMoRequested.value = true
  }
}

function onFocusRelease() {
  slowMoRequested.value = false
}

function getSpawnInterval() {
  const chaos = Math.min(0.45, (stage.value - 1) * activeConfig.stageRamp)
  const next = activeConfig.spawnBase - chaos + randomBetween(0.06, 0.32)
  return clamp(next, 0.2, 1.2)
}

function pickEnemyKind(): EnemyKind {
  const roll = Math.random()
  const tankThreshold = stage.value >= 3 ? 0.82 : 0.93
  const dartThreshold = stage.value >= 5 ? 0.56 : 0.64

  if (roll > tankThreshold)
    return 'tank'
  if (roll > dartThreshold)
    return 'dart'
  return 'meteor'
}

function createSpawnPoint(margin = 90) {
  const side = Math.floor(Math.random() * 4)
  if (side === 0)
    return { x: randomBetween(-20, arenaWidth + 20), y: -margin }
  if (side === 1)
    return { x: arenaWidth + margin, y: randomBetween(-20, arenaHeight + 20) }
  if (side === 2)
    return { x: randomBetween(-20, arenaWidth + 20), y: arenaHeight + margin }
  return { x: -margin, y: randomBetween(-20, arenaHeight + 20) }
}

function spawnEnemy() {
  const start = createSpawnPoint()
  const kind = pickEnemyKind()

  const aimX = clamp(player.targetX + randomBetween(-70, 70), 0, arenaWidth)
  const aimY = clamp(player.targetY + randomBetween(-70, 70), 0, arenaHeight)

  const dx = aimX - start.x
  const dy = aimY - start.y
  const len = Math.max(1, Math.sqrt(dx * dx + dy * dy))

  const statMap = {
    meteor: { speed: 190, radius: 14, damage: 12, color: '#f97316', value: 16 },
    dart: { speed: 255, radius: 11, damage: 9, color: '#ef4444', value: 19 },
    tank: { speed: 145, radius: 20, damage: 19, color: '#fb7185', value: 22 },
  } as const

  const stat = statMap[kind]
  const stageBonus = 1 + (stage.value - 1) * 0.07
  const speed = (stat.speed + randomBetween(-14, 28)) * activeConfig.speedMultiplier * stageBonus

  enemies.push({
    id: ++idSeed,
    x: start.x,
    y: start.y,
    vx: (dx / len) * speed,
    vy: (dy / len) * speed,
    radius: stat.radius,
    damage: Math.round(stat.damage * activeConfig.damageMultiplier),
    rotation: randomBetween(0, Math.PI * 2),
    spin: randomBetween(-2.8, 2.8),
    color: stat.color,
    kind,
    nearMissed: false,
    value: stat.value,
  })
}

function spawnPickup() {
  const kind: PickupKind = health.value < 52 && Math.random() < 0.5 ? 'heal' : 'focus'
  const start = createSpawnPoint(60)
  const targetX = randomBetween(arenaWidth * 0.18, arenaWidth * 0.82)
  const targetY = randomBetween(arenaHeight * 0.16, arenaHeight * 0.84)
  const dx = targetX - start.x
  const dy = targetY - start.y
  const len = Math.max(1, Math.sqrt(dx * dx + dy * dy))

  const speed = randomBetween(85, 125)
  pickups.push({
    id: ++idSeed,
    x: start.x,
    y: start.y,
    vx: (dx / len) * speed,
    vy: (dy / len) * speed,
    radius: kind === 'heal' ? 14 : 12,
    value: kind === 'heal' ? 18 : 26,
    kind,
    rotation: randomBetween(0, Math.PI * 2),
    spin: randomBetween(-1.5, 1.5),
  })
}

function addFloater(text: string, x: number, y: number, color: string, size = 18) {
  floaters.push({
    id: ++idSeed,
    x,
    y,
    text,
    color,
    ttl: 1,
    vy: randomBetween(20, 36),
    size,
  })
}

function applyDamage(damage: number, sourceX: number, sourceY: number) {
  health.value = clamp(health.value - damage, 0, 100)
  focus.value = clamp(focus.value - 8, 0, 100)
  hitFlash.value = 1
  playDamageSfx(damage)
  vibrate([30, 24, 44], 120)
  addFloater(`-${damage}`, sourceX, sourceY, '#fda4af')

  if (health.value <= 0)
    finishGame()
}

function collectPickup(item: Pickup) {
  pickupCount.value++
  playPickupSfx(item.kind)
  vibrate(item.kind === 'heal' ? [18, 16, 14] : 16, 100)

  if (item.kind === 'focus') {
    focus.value = clamp(focus.value + item.value, 0, 100)
    score.value += 32
    addFloater('+聚焦', item.x, item.y, '#67e8f9')
    return
  }

  health.value = clamp(health.value + item.value, 0, 100)
  score.value += 24
  addFloater('+修复', item.x, item.y, '#86efac')
}

function updateSlowMotion(dt: number) {
  const wasActive = slowMoActive.value

  if (slowMoRequested.value && focus.value > 0) {
    slowMoActive.value = true
    focus.value = clamp(focus.value - activeConfig.focusDrain * dt, 0, 100)
    if (focus.value <= 0)
      slowMoRequested.value = false
    return
  }

  slowMoActive.value = false
  focus.value = clamp(focus.value + 4.8 * dt, 0, 100)

  if (!wasActive && slowMoActive.value) {
    playSlowMoSfx(true)
    vibrate(12, 120)
  }
  else if (wasActive && !slowMoActive.value) {
    playSlowMoSfx(false)
  }
}

function updatePlayer(dt: number) {
  const follow = 1 - Math.exp(-dt * 18)
  player.x += (player.targetX - player.x) * follow
  player.y += (player.targetY - player.y) * follow

  player.trail.unshift({
    x: player.x,
    y: player.y,
    alpha: 0.85,
  })

  if (player.trail.length > 14)
    player.trail.length = 14

  for (let i = player.trail.length - 1; i >= 0; i--) {
    player.trail[i].alpha -= dt * 2.7
    if (player.trail[i].alpha <= 0)
      player.trail.splice(i, 1)
  }
}

function updateEnemies(dt: number, worldScale: number) {
  for (let i = enemies.length - 1; i >= 0; i--) {
    const enemy = enemies[i]
    enemy.x += enemy.vx * dt * worldScale
    enemy.y += enemy.vy * dt * worldScale
    enemy.rotation += enemy.spin * dt * worldScale

    const hitDistance = enemy.radius + player.radius
    const dist = distance(enemy.x, enemy.y, player.x, player.y)

    if (dist <= hitDistance) {
      applyDamage(enemy.damage, enemy.x, enemy.y)
      enemies.splice(i, 1)
      continue
    }

    if (!enemy.nearMissed && dist <= hitDistance + activeConfig.nearMissWindow) {
      enemy.nearMissed = true
      nearMissCount.value++
      const bonus = 18 + stage.value * 4
      score.value += bonus
      focus.value = clamp(focus.value + 9, 0, 100)
      addFloater(`擦身 +${bonus}`, enemy.x, enemy.y, '#facc15', 16)
      playNearMissSfx()
      vibrate(8, 180)
    }

    const outOfBounds
      = enemy.x < -140
        || enemy.x > arenaWidth + 140
        || enemy.y < -140
        || enemy.y > arenaHeight + 140

    if (outOfBounds) {
      score.value += enemy.value
      enemies.splice(i, 1)
    }
  }
}

function updatePickups(dt: number, worldScale: number) {
  for (let i = pickups.length - 1; i >= 0; i--) {
    const item = pickups[i]
    item.x += item.vx * dt * worldScale
    item.y += item.vy * dt * worldScale
    item.rotation += item.spin * dt

    const dist = distance(item.x, item.y, player.x, player.y)
    if (dist <= item.radius + player.radius) {
      collectPickup(item)
      pickups.splice(i, 1)
      continue
    }

    const outOfBounds
      = item.x < -120
        || item.x > arenaWidth + 120
        || item.y < -120
        || item.y > arenaHeight + 120

    if (outOfBounds)
      pickups.splice(i, 1)
  }
}

function updateFloaters(dt: number) {
  for (let i = floaters.length - 1; i >= 0; i--) {
    const text = floaters[i]
    text.y -= text.vy * dt
    text.ttl -= dt * 1.1

    if (text.ttl <= 0)
      floaters.splice(i, 1)
  }
}

function updateGame(dt: number) {
  updateSlowMotion(dt)

  const worldScale = slowMoActive.value ? 0.42 : 1

  elapsed.value += dt
  stage.value = 1 + Math.floor(elapsed.value / 18)
  score.value += dt * (16 + stage.value * 2.8)

  spawnClock -= dt * worldScale
  while (spawnClock <= 0) {
    spawnEnemy()
    spawnClock += getSpawnInterval()
  }

  pickupClock -= dt * worldScale
  if (pickupClock <= 0) {
    spawnPickup()
    pickupClock = randomBetween(5.2, 8.4)
  }

  updatePlayer(dt)
  updateEnemies(dt, worldScale)
  updatePickups(dt, worldScale)
  updateFloaters(dt)

  hitFlash.value = Math.max(0, hitFlash.value - dt * 2.8)
}

function drawBackdrop(context: CanvasRenderingContext2D, time: number) {
  const gradient = context.createLinearGradient(0, 0, 0, arenaHeight)
  gradient.addColorStop(0, '#081421')
  gradient.addColorStop(0.55, '#0b2032')
  gradient.addColorStop(1, '#101926')

  context.fillStyle = gradient
  context.fillRect(0, 0, arenaWidth, arenaHeight)

  const glow = context.createRadialGradient(
    arenaWidth * 0.5,
    arenaHeight * 0.48,
    50,
    arenaWidth * 0.5,
    arenaHeight * 0.48,
    arenaWidth * 0.66,
  )
  glow.addColorStop(0, 'rgba(59, 130, 246, 0.22)')
  glow.addColorStop(1, 'rgba(16, 185, 129, 0.02)')
  context.fillStyle = glow
  context.fillRect(0, 0, arenaWidth, arenaHeight)

  for (const star of stars) {
    const pulse = 0.35 + 0.65 * Math.sin(time * star.speed + star.twinkle)
    context.fillStyle = `rgba(191, 219, 254, ${0.15 + pulse * 0.3})`
    context.beginPath()
    context.arc(star.x, star.y, star.size, 0, Math.PI * 2)
    context.fill()
  }

  const gridOffset = (time * 34) % 42
  context.strokeStyle = 'rgba(148, 163, 184, 0.11)'
  context.lineWidth = 1

  context.beginPath()
  for (let x = -40 + gridOffset; x <= arenaWidth + 40; x += 42) {
    context.moveTo(x, 0)
    context.lineTo(x, arenaHeight)
  }
  for (let y = -40 + gridOffset; y <= arenaHeight + 40; y += 42) {
    context.moveTo(0, y)
    context.lineTo(arenaWidth, y)
  }
  context.stroke()

  context.fillStyle = 'rgba(8, 15, 23, 0.48)'
  context.fillRect(0, arenaHeight - 120, arenaWidth, 120)
}

function drawEnemy(context: CanvasRenderingContext2D, enemy: Enemy, time: number) {
  context.save()
  context.translate(enemy.x, enemy.y)
  context.rotate(enemy.rotation)

  if (enemy.kind === 'meteor') {
    context.fillStyle = enemy.color
    context.strokeStyle = 'rgba(255, 255, 255, 0.45)'
    context.lineWidth = 1.4
    context.beginPath()
    for (let i = 0; i < 8; i++) {
      const angle = (Math.PI * 2 * i) / 8
      const wobble = Math.sin(time * 7 + i) * 2.2
      const radius = enemy.radius + (i % 2 ? 4 : 0) + wobble
      const x = Math.cos(angle) * radius
      const y = Math.sin(angle) * radius
      if (i === 0)
        context.moveTo(x, y)
      else
        context.lineTo(x, y)
    }
    context.closePath()
    context.fill()
    context.stroke()
  }

  if (enemy.kind === 'dart') {
    context.fillStyle = enemy.color
    context.beginPath()
    context.moveTo(enemy.radius + 6, 0)
    context.lineTo(-enemy.radius, enemy.radius * 0.75)
    context.lineTo(-enemy.radius, -enemy.radius * 0.75)
    context.closePath()
    context.fill()

    context.fillStyle = 'rgba(255, 255, 255, 0.45)'
    context.beginPath()
    context.moveTo(-enemy.radius - 3, 0)
    context.lineTo(-enemy.radius - 15, 4)
    context.lineTo(-enemy.radius - 15, -4)
    context.closePath()
    context.fill()
  }

  if (enemy.kind === 'tank') {
    context.fillStyle = enemy.color
    context.beginPath()
    context.arc(0, 0, enemy.radius, 0, Math.PI * 2)
    context.fill()

    context.strokeStyle = 'rgba(255, 255, 255, 0.65)'
    context.lineWidth = 3
    context.beginPath()
    context.arc(0, 0, enemy.radius - 5, 0, Math.PI * 2)
    context.stroke()

    context.fillStyle = 'rgba(255, 255, 255, 0.25)'
    context.beginPath()
    context.arc(-4, -4, enemy.radius * 0.35, 0, Math.PI * 2)
    context.fill()
  }

  context.restore()
}

function drawPickup(context: CanvasRenderingContext2D, item: Pickup, time: number) {
  context.save()
  context.translate(item.x, item.y)
  context.rotate(item.rotation)

  if (item.kind === 'focus') {
    const glow = context.createRadialGradient(0, 0, 1, 0, 0, item.radius + 10)
    glow.addColorStop(0, 'rgba(34, 211, 238, 0.9)')
    glow.addColorStop(1, 'rgba(34, 211, 238, 0)')
    context.fillStyle = glow
    context.beginPath()
    context.arc(0, 0, item.radius + 10, 0, Math.PI * 2)
    context.fill()

    context.strokeStyle = '#22d3ee'
    context.lineWidth = 2
    context.beginPath()
    context.arc(0, 0, item.radius, 0, Math.PI * 2)
    context.stroke()

    context.strokeStyle = 'rgba(186, 230, 253, 0.9)'
    context.lineWidth = 2
    context.beginPath()
    const wave = Math.sin(time * 5) * 0.4
    context.arc(0, 0, item.radius * (0.58 + wave), 0, Math.PI * 2)
    context.stroke()
  }

  if (item.kind === 'heal') {
    const glow = context.createRadialGradient(0, 0, 1, 0, 0, item.radius + 10)
    glow.addColorStop(0, 'rgba(74, 222, 128, 0.88)')
    glow.addColorStop(1, 'rgba(74, 222, 128, 0)')
    context.fillStyle = glow
    context.beginPath()
    context.arc(0, 0, item.radius + 10, 0, Math.PI * 2)
    context.fill()

    context.fillStyle = '#4ade80'
    context.beginPath()
    context.arc(0, 0, item.radius, 0, Math.PI * 2)
    context.fill()

    context.strokeStyle = '#ecfdf5'
    context.lineWidth = 3
    context.beginPath()
    context.moveTo(-item.radius * 0.45, 0)
    context.lineTo(item.radius * 0.45, 0)
    context.moveTo(0, -item.radius * 0.45)
    context.lineTo(0, item.radius * 0.45)
    context.stroke()
  }

  context.restore()
}

function drawPlayer(context: CanvasRenderingContext2D) {
  for (const trail of player.trail) {
    context.fillStyle = `rgba(125, 211, 252, ${trail.alpha * 0.3})`
    context.beginPath()
    context.arc(trail.x, trail.y, player.radius * (0.52 + trail.alpha * 0.45), 0, Math.PI * 2)
    context.fill()
  }

  const gradient = context.createRadialGradient(
    player.x - 4,
    player.y - 4,
    2,
    player.x,
    player.y,
    player.radius + 8,
  )
  gradient.addColorStop(0, '#e0f2fe')
  gradient.addColorStop(1, '#22d3ee')

  context.fillStyle = gradient
  context.beginPath()
  context.arc(player.x, player.y, player.radius, 0, Math.PI * 2)
  context.fill()

  context.strokeStyle = slowMoActive.value ? '#facc15' : '#bae6fd'
  context.lineWidth = 2.4
  context.beginPath()
  context.arc(player.x, player.y, player.radius + 7, 0, Math.PI * 2)
  context.stroke()

  context.strokeStyle = 'rgba(224, 242, 254, 0.9)'
  context.lineWidth = 1.6
  context.beginPath()
  context.moveTo(player.x - 11, player.y)
  context.lineTo(player.x + 11, player.y)
  context.moveTo(player.x, player.y - 11)
  context.lineTo(player.x, player.y + 11)
  context.stroke()
}

function drawFloaters(context: CanvasRenderingContext2D) {
  context.textAlign = 'center'
  context.textBaseline = 'middle'

  for (const text of floaters) {
    context.globalAlpha = clamp(text.ttl, 0, 1)
    context.fillStyle = text.color
    context.font = `700 ${text.size}px "Rajdhani", "Noto Sans SC", sans-serif`
    context.fillText(text.text, text.x, text.y)
  }

  context.globalAlpha = 1
}

function drawScene(time: number) {
  if (!ctx)
    return

  const context = ctx
  context.clearRect(0, 0, arenaWidth, arenaHeight)
  drawBackdrop(context, time)

  if (hitFlash.value > 0.01) {
    const shake = hitFlash.value * 5
    context.save()
    context.translate(randomBetween(-shake, shake), randomBetween(-shake, shake))
  }

  for (const item of pickups)
    drawPickup(context, item, time)

  for (const enemy of enemies)
    drawEnemy(context, enemy, time)

  drawPlayer(context)
  drawFloaters(context)

  if (hitFlash.value > 0.01)
    context.restore()

  if (hitFlash.value > 0.01) {
    context.fillStyle = `rgba(248, 113, 113, ${hitFlash.value * 0.22})`
    context.fillRect(0, 0, arenaWidth, arenaHeight)
  }
}

const loop = (now: number) => {
  if (gameState.value !== 'playing')
    return

  if (!lastTick)
    lastTick = now

  const dt = Math.min((now - lastTick) / 1000, 0.05)
  lastTick = now

  updateGame(dt)
  drawScene(now / 1000)

  rafId = window.requestAnimationFrame(loop)
}

function stopLoop() {
  if (!rafId)
    return

  window.cancelAnimationFrame(rafId)
  rafId = 0
}

function startLoop() {
  stopLoop()
  lastTick = performance.now()
  rafId = window.requestAnimationFrame(loop)
}

function persistPlayerName() {
  try {
    localStorage.setItem(NAME_KEY, playerName.value.trim())
  }
  catch (error) {
    console.error('保存名字失败:', error)
  }
}

function loadPlayerName() {
  try {
    playerName.value = localStorage.getItem(NAME_KEY) || ''
  }
  catch (error) {
    console.error('读取名字失败:', error)
  }
}

function loadRankings() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    if (!raw)
      return

    const parsed = JSON.parse(raw) as RankingEntry[]
    if (Array.isArray(parsed))
      rankings.value = parsed
  }
  catch (error) {
    console.error('读取排行榜失败:', error)
  }
}

function saveRankings() {
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(rankings.value))
  }
  catch (error) {
    console.error('写入排行榜失败:', error)
  }
}

function saveCurrentRun() {
  const entry: RankingEntry = {
    name: playerName.value.trim() || '匿名玩家',
    score: Math.round(score.value),
    time: Number(elapsed.value.toFixed(1)),
    stage: stage.value,
    nearMiss: nearMissCount.value,
    pickup: pickupCount.value,
    difficulty: difficulty.value,
    date: new Date().toLocaleString('zh-CN', {
      year: 'numeric',
      month: '2-digit',
      day: '2-digit',
      hour: '2-digit',
      minute: '2-digit',
      hour12: false,
    }),
  }

  const currentDifficultyScores = rankings.value
    .filter(item => item.difficulty === difficulty.value)
    .map(item => item.score)

  const previousBest = currentDifficultyScores.length ? Math.max(...currentDifficultyScores) : 0

  const next = [...rankings.value, entry]

  const casual = next
    .filter(item => item.difficulty === 'casual')
    .sort((a, b) => b.score - a.score)
    .slice(0, LEADERBOARD_LIMIT)

  const pro = next
    .filter(item => item.difficulty === 'pro')
    .sort((a, b) => b.score - a.score)
    .slice(0, LEADERBOARD_LIMIT)

  rankings.value = [...casual, ...pro]
  saveRankings()

  latestScore.value = entry
  isNewRecord.value = entry.score > previousBest
  if (isNewRecord.value) {
    playNewRecordSfx()
    vibrate([28, 22, 28, 22, 50], 280)
  }
}

function resizeCanvas() {
  const canvas = canvasRef.value
  if (!canvas)
    return

  const rect = canvas.getBoundingClientRect()
  if (!rect.width || !rect.height)
    return

  arenaWidth = rect.width
  arenaHeight = rect.height

  const dpr = Math.min(window.devicePixelRatio || 1, 2)
  canvas.width = Math.round(rect.width * dpr)
  canvas.height = Math.round(rect.height * dpr)

  ctx = canvas.getContext('2d')
  if (!ctx)
    return

  ctx.setTransform(dpr, 0, 0, dpr, 0, 0)

  stars.length = 0
  const starCount = Math.floor((arenaWidth * arenaHeight) / 24000)
  for (let i = 0; i < Math.max(36, starCount); i++) {
    stars.push({
      x: randomBetween(0, arenaWidth),
      y: randomBetween(0, arenaHeight),
      size: randomBetween(0.5, 1.8),
      twinkle: randomBetween(0, Math.PI * 2),
      speed: randomBetween(0.6, 1.6),
    })
  }

  if (gameState.value !== 'playing' && gameState.value !== 'paused')
    drawScene(performance.now() / 1000)
}

function startGame() {
  ensureAudioReady()
  playStartSfx()
  persistPlayerName()
  clearRoundData()
  activeConfig = difficultyConfig[difficulty.value]

  resetPlayerToCenter()

  gameState.value = 'playing'
  startLoop()
}

function finishGame() {
  if (gameState.value !== 'playing')
    return

  playGameOverSfx()
  vibrate([24, 28, 24], 220)
  stopLoop()
  slowMoRequested.value = false
  slowMoActive.value = false

  saveCurrentRun()
  gameState.value = 'gameover'
}

function pauseGame() {
  if (gameState.value !== 'playing')
    return

  stopLoop()
  slowMoRequested.value = false
  slowMoActive.value = false
  gameState.value = 'paused'
}

function resumeGame() {
  if (gameState.value !== 'paused')
    return

  gameState.value = 'playing'
  startLoop()
}

function toMenu() {
  stopLoop()
  slowMoRequested.value = false
  slowMoActive.value = false
  gameState.value = 'menu'
  drawScene(performance.now() / 1000)
}

function toRanking() {
  stopLoop()
  slowMoRequested.value = false
  slowMoActive.value = false
  loadRankings()
  gameState.value = 'ranking'
}

function handleKeyDown(event: KeyboardEvent) {
  if (event.code === 'ShiftLeft' || event.code === 'ShiftRight') {
    if (gameState.value === 'playing')
      slowMoRequested.value = true
    return
  }

  if (event.code === 'KeyP') {
    if (gameState.value === 'playing')
      pauseGame()
    else if (gameState.value === 'paused')
      resumeGame()
    return
  }

  if (event.code === 'Escape') {
    if (gameState.value === 'playing' || gameState.value === 'paused')
      toMenu()
  }
}

function handleKeyUp(event: KeyboardEvent) {
  if (event.code === 'ShiftLeft' || event.code === 'ShiftRight')
    slowMoRequested.value = false
}

onMounted(() => {
  isTouchDevice.value = window.matchMedia('(pointer: coarse)').matches

  loadFeedbackSettings()
  loadPlayerName()
  loadRankings()

  resizeCanvas()
  resetPlayerToCenter()
  drawScene(performance.now() / 1000)

  window.addEventListener('resize', resizeCanvas)
  window.addEventListener('keydown', handleKeyDown)
  window.addEventListener('keyup', handleKeyUp)
})

onBeforeUnmount(() => {
  stopLoop()
  releaseAudio()
  window.removeEventListener('resize', resizeCanvas)
  window.removeEventListener('keydown', handleKeyDown)
  window.removeEventListener('keyup', handleKeyUp)
})
</script>

<template>
  <main class="game-shell">
    <section
      class="arena"
      @pointermove="onArenaPointerMove"
      @pointerdown="onArenaPointerDown"
    >
      <canvas ref="canvasRef" class="arena-canvas" />

      <div v-if="gameState === 'playing' || gameState === 'paused'" class="hud">
        <div class="meter-card">
          <div class="meter-title">
            生命值
          </div>
          <div class="meter-track">
            <div class="meter-fill health-fill" :style="{ width: `${hpPercent}%` }" />
          </div>
          <div class="meter-value">
            {{ hpPercent.toFixed(0) }}%
          </div>
        </div>

        <div class="meter-card">
          <div class="meter-title">
            聚焦能量
          </div>
          <div class="meter-track">
            <div class="meter-fill focus-fill" :style="{ width: `${focusPercent}%` }" />
          </div>
          <div class="meter-value">
            {{ focusPercent.toFixed(0) }}%
          </div>
        </div>

        <div class="stats-row">
          <div class="stat-pill">
            <span class="stat-label">分数</span>
            <span class="stat-value">{{ scoreDisplay }}</span>
          </div>
          <div class="stat-pill">
            <span class="stat-label">时间</span>
            <span class="stat-value">{{ timeDisplay }}s</span>
          </div>
          <div class="stat-pill">
            <span class="stat-label">阶段</span>
            <span class="stat-value">{{ stage }} · {{ stageLabel }}</span>
          </div>
        </div>

        <p class="hud-tip">
          按住 Shift 进入时缓，P 暂停，Esc 返回菜单 · {{ feedbackStatusText }}
        </p>
      </div>

      <button
        v-if="isTouchDevice && (gameState === 'playing' || gameState === 'paused')"
        class="focus-button"
        :class="{ active: slowMoRequested && gameState === 'playing' }"
        @pointerdown.prevent="onFocusPress"
        @pointerup.prevent="onFocusRelease"
        @pointercancel.prevent="onFocusRelease"
        @pointerleave.prevent="onFocusRelease"
      >
        长按时缓
      </button>

      <div v-if="gameState === 'paused'" class="pause-layer">
        <div class="panel pause-panel">
          <p class="panel-subtitle">
            游戏已暂停
          </p>
          <h2>继续挑战</h2>
          <div class="button-row">
            <button class="primary-btn" @click="resumeGame">
              继续
            </button>
            <button class="ghost-btn" @click="toMenu">
              回到菜单
            </button>
          </div>
        </div>
      </div>

      <div v-if="gameState === 'menu'" class="overlay-layer">
        <div class="panel intro-panel">
          <p class="panel-subtitle">
            REACTION ARENA
          </p>
          <h1>极限反应场</h1>
          <p class="panel-description">
            不是只要活着就够了。擦身拿分、补给续命、在关键时刻开启时缓，才能打出高分。
          </p>

          <ul class="feature-list">
            <li>擦身飞行体可获得高分和聚焦能量</li>
            <li>青色道具恢复聚焦，绿色道具修复生命</li>
            <li>阶段越高，生成频率和速度越快</li>
            <li>受击、补给、时缓切换支持音效和震动反馈</li>
            <li>反馈风格可切换为爽快或克制，并自动保存</li>
          </ul>

          <div class="form-row">
            <label for="playerName" class="field-label">玩家名称</label>
            <input
              id="playerName"
              v-model="playerName"
              class="name-input"
              type="text"
              maxlength="18"
              placeholder="输入你的代号"
            >
          </div>

          <div class="difficulty-switch">
            <button
              class="difficulty-btn"
              :class="{ selected: difficulty === 'casual' }"
              @click="difficulty = 'casual'"
            >
              校准模式
            </button>
            <button
              class="difficulty-btn"
              :class="{ selected: difficulty === 'pro' }"
              @click="difficulty = 'pro'"
            >
              压力模式
            </button>
          </div>

          <div class="feedback-settings">
            <p class="feedback-title">
              反馈设置
            </p>
            <div class="feedback-toggle-row">
              <button
                class="toggle-chip"
                :class="{ on: soundEnabled }"
                @click="setSoundEnabled(!soundEnabled)"
              >
                音效 {{ soundEnabled ? '开' : '关' }}
              </button>
              <button
                class="toggle-chip"
                :class="{ on: hapticsEnabled }"
                @click="setHapticsEnabled(!hapticsEnabled)"
              >
                震动 {{ hapticsEnabled ? '开' : '关' }}
              </button>
            </div>

            <div class="feedback-style-row">
              <button
                class="style-chip"
                :class="{ selected: audioStyle === 'arcade' }"
                @click="setAudioStyle('arcade')"
              >
                爽快声场
              </button>
              <button
                class="style-chip"
                :class="{ selected: audioStyle === 'minimal' }"
                @click="setAudioStyle('minimal')"
              >
                克制声场
              </button>
            </div>
          </div>

          <p v-if="bestByDifficulty" class="best-tip">
            当前模式最高分：{{ bestByDifficulty.score }}（{{ bestByDifficulty.name }}）
          </p>

          <div class="button-row">
            <button class="primary-btn" @click="startGame">
              开始挑战
            </button>
            <button class="ghost-btn" @click="toRanking">
              历史排名
            </button>
          </div>
        </div>
      </div>

      <div v-if="gameState === 'ranking'" class="overlay-layer">
        <div class="panel ranking-panel">
          <p class="panel-subtitle">
            LEADERBOARD
          </p>
          <h2>历史成绩</h2>

          <div class="difficulty-switch">
            <button
              class="difficulty-btn"
              :class="{ selected: rankingTab === 'casual' }"
              @click="rankingTab = 'casual'"
            >
              校准模式
            </button>
            <button
              class="difficulty-btn"
              :class="{ selected: rankingTab === 'pro' }"
              @click="rankingTab = 'pro'"
            >
              压力模式
            </button>
          </div>

          <div class="table-wrap">
            <table>
              <thead>
                <tr>
                  <th>#</th>
                  <th>玩家</th>
                  <th>分数</th>
                  <th>时间</th>
                  <th>阶段</th>
                  <th>日期</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="(row, index) in leaderboardRows" :key="`${row.date}-${row.name}-${index}`">
                  <td>{{ index + 1 }}</td>
                  <td>{{ row.name }}</td>
                  <td>{{ row.score }}</td>
                  <td>{{ row.time }}s</td>
                  <td>{{ row.stage }}</td>
                  <td>{{ row.date }}</td>
                </tr>
                <tr v-if="leaderboardRows.length === 0">
                  <td colspan="6">
                    暂无记录
                  </td>
                </tr>
              </tbody>
            </table>
          </div>

          <div class="button-row">
            <button class="primary-btn" @click="startGame">
              开始挑战
            </button>
            <button class="ghost-btn" @click="toMenu">
              返回菜单
            </button>
          </div>
        </div>
      </div>

      <div v-if="gameState === 'gameover'" class="overlay-layer">
        <div class="panel gameover-panel" :class="{ highlight: isNewRecord }">
          <p class="panel-subtitle">
            RUN COMPLETE
          </p>
          <h2>{{ isNewRecord ? '新纪录达成' : '本局结束' }}</h2>

          <p class="summary-line">
            最终分数 <strong>{{ latestScore?.score || scoreDisplay }}</strong>
          </p>

          <div class="summary-grid">
            <div>
              <span>生存时间</span>
              <strong>{{ latestScore?.time || timeDisplay }}s</strong>
            </div>
            <div>
              <span>到达阶段</span>
              <strong>{{ latestScore?.stage || stage }}</strong>
            </div>
            <div>
              <span>擦身次数</span>
              <strong>{{ latestScore?.nearMiss || nearMissCount }}</strong>
            </div>
            <div>
              <span>补给收集</span>
              <strong>{{ latestScore?.pickup || pickupCount }}</strong>
            </div>
          </div>

          <div class="button-row">
            <button class="primary-btn" @click="startGame">
              再来一局
            </button>
            <button class="ghost-btn" @click="toRanking">
              查看排名
            </button>
          </div>
        </div>
      </div>
    </section>
  </main>
</template>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Rajdhani:wght@500;600;700&family=Noto+Sans+SC:wght@400;500;700;900&display=swap');

.game-shell {
  --bg-primary: #081421;
  --bg-secondary: #0f1f2f;
  --panel-bg: rgba(8, 18, 30, 0.78);
  --panel-border: rgba(186, 230, 253, 0.22);
  --text-main: #e0f2fe;
  --text-soft: rgba(186, 230, 253, 0.82);
  --accent: #22d3ee;
  --accent-strong: #06b6d4;
  --danger: #f97316;
  --focus: #facc15;

  width: 100%;
  height: 100%;
  overflow: hidden;
  background: radial-gradient(circle at 18% 16%, #163658 0%, var(--bg-primary) 44%, #050b11 100%);
  font-family: 'Noto Sans SC', 'Rajdhani', 'Avenir Next Condensed', 'PingFang SC', sans-serif;
  color: var(--text-main);
}

.arena {
  position: relative;
  width: 100%;
  height: 100%;
}

.arena-canvas {
  display: block;
  width: 100%;
  height: 100%;
  touch-action: none;
}

.hud {
  position: absolute;
  top: 1rem;
  left: 1rem;
  right: 1rem;
  z-index: 6;
  display: grid;
  gap: 0.8rem;
}

.meter-card {
  display: grid;
  grid-template-columns: auto 1fr auto;
  align-items: center;
  gap: 0.8rem;
  padding: 0.55rem 0.75rem;
  background: rgba(15, 23, 42, 0.55);
  border: 1px solid rgba(148, 163, 184, 0.3);
  border-radius: 12px;
  backdrop-filter: blur(10px);
}

.meter-title {
  min-width: 4.6rem;
  font-weight: 700;
  color: var(--text-soft);
  letter-spacing: 0.05em;
}

.meter-track {
  height: 0.72rem;
  border-radius: 999px;
  background: rgba(30, 41, 59, 0.85);
  overflow: hidden;
}

.meter-fill {
  height: 100%;
  border-radius: inherit;
  transition: width 180ms ease;
}

.health-fill {
  background: linear-gradient(90deg, #ef4444 0%, #f97316 40%, #22c55e 100%);
}

.focus-fill {
  background: linear-gradient(90deg, #0ea5e9 0%, #22d3ee 50%, #facc15 100%);
}

.meter-value {
  min-width: 3.4rem;
  text-align: right;
  font-weight: 700;
  color: #f8fafc;
}

.stats-row {
  display: flex;
  flex-wrap: wrap;
  gap: 0.7rem;
}

.stat-pill {
  display: flex;
  align-items: center;
  gap: 0.45rem;
  padding: 0.42rem 0.75rem;
  border-radius: 999px;
  background: rgba(15, 23, 42, 0.62);
  border: 1px solid rgba(148, 163, 184, 0.34);
  backdrop-filter: blur(8px);
}

.stat-label {
  color: var(--text-soft);
  font-size: 0.84rem;
}

.stat-value {
  font-family: 'Rajdhani', 'Noto Sans SC', sans-serif;
  font-weight: 700;
  color: #f8fafc;
  letter-spacing: 0.03em;
}

.hud-tip {
  margin: 0;
  color: rgba(191, 219, 254, 0.7);
  font-size: 0.86rem;
  letter-spacing: 0.02em;
  line-height: 1.4;
}

.focus-button {
  position: absolute;
  right: 1rem;
  bottom: 1rem;
  z-index: 6;
  border: none;
  border-radius: 999px;
  padding: 0.72rem 1rem;
  font-weight: 700;
  color: #082f49;
  background: linear-gradient(140deg, #67e8f9, #06b6d4);
  box-shadow: 0 10px 24px rgba(6, 182, 212, 0.35);
}

.focus-button.active {
  color: #422006;
  background: linear-gradient(140deg, #fde68a, #facc15);
  box-shadow: 0 10px 24px rgba(250, 204, 21, 0.38);
}

.overlay-layer,
.pause-layer {
  position: absolute;
  inset: 0;
  z-index: 10;
  display: grid;
  place-items: center;
  padding: 1.2rem;
  background: linear-gradient(160deg, rgba(8, 14, 22, 0.7), rgba(7, 15, 26, 0.82));
  backdrop-filter: blur(6px);
}

.panel {
  width: min(780px, 100%);
  border-radius: 20px;
  border: 1px solid var(--panel-border);
  background: linear-gradient(165deg, rgba(7, 20, 34, 0.92) 0%, rgba(10, 25, 40, 0.88) 100%);
  box-shadow: 0 24px 80px rgba(0, 0, 0, 0.45);
  padding: clamp(1.1rem, 2.8vw, 2rem);
  animation: panel-in 420ms cubic-bezier(0.22, 1, 0.36, 1);
}

.panel-subtitle {
  margin: 0;
  color: rgba(125, 211, 252, 0.88);
  letter-spacing: 0.26em;
  font-size: 0.76rem;
  font-family: 'Rajdhani', 'Noto Sans SC', sans-serif;
  font-weight: 700;
  text-transform: uppercase;
}

.panel h1,
.panel h2 {
  margin: 0.35rem 0 0;
  font-size: clamp(1.6rem, 3.5vw, 2.4rem);
  color: #f0f9ff;
  line-height: 1.15;
}

.panel-description {
  margin: 0.9rem 0 0;
  color: var(--text-soft);
  line-height: 1.65;
}

.feature-list {
  margin: 1rem 0 0;
  padding-left: 1.1rem;
  color: rgba(186, 230, 253, 0.92);
  display: grid;
  gap: 0.35rem;
}

.feature-list li {
  opacity: 0;
  animation: line-in 350ms ease forwards;
}

.feature-list li:nth-child(1) {
  animation-delay: 130ms;
}

.feature-list li:nth-child(2) {
  animation-delay: 220ms;
}

.feature-list li:nth-child(3) {
  animation-delay: 310ms;
}

.feature-list li:nth-child(4) {
  animation-delay: 390ms;
}

.feature-list li:nth-child(5) {
  animation-delay: 470ms;
}

.form-row {
  margin-top: 1rem;
  display: grid;
  gap: 0.45rem;
}

.field-label {
  color: rgba(186, 230, 253, 0.84);
  font-size: 0.88rem;
}

.name-input {
  border: 1px solid rgba(148, 163, 184, 0.42);
  border-radius: 12px;
  padding: 0.7rem 0.88rem;
  font-size: 1rem;
  color: #f8fafc;
  background: rgba(15, 23, 42, 0.65);
  transition: border-color 160ms ease, box-shadow 160ms ease;
}

.name-input:focus {
  outline: none;
  border-color: rgba(34, 211, 238, 0.72);
  box-shadow: 0 0 0 3px rgba(34, 211, 238, 0.18);
}

.difficulty-switch {
  display: inline-flex;
  margin-top: 0.95rem;
  padding: 0.2rem;
  border-radius: 999px;
  background: rgba(15, 23, 42, 0.68);
  border: 1px solid rgba(148, 163, 184, 0.32);
}

.difficulty-btn {
  border: none;
  border-radius: 999px;
  padding: 0.45rem 0.95rem;
  font-weight: 700;
  color: rgba(186, 230, 253, 0.76);
  background: transparent;
  transition: background-color 160ms ease, color 160ms ease, transform 160ms ease;
}

.difficulty-btn.selected {
  background: linear-gradient(120deg, #22d3ee, #0ea5e9);
  color: #042f4c;
  transform: translateY(-1px);
}

.feedback-settings {
  margin-top: 0.95rem;
  border: 1px solid rgba(148, 163, 184, 0.3);
  border-radius: 12px;
  background: rgba(15, 23, 42, 0.52);
  padding: 0.7rem 0.75rem;
  display: grid;
  gap: 0.52rem;
}

.feedback-title {
  margin: 0;
  font-size: 0.84rem;
  letter-spacing: 0.04em;
  color: rgba(186, 230, 253, 0.9);
  font-weight: 700;
}

.feedback-toggle-row,
.feedback-style-row {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
}

.toggle-chip,
.style-chip {
  border: 1px solid rgba(148, 163, 184, 0.36);
  border-radius: 999px;
  padding: 0.38rem 0.8rem;
  font-size: 0.84rem;
  font-weight: 700;
  color: rgba(186, 230, 253, 0.76);
  background: rgba(30, 41, 59, 0.72);
  transition: transform 160ms ease, background-color 160ms ease, color 160ms ease, border-color 160ms ease;
}

.toggle-chip:hover,
.style-chip:hover {
  transform: translateY(-1px);
  border-color: rgba(186, 230, 253, 0.62);
}

.toggle-chip.on {
  color: #042f4c;
  border-color: rgba(34, 211, 238, 0.8);
  background: linear-gradient(120deg, #67e8f9, #22d3ee);
}

.style-chip.selected {
  color: #082f49;
  border-color: rgba(56, 189, 248, 0.84);
  background: linear-gradient(120deg, #7dd3fc, #38bdf8);
}

.best-tip {
  margin: 0.8rem 0 0;
  color: rgba(250, 204, 21, 0.9);
  font-weight: 600;
}

.button-row {
  display: flex;
  flex-wrap: wrap;
  gap: 0.65rem;
  margin-top: 1.15rem;
}

.primary-btn,
.ghost-btn {
  border: none;
  border-radius: 12px;
  padding: 0.64rem 1rem;
  font-weight: 700;
  font-size: 0.98rem;
  transition: transform 160ms ease, box-shadow 160ms ease, background-color 160ms ease, color 160ms ease;
}

.primary-btn {
  background: linear-gradient(130deg, var(--accent), var(--accent-strong));
  color: #082f49;
  box-shadow: 0 10px 22px rgba(6, 182, 212, 0.34);
}

.primary-btn:hover {
  transform: translateY(-1px);
  box-shadow: 0 14px 28px rgba(6, 182, 212, 0.38);
}

.ghost-btn {
  background: rgba(148, 163, 184, 0.2);
  color: #dbeafe;
  border: 1px solid rgba(148, 163, 184, 0.32);
}

.ghost-btn:hover {
  transform: translateY(-1px);
  background: rgba(148, 163, 184, 0.28);
}

.pause-panel {
  width: min(460px, 100%);
}

.table-wrap {
  margin-top: 0.95rem;
  max-height: 380px;
  overflow: auto;
  border-radius: 12px;
  border: 1px solid rgba(148, 163, 184, 0.24);
}

.table-wrap table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.9rem;
}

.table-wrap th,
.table-wrap td {
  padding: 0.58rem 0.5rem;
  border-bottom: 1px solid rgba(148, 163, 184, 0.15);
  text-align: center;
}

.table-wrap thead {
  background: rgba(15, 23, 42, 0.75);
  color: rgba(224, 242, 254, 0.84);
}

.table-wrap tbody tr:nth-child(odd) {
  background: rgba(15, 23, 42, 0.42);
}

.table-wrap tbody tr:hover {
  background: rgba(34, 211, 238, 0.1);
}

.summary-line {
  margin: 0.8rem 0 0;
  font-size: 1.05rem;
  color: var(--text-soft);
}

.summary-line strong {
  color: #facc15;
  margin-left: 0.25rem;
  font-family: 'Rajdhani', 'Noto Sans SC', sans-serif;
  font-size: 1.45rem;
}

.summary-grid {
  margin-top: 1rem;
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 0.7rem;
}

.summary-grid div {
  border-radius: 12px;
  border: 1px solid rgba(148, 163, 184, 0.22);
  background: rgba(15, 23, 42, 0.54);
  padding: 0.66rem 0.78rem;
  display: grid;
  gap: 0.34rem;
}

.summary-grid span {
  color: rgba(186, 230, 253, 0.72);
  font-size: 0.86rem;
}

.summary-grid strong {
  font-size: 1.12rem;
  color: #f8fafc;
  font-family: 'Rajdhani', 'Noto Sans SC', sans-serif;
}

.gameover-panel.highlight {
  border-color: rgba(250, 204, 21, 0.5);
  box-shadow: 0 24px 80px rgba(250, 204, 21, 0.16), 0 24px 80px rgba(0, 0, 0, 0.42);
}

@keyframes panel-in {
  from {
    opacity: 0;
    transform: translateY(18px) scale(0.98);
  }

  to {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
}

@keyframes line-in {
  from {
    opacity: 0;
    transform: translateX(-8px);
  }

  to {
    opacity: 1;
    transform: translateX(0);
  }
}

@media (max-width: 900px) {
  .hud {
    top: 0.65rem;
    left: 0.65rem;
    right: 0.65rem;
    gap: 0.55rem;
  }

  .meter-card {
    padding: 0.5rem 0.62rem;
    gap: 0.5rem;
  }

  .meter-title {
    min-width: 3.6rem;
    font-size: 0.82rem;
  }

  .meter-value {
    font-size: 0.84rem;
  }

  .stat-pill {
    padding: 0.34rem 0.64rem;
  }

  .hud-tip {
    font-size: 0.76rem;
  }

  .button-row {
    width: 100%;
  }

  .primary-btn,
  .ghost-btn {
    flex: 1;
    min-width: 8rem;
  }

  .table-wrap {
    max-height: 300px;
  }
}

@media (max-width: 640px) {
  .panel {
    width: 100%;
    border-radius: 16px;
    padding: 1rem;
  }

  .feature-list {
    font-size: 0.9rem;
  }

  .summary-grid {
    grid-template-columns: 1fr;
  }

  .table-wrap th,
  .table-wrap td {
    font-size: 0.78rem;
    padding: 0.45rem 0.3rem;
  }
}
</style>
