<script setup>

import { ref, onMounted, onUnmounted } from 'vue'
import { Vue3Lottie } from "vue3-lottie";
import search from '../assets/lottie/search.json';

const {
  recentList,
} = defineProps({
  recentList: Array,
})

const emit = defineEmits([
  "loadLocation",
  "searchLocation",
  "saveRecentList",
  "delRecentList",
])

const searchText = ref("");
const isRecentListOpen = ref(false);
const recentRef = ref(null);

const handleRecentOutsideClick = (e) => {
  if (!recentRef.value) return;
  if (!recentRef.value.contains(e.target)) {
    isRecentListOpen.value = false;
  }
};
onMounted(() => { document.addEventListener("click", handleRecentOutsideClick); })
onUnmounted(() => { document.removeEventListener("click", handleRecentOutsideClick); })

const clickRecentList = (item) => {
  isRecentListOpen.value = false;
  emit("loadLocation", item.lat, item.lon)
  emit("saveRecentList",item.text, item.lat, item.lon)
}

const openRecentList = () => {
  if (recentList.length > 0) {
    isRecentListOpen.value = true;
  }
}

</script>

<template>
  <div class="inputArea" :class="{ on: isRecentListOpen }" ref="recentRef">
    <input placeholder="지역(동,면)을 입력하여 검색 또는 지도를 클릭하세요." v-model="searchText" @keyup.enter="emit('searchLocation',searchText)"
      @focus="openRecentList">
    <div class="recentBox">
      <div v-for="item in recentList" class="recentItem" @click="clickRecentList(item)">
        {{ item.text }}
        <button @click.stop="emit('delRecentList',item)">삭제</button>
      </div>
    </div>
    <button @click="emit('searchLocation',searchText)">
      <Vue3Lottie class="searchLottie" :animationData="search" :width="30" :height="30" />
    </button>
  </div>
</template>

<style scoped></style>