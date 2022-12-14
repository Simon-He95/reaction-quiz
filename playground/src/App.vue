<script setup lang="ts">
import {
  collisionDetection,
  createElement,
  findElement,
  insertElement,
  removeElement,
  useAnimationFrame,
  useEventListener,
  useIntersectionObserver,
} from 'lazy-js-utils'
import { computed, ref } from 'vue'
import { blocks } from './ball'
const top = ref(0)
const left = ref(0)
let over: () => void
const status = ref(true)
const times = ref(0)
const blood = ref(100)
const speed = ref(8)

const target = ref<HTMLElement>()
let start = false
let timer: any = null
const mousemove = (e: MouseEvent) => {
  top.value = e.y - target.value!.offsetWidth / 2
  left.value = e.x - target.value!.offsetHeight / 2
  if (start)
    return
  start = true
  over = useAnimationFrame(() => {
    const len = (findElement('.blocks', true) as NodeListOf<Element>).length
    if (len <= 10)
      generateObject()
  }, 2000)
  timer = setInterval(() => {
    times.value++
  }, 1000)
}
useEventListener(document, 'mousemove', mousemove)
const w = window.innerWidth
const h = window.innerHeight
function generateObject() {
  let random = 0
  const div = createElement('div', {
    class: 'blocks',
  })
  const ball = blocks[Math.floor(Math.random() * blocks.length)]
  div.innerHTML = ball
  const width = 32
  const height = 32
  const randomWidth = getRandomWidth() + width / 2
  const randomHeight = getRandomHeight() + height / 2
  if (left.value < w / 2) {
    if (top.value < h / 2) {
      if (Math.abs(left.value) > Math.abs(top.value))
        random = 0
      else random = 2
    }
    else {
      random = Math.abs(left.value) > Math.abs(h - top.value) ? 1 : 2
    }
  }
  else {
    if (top.value < h / 2)
      random = Math.abs(w - left.value) > Math.abs(top.value) ? 0 : 3
    else random = Math.abs(w - left.value) > Math.abs(h - top.value) ? 1 : 3
  }
  if (times.value > 60)
    speed.value++
  if (blood.value < 100)
    blood.value += 1
  let [lat1, lng1] = getStartPosition(random)
  const initialStatus = `left:${lat1}px;top:${lng1}px;position:absolute;`
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
  const stop = useAnimationFrame(() => {
    if (!status.value)
      return stop()
    gameover()
    lat1 += speed.value * directionX
    lng1 += speed.value * directionY
    div.style.left = `${lat1}px`
    div.style.top = `${lng1}px`
  }, 0.01)
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
      return getRandomWidth()
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
      if (blood.value <= 0) {
        status.value = false
        clearInterval(timer)
        setTimeout(() => {
          alert(`game over ! lasted ${times.value} seconds`)
          over()
        })
      }
    }
  }
}

function isStr(o: any): o is string {
  return typeof o === 'string'
}

const bloodColor = computed(() => {
  if (blood.value >= 80)
    return 'bg-green'
  if (blood.value >= 60)
    return 'bg-yellow'
  return 'bg-red'
})
</script>

<template>
  <main ref="main" font-sans h-full text="center gray-700 dark:gray-200" overflow-hidden>
    <div w-100 border-1 border-lightgray border-rd-2 h-2 relative ma text-1>
      <div
        :class="[bloodColor]"
        :style="{ width: `${blood <= 0 ? 0 : blood}%` }"
        h-full
        text-center
        color-white
        absolute
        flex
        items-center
        justify-center
      >
        {{ blood }}%
      </div>
    </div>
    <span mt1 ma>{{ times }}s</span>

    <div
      ref="target"
      class="target"
      absolute
      :style="{ top: `${top}px`, left: `${left}px` }"
    >
      <svg width="32" height="32" viewBox="0 0 256 256">
        <path
          fill="currentColor"
          d="m214.8 136.9l-39.2-50.4a5.2 5.2 0 0 0-1-1.1A31.5 31.5 0 0 0 152 76a40 40 0 1 0-48 0a31.5 31.5 0 0 0-22.6 9.4a5.2 5.2 0 0 0-1 1.1l-39.2 50.4a24 24 0 0 0 33.9 33.9l3.8-2.9L63 218.1a23.5 23.5 0 0 0-.4 17.5a24 24 0 0 0 43.9 2.7l21.5-33.8l21.5 33.8a23.9 23.9 0 0 0 13.2 11.6a23.3 23.3 0 0 0 8.2 1.5a24 24 0 0 0 22.1-33.3l-15.9-50.2l3.8 2.9a24 24 0 0 0 33.9-33.9ZM128 28a16 16 0 1 1-16 16a16 16 0 0 1 16-16Zm68.1 124.3l-34.7-27.1a12 12 0 0 0-18.8 13.1l27.7 87.6l.6 1.5a10 10 0 0 0-.8-1.4l-32-50.4a12 12 0 0 0-20.2 0l-32 50.4a10 10 0 0 0-.8 1.4l.6-1.5l27.7-87.6a11.9 11.9 0 0 0-4.6-13.4a11.3 11.3 0 0 0-6.8-2.2a12.7 12.7 0 0 0-7.4 2.5l-34.7 27.1l-1.2 1l1-1.2l39.1-50.2a8.1 8.1 0 0 1 5.2-1.9h48a8.1 8.1 0 0 1 5.2 1.9l39.1 50.2l1 1.2Z"
        />
      </svg>
    </div>
  </main>
</template>

<style scoped></style>
