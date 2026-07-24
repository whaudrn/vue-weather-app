<script setup>

import { ref, computed, watch, nextTick } from 'vue'
import { Vue3Lottie } from "vue3-lottie";



const {
  forecastData,
  weatherLottieMap,
} = defineProps({
  forecastData: Array,
  weatherLottieMap: Object,
})

const forecastDays = computed(() => {
  return forecastData
    .map(item => item.day)
    .filter((item, index, self) => self.indexOf(item) === index);
});

const selectedDay = ref("");
const isDragging = ref(false);
const translateX = ref(0);
const minX = ref(0);

const slideRef = ref(null);
const slideWrapperRef = ref(null);
const cardRefs = ref([]);

let startX = 0;
let moveX = 0;
let prevX = 0;

watch(
  () => forecastData,
  async () => {
    await nextTick();
    minX.value = slideWrapperRef.value.offsetWidth - slideRef.value.scrollWidth;
  },
  { deep: true }
);

const startDrag = (e) => {
  isDragging.value = true;
  startX = e.pageX;
  e.currentTarget.setPointerCapture(e.pointerId);
};

const moveDrag = (e) => {
  if (!isDragging.value) return;
  moveX = e.pageX - startX;
  const total = prevX + moveX;
  if (total > 0) {
    translateX.value = 0;
    prevX = 0;
    return;
  }
  if (total < minX.value) {
    translateX.value = minX.value;
    prevX = minX.value;
    return;
  }
  translateX.value = total;
};

const endDrag = (e) => {
  prevX += moveX;
  moveX = 0;
  isDragging.value = false;
  e.currentTarget.releasePointerCapture(e.pointerId);
};

const clickDayBtn = (day, index) => {
  translateX.value = 0;
  if (index === 0) {
    translateX.value = 0;
  }
  if (index === forecastDays.value.length - 1) {
    translateX.value = minX.value;
  }
  const idx = forecastData.findIndex(item => item.day === day);
  const gap = cardRefs.value[idx].offsetWidth / 2;
  const left = cardRefs.value[idx].offsetLeft;
  if (0 < index && 5 > index) {
    translateX.value =
      translateX.value - left + gap;
  }
  prevX = translateX.value;
  selectedDay.value = day;
};

</script>

<template>
  <div class="forecast" ref="forecastRef">
    <div class="slideWrapper" ref="slideWrapperRef">
      <div class="slide" :class="{ noTransition: isDragging }" ref="slideRef" @pointerdown="startDrag"
        @pointermove="moveDrag" @pointerup="endDrag" :style="{ transform: `translateX(${translateX}px)` }">
        <div v-for="(item, index) in forecastData" :key="index" :class="{ active: item.day === selectedDay }"
          :ref="el => cardRefs[index] = el">
          <span>{{ item.day }}</span>
          <span>{{ item.time }}</span>
          <Vue3Lottie class="forecastWeather" :animationData="weatherLottieMap[item.weather]" :width="30" :height="30" />
          <span>{{ item.temp }}℃</span>
        </div>
      </div>
    </div>
    <div class="btnWrap">
      <button v-for="(day, index) in forecastDays" @click="clickDayBtn(day, index)"
        :class="{ highlight: day === selectedDay }">
        {{ day }}</button>
    </div>
  </div>
</template>

<style scoped></style>