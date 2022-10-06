<script setup lang="ts">
import { onMounted } from "@vue/runtime-core";
import {
  addEventListener,
  animationFrameWrapper,
  collisionDetection,
  createElement,
  findElement,
  insertElement,
  removeElement,
  useIntersectionObserver,
} from "simon-js-tool";
import { blocks } from "./ball";
const top = ref(0);
const left = ref(0);
let over;
const status = ref(true);
const times = ref(0);
const blood = ref(100);
const level = ref(1);

const target = ref(null);

addEventListener(document, "mousemove", (e) => {
  top.value = e.y - target.value?.offsetWidth / 2;
  left.value = e.x - target.value?.offsetHeight / 2;
});
const w = window.innerWidth;
const h = window.innerHeight;
function generateObject() {
  let random = 0;
  const div = createElement("div", {
    class: "blocks",
  });
  const ball = blocks[Math.floor(Math.random() * blocks.length)];
  div.innerHTML = ball;
  const width = div.offsetWidth;
  const height = div.offsetHeight;
  const randomWidth = getRandomWidth();
  const randomHeight = getRandomHeight();
  if (left.value < w / 2) {
    if (top.value < h / 2) {
      if (Math.abs(left.value) > Math.abs(top.value)) random = 0;
      else random = 2;
    } else {
      random = Math.abs(left.value) > Math.abs(h - top.value) ? 1 : 2;
    }
  } else {
    if (top.value < h / 2)
      random = Math.abs(w - left.value) > Math.abs(top.value) ? 0 : 3;
    else random = Math.abs(w - left.value) > Math.abs(h - top.value) ? 1 : 3;
  }
  times.value++;
  if (times.value > 60) level.value++;
  if (blood.value < 100) blood.value += 1;
  let [lat1, lng1] = getStartPosition(random);
  const initialStatus = `left:${lat1}px;top:${lng1}px;position:absolute;`;
  div.setAttribute("style", initialStatus);
  function getEndPosition(random: number): number[] {
    const end: Record<number, number[]> = {
      0:
        randomWidth > left.value
          ? [
              -width,
              maxHeight(
                ((top.value + height) * (randomWidth + width)) /
                  (randomWidth - left.value) -
                  height
              ),
            ]
          : [
              maxWidth(
                randomWidth +
                  ((randomWidth + width) * (top.value + height)) /
                    (randomWidth - left.value)
              ),
              height + h,
            ],
      1:
        randomWidth > left.value
          ? [
              -width,
              maxHeight(
                h +
                  height -
                  ((randomWidth + width) * (h + height - top.value)) /
                    (randomWidth - left.value)
              ),
            ]
          : [
              maxWidth(
                randomWidth +
                  ((h + 2 * height) * (left.value - randomWidth)) /
                    (h + height - top.value)
              ),
              -height,
            ],
      2:
        randomHeight > top.value
          ? [
              maxWidth(
                ((randomHeight + height) * (left.value + width)) /
                  (randomHeight - top.value) -
                  width
              ),
              -height,
            ]
          : [
              maxWidth(
                ((h + height - randomHeight) * (left.value + width)) /
                  (top.value - randomHeight) -
                  width
              ),
              height + h,
            ],
      3:
        randomHeight > top.value
          ? [
              maxWidth(
                width +
                  w -
                  ((randomHeight + height) * (w + width - left.value)) /
                    (randomHeight - top.value)
              ),
              -height,
            ]
          : [
              maxWidth(
                w +
                  width -
                  ((h + height - randomHeight) * (w + width - left.value)) /
                    (top.value - randomHeight)
              ),
              h + height,
            ],
    };
    return end[random];
  }
  function getStartPosition(random: number): number[] {
    const start: Record<number, number[]> = {
      0: [randomWidth, -height],
      1: [randomWidth, height + h],
      2: [-width, randomHeight],
      3: [width + w, randomHeight],
    };
    return start[random];
  }

  function maxHeight(data: number) {
    return data > 0 ? Math.min(data, h + height) : Math.max(data, -height);
  }
  function maxWidth(data: number) {
    return data > 0 ? Math.min(data, w + width) : Math.max(data, -width);
  }
  const [lat2, lng2] = getEndPosition(random);
  const distanceX = lat2 - lat1;
  const distanceY = lng2 - lng1;
  useIntersectionObserver(div, ([{ isIntersecting }, observerElement]) => {
    if (!observerElement) removeElement(div);
  });
  const stop = animationFrameWrapper(() => {
    if (!status.value) return stop();
    lat1 += distanceX / 100;
    lng1 += distanceY / 100;
    div.style.left = `${lat1}px`;
    div.style.top = `${lng1}px`;
  }, 100 / level.value);
  const stop1 = animationFrameWrapper(() => {
    if (!status.value) return stop1();
    if (collisionDetection(div, ".target")) {
      blood.value -= 20;
      if (blood.value <= 0) {
        status.value = false;
        setTimeout(() => {
          alert(`game over ! lasted ${times.value} seconds`);
          over();
        });
      }
      return stop();
    }
  }, 0.01);

  insertElement("main", div, null);

  function getRandomWidth() {
    const result = Math.random() * (w + 2 * width) - width;
    if (result < 100) return getRandomWidth();
    return result;
  }
  function getRandomHeight() {
    const result = Math.random() * (h + 2 * height) - h;
    if (result < 100) return getRandomWidth();
    return result;
  }
}

onMounted(() => {
  over = animationFrameWrapper(() => {
    const len = findElement(".blocks", true)!.length;
    if (len <= 10) generateObject();
  }, 1000);
});

function isStr(o: any): o is string {
  return typeof o === "string";
}

const bloodColor = computed(() => {
  if (blood.value >= 80) return "bg-green";
  if (blood.value >= 60) return "bg-yellow";
  return "bg-red";
});
</script>

<template>
  <main
    ref="main"
    font-sans
    p="x-4 y-10"
    h-full
    text="center gray-700 dark:gray-200"
  >
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
