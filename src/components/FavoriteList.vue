<script setup>

import { ref, computed, watch, nextTick, onMounted, onUnmounted } from 'vue'

import rename from '../assets/images/rename.png';
import save from '../assets/images/save.png';
import minus from '../assets/images/minus.png'
import star2 from '../assets/star2.png';
import down from '../assets/images/down.png';

const {
  favoriteList,
  selectedFavorDiv,
} = defineProps({
  favoriteList: Array,
  selectedFavorDiv: String,
})

const emit = defineEmits([
  "deleteFavorite",
  "deleteAllFavorites",
  "select",
  "clearSelect",
  "selectFavorite",
])

const favorListRef = ref(null);
const inputRef = ref(null)
const openBtnRef = ref(null)

const hoverItem = ref(null);
const editFavor = ref(null)
const editText = ref("")
const isClose = ref(false)
const currentDeg = ref(0)

const favoriteCount = computed(() => favoriteList.length);

onMounted(async () => {
  document.addEventListener("pointerdown", handleFavorOutsideClick);
  await nextTick();
  updateHeight()
})
onUnmounted(() => { document.removeEventListener("pointerdown", handleFavorOutsideClick); })

const deleteFavorite = (item) => { emit("deleteFavorite", item) }
const deleteAllFavorites = () => { emit("deleteAllFavorites") }

const toggleAccordion = () => {
  isClose.value = !isClose.value;
  currentDeg.value += 180;
  openBtnRef.value.style.transform = `rotate(${currentDeg.value}deg)`
  updateHeight();
}
watch(() => favoriteList, async () => {
  await nextTick();
  updateHeight()
}, { deep: true, immediate: true });

const updateHeight = () => {
  if (!favorListRef.value) return;

  if (isClose.value) {
    favorListRef.value.style.maxHeight = "0px";
  } else {
    favorListRef.value.style.maxHeight = `${favorListRef.value.scrollHeight}px`
  }
  console.log(favorListRef.value.scrollHeight);
}

const handleFavorOutsideClick = (e) => {
  if (!favorListRef.value) return;
  if (!favorListRef.value.contains(e.target)) {
    emit("clearSelect")
  }
}

const startRename = async (item) => {
  emit("select", item.add)
  editFavor.value = item.add;
  editText.value = item.name;
  await nextTick();
  inputRef.value.focus();
}

const saveRename = (item) => {
  if (editText.value.trim()) {
    item.name = editText.value;
  }
  editFavor.value = null;
  emit("clearSelect")
}

</script>

<template>
  <div class="favor">
    <div class="favorHead">
      <p>관심지역({{ favoriteCount }}/10)</p>
      <div class="btnBox">
        <button class="openBtn" @click="toggleAccordion"><img :src="down" ref="openBtnRef"></button>
        <button class="delAll" @click="deleteAllFavorites">전체삭제</button>
      </div>
    </div>
    <div class="favorList" :class="{ close: isClose }" ref="favorListRef">
      <div :class="{ highlight: selectedFavorDiv === item.add }" v-for="item in favoriteList" :key="item.add"
        @pointerdown="emit('selectFavorite', item)" @dblclick.stop="startRename(item)">
        <div v-if="editFavor === item.add" class="reName">
          <input maxlength="7" :ref="el => inputRef = el" v-model="editText" @keyup.enter="inputRef.blur()"
            @blur="saveRename(item)">
        </div>
        <div v-else>{{ item.name || item.add }}</div>
        <div class="iconBox">
          <div @pointerdown.stop.prevent="editFavor === item.add ? saveRename(item) : startRename(item)"><img
              :src="editFavor === item.add ? save : rename" alt=" 이름변경 아이콘"></div>
          <div class="favorBtn" @pointerdown.stop.prevent="deleteFavorite(item)" @mouseenter="hoverItem = item.add"
            @mouseleave="hoverItem = null"><img :src="hoverItem === item.add ? minus : star2" alt="즐겨찾기 제거 아이콘">
          </div>
        </div>
      </div>
    </div>
  </div>

</template>

<style scoped></style>