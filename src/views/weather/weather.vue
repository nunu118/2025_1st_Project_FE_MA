<script setup>
import { ref, onMounted, computed, watch } from 'vue';
import { getWeather, getNickName, saveGpsAddress } from '@/services/weather/weatherHomeService';
import DailyWeather from '@/components/weather/DailyWeather.vue';
import Location from '@/components/weather/Location.vue';
import { useWeatherStore } from '@/stores/weatherStore';

const weatherStore = useWeatherStore();
const weather = ref(null);
const open = ref(false);
const nickName = ref('');
const dialog = ref({
  daily: false,
  location: false,
});
const today = new Date();
const year = today.getFullYear();
const month = today.getMonth() + 1;
const day = today.getDate();
const days = ['일', '월', '화', '수', '목', '금', '토'];
const weekday = days[today.getDay()];
const nowDate = `${year}년 ${month}월 ${day}일(${weekday})`;

const openDialog = (type) => {
  dialog.value[type] = true;
};

const processGpsLocation = async (lat, lon) => {
  try {
    const locationReq = {
      title: '내 위치(GPS)',
      roadAddress: 'GPS 자동 수집 위치',
      parcelAddress: 'GPS 자동 수집 위치',
      lat: lat,
      lon: lon,
      isSelected: true
    };

    await saveGpsAddress(locationReq);
    console.log('GPS 위치 저장 성공!');

    sessionStorage.setItem('gpsUpdated', 'true');
    window.location.reload(); 
  } catch (err) {
    console.error('GPS 위치 저장 실패:', err);
  }
};

// GPS 위치 정보 수집 및 처리 함수
const checkAndRequestGPS = () => {
  if ('geolocation' in navigator) {
    if (confirm('현재 설정된 위치가 없습니다.\n현재 위치(GPS)의 날씨 정보를 불러오시겠습니까?')) {
      navigator.geolocation.getCurrentPosition(
        async (position) => {
          const lat = position.coords.latitude;
          const lon = position.coords.longitude;
          console.log('GPS 좌표 수집:', lat, lon);

          await processGpsLocation(lat, lon);
        },
        (error) => {
          console.warn('GPS 위치 접근 거부/실패:', error);
          alert('GPS 위치 접근이 거부되었거나 실패했습니다. 위치를 수동으로 설정해주세요.');},
        { timeout: 10000 }
      );
    }
  } else {
    alert('이 브라우저에서는 GPS(Geolocation)를 지원하지 않습니다.');
  }
};

const LocalWeather = async () => {
  try {
    const res = await getWeather();
    console.log('Weather res.data', res.data);
    weather.value = res.data;

    if (weather.value && weather.value.ncstPty !== '없음') {
      weather.value.villageSky = weather.value.ncstPty;
    }

    // ⭕ 서울특별시(기본값)이고, 이번 세션에서 아직 GPS 업데이트를 안 한 경우 팝업 실행
    if (weather.value && weather.value.localName === '서울특별시' && !sessionStorage.getItem('gpsUpdated')) {
      checkAndRequestGPS();
    }
  } catch (err) {
    console.error('날씨 데이터 조회 실패:', err);
  }
};

// 한줄 알림
const memberNickName = async () => {
  try {
    const res = await getNickName();
    nickName.value = res.data?.memberNick || '';
  } catch (err) {
    console.error('닉네임 조회 실패:', err);
  }
};

const popMessage = computed(() => {
  if (!weather.value) return '';
  const pop = weather.value.villagePop;
  const per = '오늘은 비올 확률이 ' + weather.value.villagePop + '% !!';
  const sky = weather.value.villageSky;
  if ((pop < 10 && sky === '맑음') || (pop < 10 && sky === '구름 많음')) {
    return '오늘의 날씨는 ' + sky + '이네요! 즐거운 하루 보내세요.';
  } else if (
    pop < 30 &&
    (sky === '맑음' || sky === '구름 많음' || sky === '흐림')
  ) {
    return per + '걱정 되신다면 우산을 챙기길 추천해요!';
  } else if (pop < 50 && (sky === '흐림' || sky === '비')) {
    return per + '비가 올 수도 있으니 휴대하기 편한 우산 챙기길 추천해요!';
  } else if (pop < 90 && (sky === '흐림' || sky === '비')) {
    return per + '우산 챙기셨나요? 우산 챙겨가세요!';
  } else if (pop > 90 || sky === '비') {
    return per + '우산을 꼭 챙기고 빗길 조심하세요!';
  } else {
    return '오늘 하루도 화이팅!';
  }
});

const toggleMenu = () => (open.value = !open.value);

// 이모트 및 사진
const skyEmojiList = {
  맑음: '☀️',
  흐림: '☁️',
  '구름 많음': '🌤️',
  비: '🌧️',
  눈: '❄️',
  '비/눈': '🌨️',
};

const skyEmoji = computed(() => {
  return (
    skyEmojiList[weather.value?.ncstPty] ||
    skyEmojiList[weather.value?.villageSky]
  );
});

