<script setup>

import { computed } from 'vue'

import star1 from '../assets/star1.png';
import star2 from '../assets/star2.png';

const {
  favoriteList,
  address,
  hasAddress,
} = defineProps({
  favoriteList: Array,
  address: String,
  hasAddress: Boolean,
})

const emit = defineEmits([
  "addFavorite",
])
const currentFavor = computed(() => favoriteList.find(item => item.add === address))
const isFavor = computed(() => {
  return favoriteList.some(item => item.add === address)
});
</script>

<template>
  <div class="currentLocation">
    <p style="color: red;font-weight: bold;">현재위치</p>
    <div :class="{ isCurrentFavor: currentFavor && currentFavor.add !== currentFavor.name }">
      <span v-if="currentFavor && currentFavor.add !== currentFavor.name">{{ currentFavor.name
      }}</span>
      <span>{{ address }}</span>
      <button @click="emit('addFavorite')" :disabled="!hasAddress">
        <img :src="isFavor ? star2 : star1">
      </button>
    </div>
  </div>
</template>

<style scoped></style>