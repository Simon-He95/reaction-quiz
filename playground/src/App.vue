<script setup lang="ts">
import { onMounted } from '@vue/runtime-core'
import {
  addEventListener,
  animationFrameWrapper,
  collisionDetection,
  createElement,
  insertElement,
  randomRgb,
  removeElement,
} from 'simon-js-tool'
const top = ref(0)
const left = ref(0)
let over
const status = ref(true)
const times = ref(0)
const blood = ref(100)
const level = ref(1)

const target = ref(null)
addEventListener(document, 'mousemove', (e) => {
  top.value = e.y - target.value?.offsetWidth / 2
  left.value = e.x - target.value?.offsetHeight / 2
})
const w = window.innerWidth
const h = window.innerHeight
function generateObject() {
  times.value++
  if (times.value > 60)
    level.value++
  if (blood.value < 100)
    blood.value += 1
  const random: number = Math.floor(Math.random() * 4)
  const div = createElement('div')
  const width = div.offsetWidth
  const height = div.offsetHeight
  const randomWidth = Math.random() * (w + 2 * width) - width
  const randomHeight = Math.random() * (h + 2 * height) - h
  const randomBlockW = Math.random() * 10 + 10
  const randomBlockH = Math.random() * 10 + 10
  const radius = Math.random() * 100
  let [lat1, lng1] = getStartPosition(random)
  const initialStatus = `left:${lat1}px;top:${lng1}px;background:${randomRgb()};width:${randomBlockW}px;height:${randomBlockH}px;position:absolute;border-radius:${radius}%;`
  div.setAttribute('style', initialStatus)
  function getEndPosition(random: number): number[] {
    const end: Record<number, number[]> = {
      0:
        randomWidth > left.value
          ? [
              -width,
              ((top.value + height) * (randomWidth + width))
                / (randomWidth - left.value)
                - height,
            ]
          : [
              (randomWidth + (randomWidth + width) * (top.value + height))
                / (randomWidth - left.value),
              height + h,
            ],
      1:
        randomWidth > left.value
          ? [
              -width,
              h
                + height
                - ((randomWidth + width) * (h + height - top.value))
                  / (randomWidth - left.value),
            ]
          : [
              randomWidth
                + ((h + 2 * height) * (left.value - randomWidth))
                  / (h + height - top.value),
              -height,
            ],
      2:
        randomHeight > top.value
          ? [
              ((randomHeight + height) * (left.value + width))
                / (randomHeight - top.value)
                - width,
              -height,
            ]
          : [
              ((h + height - randomHeight) * (left.value + width))
                / (top.value - randomHeight)
                - width,
              height + h,
            ],
      3:
        randomHeight > top.value
          ? [
              width
                + w
                - ((randomHeight + height) * (w + width - left.value))
                  / (randomHeight - top.value),
              -height,
            ]
          : [
              w
                + width
                - ((h + height - randomHeight) * (w + width - left.value))
                  / (top.value - randomHeight),
              h + height,
            ],
    }
    return end[random]
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
  const [lat2, lng2] = getEndPosition(random)
  const distanceX = lat2 - lat1
  const distanceY = lng2 - lng1
  const stop = animationFrameWrapper(() => {
    if (!status.value)
      return stop()
    if (collisionDetection(div, '.target')) {
      blood.value -= 20
      if (blood.value <= 0) {
        status.value = false
        setTimeout(() => {
          alert(`game over ! lasted ${times.value} seconds`)
          over()
        })
      }
      removeElement(div)
      return stop()
    }
    if (Math.floor(lat1) === Math.floor(lat2)) {
      removeElement(div)
      return stop()
    }

    lat1 += distanceX / 100
    lng1 += distanceY / 100
    div.style.left = `${lat1}px`
    div.style.top = `${lng1}px`
  }, 100 / level.value)
  insertElement('main', div, null)
}

onMounted(() => {
  over = animationFrameWrapper(() => {
    generateObject()
  }, 1500)
})

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
  <main font-sans p="x-4 y-10" h-full text="center gray-700 dark:gray-200">
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

<style scoped>
</style>
