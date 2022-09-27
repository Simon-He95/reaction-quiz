<script setup lang="ts">
import { onMounted } from "@vue/runtime-core";
import { addEventListener, useIntersectionObserver } from "simon-js-tool";
const top = ref(0);
const left = ref(0);

// to do 检测物体对于x轴和y轴的垂线同时重叠证明碰撞

addEventListener(document, "mousemove", (e) => {
  top.value = e.y;
  left.value = e.x;
});

const props = defineProps<{ modelValue: boolean }>();
</script>


<style scoped>
</style>

<template>
  <main font-sans p="x-4 y-10" text="center gray-700 dark:gray-200">
    <div
      w-10
      h-10
      bg-red
      class="target"
      absolute
      :style="{ top: top + 'px', left: left + 'px' }"
    ></div>
    <div w-40 h-40 border-rd-full bg-yellow class="obj"></div>
    <Footer />
  </main>
</template>
