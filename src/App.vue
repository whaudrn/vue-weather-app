<script setup>

// ======================
// Vue / Libraries
// ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
import lottie from "lottie-web";
import { ref, onMounted, onUnmounted, computed, watch } from 'vue';
import axios from 'axios';


// ======================
// Components
// ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
import Forecast from "./components/Forecast.vue";
import CurrentLocation from "./components/CurrentLocation.vue";
import FavoriteList from "./components/FavoriteList.vue";
import SearchBox from "./components/SearchBox.vue";
import WeatherBox from "./components/WeatherBox.vue";


// ======================
// Assets
// ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
import sunnyLottie from './assets/lottie/sunny.json';
import rainLottie from './assets/lottie/rain.json';
import cloudLottie from './assets/lottie/cloudsA.json';
import stormLottie from './assets/lottie/thunderstorm.json';
import snowLottie from './assets/lottie/snow.json';
import circle from './assets/lottie/circle.json';


// ======================
// Constants
// ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
const API_KEY = import.meta.env.VITE_OPENWEATHER_API_KEY;
const BASE_URL = 'https://api.openweathermap.org/data/2.5';
const SEARCH_KEY = "weather_recent_search";
const FAVOR_KEY = 'weather_favorites';
const weeks = ["일", "월", "화", "수", "목", "금", "토"];
const weatherLottieMap = {
  Clear: sunnyLottie,
  Rain: rainLottie,
  Clouds: cloudLottie,
  Thunderstorm: stormLottie,
  Snow: snowLottie
};


// ======================
// State
// ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
const currentLat = ref(null);
const currentLon = ref(null);
const address = ref('');

const favoriteList = ref([]);
const favoriteOverlays = ref([]);
const selectedFavorDiv = ref(null);

const weather = ref('');
const temp = ref(null);
const feelTemp = ref(null);
const humidity = ref(null);
const wind = ref(null);
const precipitation = ref(null)

const hasAddress = ref(false);
const forecastData = ref([]);


// ======================
// Recent Search
// ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
const recentList = ref([]);
const savedRecent = localStorage.getItem(SEARCH_KEY);

if (savedRecent) {
  recentList.value = JSON.parse(savedRecent);
}
const delRecentList = (item) => {
  recentList.value = recentList.value.filter(list => list.text !== item.text)
  localStorage.setItem(
    SEARCH_KEY,
    JSON.stringify(recentList.value)
  )
}

const saveRecentList = (text, lat, lon) => {
  recentList.value = recentList.value.filter(item => item.text !== text)
  recentList.value.unshift({
    text: text,
    lat: lat,
    lon: lon
  });
  recentList.value = recentList.value.slice(0, 10);
  localStorage.setItem(
    SEARCH_KEY,
    JSON.stringify(recentList.value)
  )
}


// ======================
// Weather Typing
// ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
const weatherInfo = computed(() => ([
  `날씨 : ${weather.value}`,
  `기온 : ${temp.value} ℃`,
  `체감 : ${feelTemp.value} ℃`,
  `습도 : ${humidity.value} %`,
  `바람 : ${wind.value} (m/s)`,
  `강수량 : ${precipitation.value} (mm/h)`
])
)


// ======================
// Search
// ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
const searchLocation = (text) => {
  if (!text.trim()) return;
  const moveSearchResult = (result) => {
    const lat = Number(result[0].y);
    const lon = Number(result[0].x);
    saveRecentList(text, lat, lon);
    loadLocation(lat, lon);
  };

  const geocoder = new kakao.maps.services.Geocoder();

  geocoder.addressSearch(text, (result, status) => {
    if (status === kakao.maps.services.Status.OK) {
      moveSearchResult(result);
    } else {
      const ps = new kakao.maps.services.Places();
      ps.keywordSearch(text, (result, status) => {
        if (status === kakao.maps.services.Status.OK) {
          moveSearchResult(result);
        } else {
          alert("검색 결과 없음");
        }
      });
    }
  });
};

// ======================
// Watch
// ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
watch(
  favoriteList,
  (newVal) => {
    localStorage.setItem(
      FAVOR_KEY,
      JSON.stringify(newVal)
    )
  },
  { deep: true }
)


// ======================
// Lifecycle
// ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
onMounted(() => {
  const saved = localStorage.getItem(FAVOR_KEY);
  if (saved) {
    favoriteList.value = JSON.parse(saved);
  }
  navigator.geolocation.getCurrentPosition(
    async (position) => {
      const lat = position.coords.latitude;
      const lon = position.coords.longitude;
      createMap(lat, lon);
      await loadLocation(lat, lon);

      await Promise.all(
        favoriteList.value.map(async (item) => {
          const data = await getWeather(item.lat, item.lon);
          item.weather = data.weather[0].main;
        })
      );
      favoriteList.value.forEach(item => {
        createFavorOverlay(item, true);
      });
    },
    (err) => {
      console.error(err);
    }
  );
});