const dayTimes = computed(() => {
  const hour = new Date().getHours();
  if (hour > 6 && hour < 19) return 'morning';
  else if (hour < 22) return 'evening';
  else return 'night';
});

const backgroundImg = {
  '맑음-morning': 'url(/image/weather/clear_morning.jpg)',
  '맑음-evening': 'url(/image/weather/clear_afternoon.jpg)',
  '맑음-night': 'url(/image/weather/clear_night.jpg)',
  '구름 많음-morning': 'url(/image/weather/littlecloudy_morning.jpg)',
  '구름 많음-evening': 'url(/image/weather/littlecloudy.jpg)',
  '구름 많음-night': 'url(/image/weather/littlecloudy_night.jpg)',
  '흐림-morning': 'url(/image/weather/cloudy.jpg)',
  '흐림-evening': 'url(/image/weather/cloudy.jpg)',
  '흐림-night': 'url(/image/weather/cloudy_night.jpg)',
  '비-morning': 'url(/image/weather/rain.jpg)',
  '비-evening': 'url(/image/weather/rain.jpg)',
  '비-night': 'url(/image/weather/rain.jpg)',
  default: 'url(/image/loading.gif)',
};

const weatherBackground = computed(() => {
  const sky = weather.value?.villageSky || '';
  const time = dayTimes.value;
  return backgroundImg[`${sky}-${time}`] || backgroundImg.default;
});

onMounted(async () => {
  memberNickName();
  await LocalWeather();
});

watch(
  () => weatherStore.refresh,
  async () => {
    await LocalWeather();
  }
);
</script>

<template>
  <div class="weather-layout-container">
    <div class="weather-alert">
      <div v-if="weather">
        <strong>{{
          nickName === '' || nickName === undefined ? '' : nickName + '님~'
        }}</strong
        >{{ popMessage }}
      </div>
      <div v-else>로딩중. . .</div>
    </div>
    
    <!-- PC날씨 정보 헤더 -->
    <div class="header flex justify-between items-center w-full px-4 pt-2">
      <span class="live px-4 py-1 text-white font-semibold text-sm">
        실시간 날씨 정보
      </span>
      <v-menu v-model="open" offset-y location="bottom end">
        <template v-slot:activator="{ props }">
          <button v-bind="props" class="menu px-2 py-1 text-sm font-bold">
            ☰ 날씨 메뉴
          </button>
        </template>
        <v-list class="menu-list">
          <v-list-item @click="openDialog('daily')">시간별 날씨</v-list-item>
          <v-list-item @click="openDialog('location')">지역 변경</v-list-item>
        </v-list>
      </v-menu>

      <v-dialog v-model="dialog.daily" max-width="1000" min-height="100">
        <v-card>
          <v-card-title class="text-h8">{{ nowDate }} 날씨</v-card-title>
          <v-card-text>
            <DailyWeather />
          </v-card-text>
        </v-card>
      </v-dialog>

      <v-dialog v-model="dialog.location" max-width="1000" min-height="200">
        <v-card>
          <v-card-title class="text-h8">지역 검색 및 목록</v-card-title>
          <v-card-text>
            <Location @close="dialog.location = false" />
          </v-card-text>
        </v-card>
      </v-dialog>
    </div>
    
    <!-- 폰화면 정보 박스 -->
    <div
      v-if="weather"
      class="weather-card d-flex d-sm-none"
      :style="{ backgroundImage: weatherBackground }"
    >
      <div class="weather-location">{{ weather.localName }}</div>
      <div class="warp">
        <div class="weather-icon">{{ skyEmoji }}</div>
        <div class="temperature">
          {{ weather.ncstTem === undefined ? '' : weather.ncstTem + '℃' }}
        </div>
      </div>
    </div>

    <!-- PC날씨 정보 박스 -->
    <div class="d-none d-sm-flex">
      <div class="weather-card" :style="{ backgroundImage: weatherBackground }">
        <div v-if="weather" class="weather-content">
          <div class="weather-left">
            <div class="weather-location">
              {{ weather.localName }}
            </div>
            <div class="condition">
              {{ weather.villageSky }}
              {{ weather.villageSky === '비' ? weather.ncstRh1 + '(mm)' : '' }}
            </div>
          </div>

          <div class="weather-right">
            <div class="warp">
              <div class="weather-icon">{{ skyEmoji }}</div>
              <div class="temperature">
                {{ weather.ncstTem === undefined ? '' : weather.ncstTem + '℃' }}
              </div>
            </div>
            <div class="max_min_temperature">
              {{
                weather.villageTmn === undefined
                  ? ''
                  : '최저 ' +
                    weather.villageTmn +
                    '° / 최고 ' +
                    weather.villageTmx +
                    '°'
              }}
            </div>
            <div class="humidity">
              {{
                weather.ncstReh === undefined
                  ? ''
                  : '습도' + weather.ncstReh + '%'
              }}
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style lang="scss" scoped>
/* 기존 style 동일 유지 */
.weather-alert {
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 0.6rem 1rem;
  margin: 1rem;
  max-width: 100%;
  font-size: 1.2rem;
  border: 1px solid #ccc;
  border-radius: 12px;
  box-shadow: 1px 1px 5px rgba(0, 0, 0, 0.2);
  background-color: #fff;
  color: #333;
  line-height: 1.4;
  user-select: none;

  strong {
    font-weight: bold;
    margin-right: 0.3rem;
  }
}
.header {
  display: flex;
  align-items: center;
  position: relative;

  .live {
    background-color: #3bbeff;
    border-radius: 16px 16px 0 0;
    gap: 0;
    user-select: none;
  }
}

