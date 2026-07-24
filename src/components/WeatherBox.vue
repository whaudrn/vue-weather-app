<script setup>

import { ref, watch } from 'vue'

const {
  weatherInfo,
} = defineProps({
  weatherInfo: Array,
})

const finishTyping = ref(false);
const isMinimize = ref(false);
const typedWeatherText = ref([]);

const minimize = () => {
  isMinimize.value = !isMinimize.value;
}

let typingTimer = null;

const typingWeatherBox = () => {
  clearInterval(typingTimer);
  finishTyping.value = false;
  typedWeatherText.value = [];
  typedWeatherText.value.push("");

  let i = 0;
  let j = 0;

  typingTimer = setInterval(() => {
    if (i >= weatherInfo.length) {
      finishTyping.value = true;
      clearInterval(typingTimer);
      return;
    }
    typedWeatherText.value[i] += weatherInfo[i][j];
    j++;
    if (j >= weatherInfo[i].length) {
      i++;
      j = 0;
      if (i < weatherInfo.length) {
        typedWeatherText.value.push("");
      }
    }
  }, 20);
};
watch(() => weatherInfo, () => { typingWeatherBox() })

</script>

<template>
  <div class="weatherBox" :class="{ mini: isMinimize }">
    <p v-for="text in typedWeatherText" :key="text">{{ text }}</p>
    <button v-if="finishTyping" @click="minimize">{{ isMinimize ? '+' : 'X' }}</button>
  </div>

</template>

<style scoped></style>