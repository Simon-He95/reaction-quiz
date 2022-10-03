<script setup lang="ts">
import { onMounted } from "@vue/runtime-core";
import {
  addEventListener,
  animationFrameWrapper,
  createElement,
  insertElement,
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
  const random = Math.floor(Math.random() * 4);
  const div = createElement("div", {
    style: "background:red",
  });
  const width = div.offsetWidth;
  const height = div.offsetHeight;
  const randomWidth = Math.random() * (w + 2 * width) - width;
  const randomHeight = Math.random() * (h + 2 * height) - h;
  console.log("----", width, div, "----");
  const start = {
    0: `top:${-height}px;left:${randomWidth}px;background:red;width:10px;height:10px;position:absolute;`,
    1: `top:${
      height + h
    }px;left:${randomWidth}px;background:red;width:10px;height:10px;position:absolute;`,
    2: `left:${-width}px;top:${randomHeight}px;background:red;width:10px;height:10px;position:absolute;`,
    3: `left:${
      width + w
    }px;top:${randomHeight}px;background:red;width:10px;height:10px;position:absolute;`,
  };
  // div.style = div.style + start[random];
  div.setAttribute("style", start[random]);
  // top.value + height / ? = randomWidth-left.value /randomWidth+width
  // left.value-randomWidth/w-randomWidth+width = top.value / ?
  // 0: if(randomWidth > left.value) top.value + height / ? = randomWidth-left.value / randomWidth+width
  // or h+height-top.value / h+2*height = left/value-randomWidth/?
  // 1: if(randomWidth > left.value) h+height-top.value / ? = randomWidth-left.value/randomWidth+width
  // or left.value-randomWidth/? = h+height-top.value/h+2*height
  // 2 : if(randomHeight>top.value) left.value + width/? = randomHeight-top.value/randomHeight+height
  // or top.value-randomHeight / h+height-randomHeight = left.value+width/?
  // 3: if(randomHeight>top.value)  w+width-left.value / ? = randomHeight-top.value / randomHeight + height
  // or w+width-left.value/? = top.value-randomHeight/  h+height-randomHeight
  function getEndPosition(random) {
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
              (randomWidth + (randomWidth + width) * (top.value + height)) /
                (randomWidth - left.value),
              height + h,
            ],
      1:
        randomWidth > left.value
          ? [
              -width,
              h +
                height -
                ((randomWidth + width) * (h + height - top.value)) /
                  (randomWidth - left.value),
            ]
          : [
              randomWidth +
                ((h + 2 * height) * (left.value - randomWidth)) /
                  (h + height - top.value),
              -height,
            ],
      2:
        randomHeight > top.value
          ? [
              ((randomHeight + height) * (left.value + width)) /
                (randomHeight - top.value) -
                width,
              -height,
            ]
          : [
              ((h + height - randomHeight) * (left.value + width)) /
                (top.value - randomHeight) -
                width,
              height + h,
            ],
      3:
        randomHeight > top.value
          ? [
              width +
                w -
                ((randomHeight + height) * (w + width - left.value)) /
                  (randomHeight - top.value),
              -height,
            ]
          : [
              w +
                width -
                ((h + height - randomHeight) * (w + width - left.value)) /
                  (top.value - randomHeight),
              h + height,
            ],
    };
    return end[random];
  }
  function getStartPosition(random) {
    const start = {
      0: [randomWidth, -height],
      1: [randomWidth, height + h],
      2: [-width, randomHeight],
      3: [width + w, randomHeight],
    };
    return start[random];
  }
  let [lat1, lng1] = getStartPosition(random);
  const [lat2, lng2] = getEndPosition(random);
  const distanceX = lat2 - lat1;
  const distanceY = lng2 - lng1;
  const stop = animationFrameWrapper(() => {
    if (Math.floor(lat1) === Math.floor(lat2)) return stop();

    lat1 += distanceX / 100;
    lng1 += distanceY / 100;
    div.style.left = `${lat1}px`;
    div.style.top = `${lng1}px`;
  }, 100);
  insertElement("main", div, null);
}

onMounted(() => {
  setTimeout(() => {
    generateObject();
  }, 1000);
});
</script>

<template>
  <main font-sans p="x-4 y-10" h-full text="center gray-700 dark:gray-200">
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