.menu {
  color: #3bbeff;
  background-color: transparent;
  border: none;
  margin-left: auto;
}
.menu-list {
  margin-top: 4px;
  color: #ccc;
  padding: 0 1rem 0.1rem;
  border-radius: 16px;
  &:hover {
    background-color: transparent;
  }
}
.weather-card {
  background-size: cover;
  background-position: center;
  border-radius: 0 16px 16px 16px;
  padding: 1rem;
  margin: 0px 1rem;
  height: 11rem;
  color: white;
  width: 100%;
  box-shadow: 1px 1px 4px #838383;
  box-sizing: border-box;
  text-shadow: 1px 1px 4px rgba(0, 0, 0, 0.5);
  user-select: none;
  text-overflow: ellipsis;
  white-space: nowrap;

  .weather-content {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 2rem;
    flex-wrap: wrap;
  }

  .weather-left {
    display: flex;
    flex-direction: column;
    gap: 2rem;

    .weather-location {
      font-size: 2.4rem;
      font-weight: bold;
      text-overflow: ellipsis;
      overflow: hidden;
      white-space: nowrap;
    }

    .condition {
      font-size: 1.2rem;
    }
  }

  .warp {
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }

  .weather-right {
    display: flex;
    flex-direction: column;
    align-items: flex-end;
    gap: 0.3rem;

    .weather-icon {
      font-size: 3rem;
    }

    .temperature {
      font-size: 2rem;
      font-weight: bold;
    }

    .max_min_temperature,
    .humidity {
      font-size: 1rem;
    }
  }
}
@media (max-width: 981px) {
  .weather-location {
    font-size: 2rem !important;
  }
}
@media (max-width: 935px) {
  .weather-alert {
    font-size: 1rem;
  }
}
@media (max-width: 873px) {
  .weather-location {
    font-size: 1.8rem !important;
  }
}
@media (max-width: 802px) {
  .weather-alert {
    font-size: 0.8rem;
  }
}
@media (max-width: 790px) {
  .condition {
    font-size: 1.2rem !important;
  }
  .weather-icon {
    font-size: 2.8rem !important;
  }

  .temperature {
    font-size: 1.8rem !important;
  }
}
@media (max-width: 686px) {
  .weather-location {
    font-size: 1.6rem !important;
  }
  .condition {
    font-size: 1rem !important;
  }
}
@media (max-width: 679px) {
  .weather-location {
    font-size: 1.4rem !important;
  }
  .condition {
    font-size: 1rem !important;
  }
  .weather-icon {
    font-size: 2.6rem !important;
  }

  .temperature {
    font-size: 1.6rem !important;
  }
  .weather-alert {
    font-size: 0.6rem;
  }
}
@media (max-width: 601px) {
  .weather-location {
    font-size: 1.2rem !important;
  }
}
@media (max-width: 573px) {
  .weather-content {
    gap: 0 !important;
  }
  .weather-card {
    height: 9rem;
  }
  .weather-location {
    font-size: 1rem !important;
  }
  .weather-right {
    .weather-icon {
      font-size: 1.8rem !important;
    }
    .temperature {
      font-size: 1.2rem !important;
    }
  }
}
@media (max-width: 534px) {
  .weather-alert {
    font-size: 0.6rem;
  }
}
@media (max-width: 522px) {
  .header {
    font-size: 10px;
  }
}
@media (max-width: 498px) {
  .weather-card {
    width: 93% !important;
  }
  .weather-content {
    gap: 0 !important;
    flex-flow: nowrap !important;
  }
  .weather-right {
    .max_min_temperature,
    .humidity {
      font-size: 0.8rem !important;
    }
  }
}
@media (max-width: 479px) {
  .weather-card {
    flex-direction: column;
  }
}
@media (max-width: 446px) {
  .weather-card {
    height: 8rem;
  }
  .weather-content {
    flex-wrap: nowrap !important;
  }
  .weather-right {
    .weather-icon {
      font-size: 1.4rem !important;
    }
    .temperature {
      font-size: 1rem !important;
    }
  }
}
@media (min-width: 480px) {
  .d-sm-flex {
    display: flex !important;
  }
  .d-sm-none {
    display: none !important;
  }
}
@media (min-width: 768px) {
  .d-md-flex {
    display: flex !important;
  }
  .d-md-none {
    display: none !important;
  }
}
@media (min-width: 1024px) {
  .d-lg-flex {
    display: flex !important;
  }
  .d-lg-none {
    display: none !important;
  }
}
</style>