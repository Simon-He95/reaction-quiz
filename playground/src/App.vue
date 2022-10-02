<script setup lang="ts">
import { onMounted } from "@vue/runtime-core";
import {
  addEventListener,
  createElement,
  useIntersectionObserver,
} from "simon-js-tool";
const props = defineProps<{ modelValue: boolean }>();
const top = ref(0);
const left = ref(0);

// to do 检测物体对于x轴和y轴的垂线同时重叠证明碰撞

addEventListener(document, "mousemove", (e) => {
  top.value = e.y;
  left.value = e.x;
});
const w = window.innerWidth;
const h = window.innerHeight;
function generateObject() {
  const div = createElement("div");
  const width = div.offsetWidth;
  const height = div.offsetHeight;
  const randomWidth = Math.random() * (w + 2 * width) - width;
  const randomHeight = Math.random() * (h + 2 * height) - h;
  const start = {
    0: `style:"top:${-height};left:${randomWidth}"`,
    1: `style:"top:${height + h};left:${randomWidth}"`,
    2: `style:"left:${width};top:${randomHeight}`,
    3: `style:"left:${width + w};top:${randomHeight}`,
  };
  // top.value + height / ? = randomWidth-left.value /randomWidth+width
  // left.value-randomWidth/w-randomWidth+width = top.value / ?
  // 0: if(randomWidth > left.value) top.value + height / ? = randomWidth-left.value / randomWidth+width
  // or top.value+height / ? = left.value-randomWidth / w+width-randomWidth
  // 1: if(randomWidth > left.value) w-randomWidth / ? = h+height-top.value / h+2*height
  // or left.value - randomWidth / ? = (h+height-top.value) / (h+2*height)

  function getEndPosition() {
    const end = {
      0:
        randomWidth > left.value
          ? [
              -width,
              ((top.value + height) * (randomWidth + width)) /
                (randomWidth - left.value) -
                height,
            ]
          : [
              w + width,
              ((top.value + height) * (w - randomWidth + width)) /
                (left.value - randomWidth) -
                height,
            ],
      1:
        randomWidth > left.value
          ? [
              -height,
              ((h + 2 * height) * (w - randomWidth)) /
                (h + height - top.value) -
                (w - randomWidth),
            ]
          : [
              -height,
              ((h + 2 * height) * (left.value - randomWidth)) /
                (h + height - top.value) +
                randomWidth,
            ],
    };
  }
  const [l, t] = getEndPosition();
  console.log(start[0], left.value, top.value, "---", l, t);
}
onMounted(() => {
  setTimeout(() => {
    generateObject();
  }, 1000);
});
</script>

<template>
  <main font-sans p="x-4 y-10" text="center gray-700 dark:gray-200">
    <div
      w-10
      h-10
      bg-red
      class="target"
      absolute
      :style="{ top: `${top}px`, left: `${left}px` }"
    />
    <div w-40 h-40 border-rd-full bg-yellow class="obj" />
    <Footer />
  </main>
</template>

<style scoped>
</style>