// ======================
// Map
// ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
let map;
let marker;

const createMap = (lat, lon) => {
  const container = document.getElementById("map");
  const center = new kakao.maps.LatLng(lat, lon);
  map = new kakao.maps.Map(container, {
    center,
    level: 12
  });
  marker = new kakao.maps.Marker({
    position: center,
    map
  });
  addMapClickEvent();
};

const addMapClickEvent = () => {
  kakao.maps.event.addListener(map, "click", (e) => {
    favoriteOverlays.value.forEach(overlay => {
      overlay.div.classList.remove("pulse")
    });
    const lat = e.latLng.getLat();
    const lon = e.latLng.getLng();

    loadLocation(lat, lon);
  });
};

const updateMap = (lat, lon, showMarker) => {
  const center = new kakao.maps.LatLng(lat, lon);
  marker.setPosition(center);
  if (showMarker) {
    marker.setMap(map)
  } else {
    marker.setMap(null)
  }
  map.panTo(center);
};


// ======================
// Location / Weather Load
// ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
const loadLocation = async (lat, lon, showMarker = true) => {
  hasAddress.value = false;
  currentLat.value = lat;
  currentLon.value = lon;
  updateMap(lat, lon, showMarker);
  let currentWeather;
  try {
    [, currentWeather] = await Promise.all([
      getAddress(lat, lon),
      updateUI(lat, lon)
    ]);
  } catch (err) {
    console.error(err);
  }
  return currentWeather;
};


// ======================
// Weather API
// ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
const getWeather = async (lat, lon) => {
  try {
    const response = await axios.get(`${BASE_URL}/weather`, {
      params: {
        lat,
        lon,
        appid: API_KEY,
        units: 'metric',
        lang: 'kr'
      }
    });
    return response.data;
  } catch (error) {

    throw error;
  }
};

const getForecast = async (lat, lon) => {
  try {
    const response = await axios.get(`${BASE_URL}/forecast`, {
      params: {
        lat,
        lon,
        appid: API_KEY,
        units: 'metric',
        lang: 'kr'
      }
    });
    return response.data;
  } catch (error) {

    throw error;
  }
};


// ======================
// Update UI
// ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
const updateUI = async (lat, lon) => {
  const [currentWeather, forecastWeather] = await Promise.all([
    getWeather(lat, lon),
    getForecast(lat, lon)
  ]);
  weather.value = currentWeather?.weather?.[0]?.main || '';
  temp.value = currentWeather?.main?.temp ?? null;
  feelTemp.value = currentWeather?.main?.feels_like ?? null;
  humidity.value = currentWeather?.main?.humidity ?? null;
  wind.value = currentWeather?.wind?.speed ?? null;
  precipitation.value = currentWeather.rain?.["1h"] ?? 0;
  forecastData.value = forecastWeather.list.map(item => {
    const date = new Date(item.dt * 1000);
    return {
      time: date.toLocaleTimeString("ko-KR", {
        hour: "2-digit",
        minute: "2-digit",
        hour12: false
      }),
      weather: item.weather[0].main,
      temp: Math.round(item.main.temp),
      day: `${date.getMonth() + 1}/${date.getDate()}(${weeks[date.getDay()]})`
    }
  });

  return currentWeather;
};


// ======================
// Address
// ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
const getAddress = (lat, lon) => {
  return new Promise((resolve, reject) => {
    const geocoder = new kakao.maps.services.Geocoder();
    geocoder.coord2Address(lon, lat, (result, status) => {
      if (status === kakao.maps.services.Status.OK) {
        address.value = result[0].address.address_name;
        const splitAddress = address.value.split(" ");
        address.value = splitAddress.slice(0, 3).join(" ");
        hasAddress.value = true;
        resolve();
      } else {
        hasAddress.value = false;
        address.value = "주소를 찾을 수 없습니다.";
        reject("주소를 가져오지 못했습니다.");
      }
    });
  })
};


// ======================
// Favorite
// ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
const addFavorite = () => {
  if (!hasAddress.value) return;
  if (favoriteList.value.some(item => item.add === address.value)) {
    alert("이미 등록된 지역입니다.")
    return;
  }
  if (favoriteList.value.length >= 10) {
    alert("관심지역은 10개 까지 등록 가능합니다.");
    return;
  }
  marker.setMap(null);
  const favorData = {
    lat: currentLat.value,
    lon: currentLon.value,
    add: address.value,
    weather: weather.value,
    name: address.value
  }
  selectedFavorDiv.value = null;
  favoriteList.value.push(favorData);
  createFavorOverlay(favorData);
};

const deleteFavorite = (item) => {
  const currentOverlay = favoriteOverlays.value.find(list => list.add === item.add);
  favoriteList.value = favoriteList.value.filter(list => list.add !== item.add);
  favoriteOverlays.value = favoriteOverlays.value.filter(list => list.add !== item.add);
  currentOverlay.animation?.destroy();
  currentOverlay.overlay.setMap(null);
};

const deleteAllFavorites = () => {
  favoriteOverlays.value.forEach(overlay => {
    overlay.animation?.destroy();
    overlay.overlay.setMap(null);
  });
  favoriteList.value = [];
  favoriteOverlays.value = [];
  selectedFavorDiv.value = null;
};

const selectFavorite = async (favorData) => {
  selectedFavorDiv.value = favorData.add;
  favoriteOverlays.value.forEach(overlay => {
    overlay.div.classList.remove("pulse")
  });
  const overlayData = favoriteOverlays.value.find(
    item => item.add === favorData.add
  );
  if (!overlayData) return;
  overlayData.div.classList.add("pulse");
  const currentWeather = await loadLocation(
    favorData.lat,
    favorData.lon,
    false
  );
  const newWeather = currentWeather.weather[0].main;
  if (favorData.weather !== newWeather) {
    favorData.weather = newWeather;
    playAnimation(
      overlayData,
      circle
    );
    overlayData.animation.addEventListener("complete", () => {
      playAnimation(
        overlayData,
        weatherLottieMap[favorData.weather],
        true
      )
    }
    )
  }
  saveRecentList(
    favorData.add,
    favorData.lat,
    favorData.lon
  )
};

const createFavorOverlay = (favorData, isRestore = false) => {
  const pos = new kakao.maps.LatLng(
    favorData.lat,
    favorData.lon
  );
  const div = document.createElement("div");
  div.style.width = "30px";
  div.style.height = "30px";
  div.style.cursor = "pointer";
  const overlay = new kakao.maps.CustomOverlay({
    position: pos,
    content: div,
    clickable: true,
    xAnchor: 0.5,
    yAnchor: 0.5
  });
  const overlayData = {
    add: favorData.add,
    overlay,
    div,
    animation: null
  }
  if (isRestore) {
    playAnimation(overlayData, circle);

    overlayData.animation.addEventListener("complete", () => {
      playAnimation(
        overlayData,
        weatherLottieMap[favorData.weather],
        true
      );
    });
  } else {
    playAnimation(
      overlayData,
      circle
    );
    overlayData.animation.addEventListener("complete", () => {
      playAnimation(overlayData, weatherLottieMap[favorData.weather], true)
      overlayData.div.classList.add("pulse");
    }
    )
  }
  overlay.setMap(map);
  favoriteOverlays.value.push(overlayData);
  div.addEventListener("click", () => {
    selectFavorite(favorData);
  })
};

const playAnimation = (target, animationData, loop = false) => {
  target.animation?.destroy();
  target.div.innerHTML = "";
  target.animation = lottie.loadAnimation({
    container: target.div,
    renderer: "svg",
    loop,
    autoplay: true,
    animationData
  });
  return target.animation;
};

</script>

<template>
  <div class="background">
    <div class="blob blob1"></div>
    <div class="blob blob2"></div>
    <div class="blob blob3"></div>
  </div>
  <div class="wrap">
    <div class="left">
      <div class="stickyHeader">
        <h1>현재 날씨 및 예보 앱</h1>
        <CurrentLocation :favoriteList="favoriteList" :address="address" :hasAddress="hasAddress"
          @addFavorite="addFavorite" />
        <SearchBox :recentList="recentList" @loadLocation="loadLocation" @searchLocation="searchLocation"
          @delRecentList="delRecentList" />
      </div>
      <FavoriteList :favoriteList="favoriteList" :selectedFavorDiv="selectedFavorDiv" @deleteFavorite="deleteFavorite"
        @deleteAllFavorites="deleteAllFavorites" @select="selectedFavorDiv = $event"
        @clearSelect="selectedFavorDiv = null" @selectFavorite="selectFavorite" />
    </div>
    <div class="right">
      <div class="mapWrap">
        <div id="map"></div>
        <WeatherBox :weatherInfo="weatherInfo" />
      </div>
      <Forecast :forecastData="forecastData" :weatherLottieMap="weatherLottieMap" />
    </div>
  </div>
</template>