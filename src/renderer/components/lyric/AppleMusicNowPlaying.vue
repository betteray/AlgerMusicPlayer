<template>
  <div
    id="amnp-root"
    :class="{ 'hide-cursor': hideCursor }"
    @dblclick="emit('exit')"
    @mousemove="hideCursorSoon"
  >
    <canvas id="amnp-bg" ref="canvasRef" />
    <div id="amnp-bg-dim" />
    <div id="amnp-layout">
      <div id="amnp-left">
        <div id="amnp-art" ref="artRef" :class="{ paused: !isPlaying }" :style="artStyle" />
        <div id="amnp-meta">
          <div id="amnp-meta-row">
            <div style="min-width: 0; flex: 1">
              <div id="amnp-title">{{ title }}</div>
              <div id="amnp-artist">{{ artistLine }}</div>
            </div>
            <button id="amnp-heart" :class="{ on: isFavorite }" @click.stop="toggleFavorite">
              <svg
                :width="22"
                :height="22"
                viewBox="0 0 24 24"
                fill="currentColor"
                xmlns="http://www.w3.org/2000/svg"
                v-html="isFavorite ? ICONS.starOn : ICONS.star"
              />
            </button>
          </div>
          <div id="amnp-progress-wrap">
            <div
              id="amnp-bar"
              ref="barRef"
              @click="seekFromEvent($event, true)"
              @mousedown="onBarMouseDown"
            >
              <div id="amnp-bar-track">
                <div id="amnp-bar-inner" :style="barInnerStyle" />
              </div>
            </div>
            <div id="amnp-times">
              <span>{{ formatTime(progress) }}</span>
              <span>-{{ formatTime(Math.max(0, duration - progress)) }}</span>
            </div>
          </div>
          <div id="amnp-controls">
            <button class="edge" :class="{ on: shuffleOn }" @click.stop="toggleShuffle">
              <svg
                width="27"
                height="27"
                viewBox="0 0 56 56"
                fill="currentColor"
                xmlns="http://www.w3.org/2000/svg"
                v-html="ICONS.shuffle"
              />
            </button>
            <button @click.stop="handlePrev">
              <svg
                width="32"
                height="32"
                viewBox="22 42 86 50"
                fill="currentColor"
                xmlns="http://www.w3.org/2000/svg"
                v-html="ICONS.prev"
              />
            </button>
            <button class="play" @click.stop="playMusicEvent">
              <svg
                :width="isPlaying ? 26 : 28"
                :height="isPlaying ? 26 : 28"
                viewBox="0 0 38 38"
                fill="currentColor"
                xmlns="http://www.w3.org/2000/svg"
                v-html="isPlaying ? ICONS.pause : ICONS.play"
              />
            </button>
            <button @click.stop="handleNext">
              <svg
                width="32"
                height="32"
                viewBox="26 42 86 50"
                fill="currentColor"
                xmlns="http://www.w3.org/2000/svg"
                v-html="ICONS.next"
              />
            </button>
            <button
              class="edge"
              :class="{ on: repeatOn, 'repeat-one': repeatOn }"
              @click.stop="toggleRepeat"
            >
              <svg
                width="27"
                height="27"
                viewBox="0 0 56 56"
                fill="currentColor"
                xmlns="http://www.w3.org/2000/svg"
                v-html="ICONS.repeat"
              />
            </button>
          </div>
        </div>
      </div>
      <div id="amnp-right">
        <div v-if="!lines.length" id="amnp-empty-lyrics">暂无歌词</div>
        <div v-else id="amnp-lyrics-viewport" ref="viewportRef">
          <div id="amnp-lyrics">
            <div
              v-for="(line, index) in lines"
              :key="index"
              :ref="(el) => setLineRef(el, index)"
              class="amnp-line"
              :class="{
                active: index === activeIndex,
                empty: !line.text
              }"
              @click="seekLine(line)"
            >
              {{ line.text || ' ' }}
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue';

import { artistList, lrcArray, lrcTimeArray, nowTime, playMusic } from '@/hooks/MusicHook';
import { useFavorite } from '@/hooks/useFavorite';
import { usePlaybackControl } from '@/hooks/usePlaybackControl';
import { audioService } from '@/services/audioService';
import { usePlayerStore } from '@/store/modules/player';
import { getImgUrl } from '@/utils';

type LyricLine = {
  startTime: number | null;
  text: string;
};

type Spring = {
  setTarget: (next: number, delay?: number) => void;
  snap: (next: number) => void;
  update: (dt: number, now?: number) => number;
  current: () => number;
};

const emit = defineEmits<{
  exit: [];
}>();

const ICONS = {
  play: '<path d="M5.80762 32.4896V5.4925C5.80762 4.305 6.12305 3.41438 6.75391 2.82063C7.38477 2.22688 8.13932 1.93 9.01758 1.93C9.78451 1.93 10.5391 2.14029 11.2812 2.56086L33.7324 15.6605C34.5859 16.1553 35.223 16.6562 35.6436 17.1634C36.0641 17.6582 36.2744 18.2705 36.2744 19.0003C36.2744 19.7054 36.0641 20.3177 35.6436 20.8372C35.223 21.3444 34.5859 21.8392 33.7324 22.3216L11.2812 35.4212C10.5391 35.8542 9.78451 36.0706 9.01758 36.0706C8.13932 36.0706 7.38477 35.7676 6.75391 35.1614C6.12305 34.5677 5.80762 33.6771 5.80762 32.4896Z"/>',
  pause:
    '<path d="M8.46953 37C7.37801 37 6.56603 36.7271 6.03359 36.1814C5.51445 35.6489 5.25488 34.8502 5.25488 33.7854V4.21464C5.25488 3.14975 5.52111 2.35108 6.05355 1.81864C6.59931 1.27288 7.40463 1 8.46953 1H13.3813C14.4329 1 15.2249 1.27288 15.7574 1.81864C16.3031 2.35108 16.576 3.14975 16.576 4.21464V33.7854C16.576 34.8502 16.3031 35.6489 15.7574 36.1814C15.2249 36.7271 14.4329 37 13.3813 37H8.46953ZM24.6426 37C23.5644 37 22.759 36.7271 22.2266 36.1814C21.6942 35.6489 21.4279 34.8502 21.4279 33.7854V4.21464C21.4279 3.14975 21.6942 2.35108 22.2266 1.81864C22.7724 1.27288 23.5777 1 24.6426 1H29.5544C30.6193 1 31.4179 1.27288 31.9504 1.81864C32.4828 2.35108 32.7491 3.14975 32.7491 4.21464V33.7854C32.7491 34.8502 32.4828 35.6489 31.9504 36.1814C31.4179 36.7271 30.6193 37 29.5544 37H24.6426Z"/>',
  prev: '<path d="M72 60.0717C68.062 62.3453 66.0931 63.4821 65.4323 64.9662C64.8559 66.2608 64.8559 67.7391 65.4323 69.0336C66.0931 70.5177 68.062 71.6545 72 73.9281L93 86.0525C96.938 88.326 98.9069 89.4628 100.523 89.293C101.932 89.1449 103.212 88.4057 104.045 87.2593C105 85.945 105 83.6714 105 79.1243V54.8755C105 50.3284 105 48.0548 104.045 46.7405C103.212 45.5941 101.932 44.8549 100.523 44.7068C98.9069 44.537 96.938 45.6738 93 47.9473L72 60.0717Z"/><path d="M32 60.0717C28.062 62.3453 26.0931 63.4821 25.4323 64.9662C24.8559 66.2608 24.8559 67.7391 25.4323 69.0336C26.0931 70.5177 28.062 71.6545 32 73.9281L53 86.0525C56.938 88.326 58.9069 89.4628 60.5226 89.293C61.9319 89.1449 63.2122 88.4057 64.0451 87.2593C65 85.945 65 83.6714 65 79.1243V54.8755C65 50.3284 65 48.0548 64.0451 46.7405C63.2122 45.5941 61.9319 44.8549 60.5226 44.7068C58.9069 44.537 56.938 45.6738 53 47.9473L32 60.0717Z"/>',
  next: '<path d="M62 60.0717C65.938 62.3453 67.9069 63.4821 68.5677 64.9662C69.1441 66.2608 69.1441 67.7391 68.5677 69.0336C67.9069 70.5177 65.938 71.6545 62 73.9281L41 86.0525C37.062 88.326 35.0931 89.4628 33.4774 89.293C32.0681 89.1449 30.7878 88.4057 29.9549 87.2593C29 85.945 29 83.6714 29 79.1243V54.8755C29 50.3284 29 48.0548 29.9549 46.7405C30.7878 45.5941 32.0681 44.8549 33.4774 44.7068C35.0931 44.537 37.062 45.6738 41 47.9473L62 60.0717Z"/><path d="M102 60.0717C105.938 62.3453 107.907 63.4821 108.568 64.9662C109.144 66.2608 109.144 67.7391 108.568 69.0336C107.907 70.5177 105.938 71.6545 102 73.9281L81 86.0525C77.062 88.326 75.0931 89.4628 73.4774 89.293C72.0681 89.1449 70.7878 88.4057 69.9549 87.2593C69 85.945 69 83.6714 69 79.1243V54.8755C69 50.3284 69 48.0548 69.9549 46.7405C70.7878 45.5941 72.0681 44.8549 73.4774 44.7068C75.0931 44.537 77.062 45.6738 81 47.9473L102 60.0717Z"/>',
  shuffle:
    '<path d="M10.624 36.3125C10.624 35.75 10.8218 35.2754 11.2173 34.8887C11.6216 34.4932 12.1094 34.2954 12.6807 34.2954H15.4756C16.3896 34.2954 17.1455 34.1372 17.7432 33.8208C18.3496 33.5044 18.9341 32.9946 19.4966 32.2915L27.3936 22.3379C28.3955 21.0811 29.4282 20.2285 30.4917 19.7803C31.5552 19.332 32.79 19.1079 34.1963 19.1079H36.4243V16.1548C36.4243 15.6714 36.5605 15.2935 36.833 15.021C37.1055 14.7397 37.479 14.5991 37.9536 14.5991C38.1821 14.5991 38.3843 14.6343 38.5601 14.7046C38.7446 14.7749 38.9072 14.8672 39.0479 14.9814L44.8223 19.8857C45.1826 20.1846 45.3628 20.5493 45.3628 20.98C45.3628 21.4106 45.1826 21.7754 44.8223 22.0742L39.0479 26.9917C38.9072 27.106 38.7446 27.2026 38.5601 27.2817C38.3843 27.3521 38.1821 27.3872 37.9536 27.3872C37.479 27.3872 37.1055 27.2466 36.833 26.9653C36.5605 26.6841 36.4243 26.3018 36.4243 25.8184V23.1421H33.9194C33.3218 23.1421 32.8076 23.2036 32.377 23.3267C31.9551 23.4497 31.564 23.6562 31.2036 23.9463C30.8521 24.2275 30.4829 24.6143 30.0962 25.1064L21.606 35.7061C20.8853 36.6113 20.0986 37.2749 19.2461 37.6968C18.3936 38.1099 17.3389 38.3164 16.082 38.3164H12.6807C12.1094 38.3164 11.6216 38.123 11.2173 37.7363C10.8218 37.3496 10.624 36.875 10.624 36.3125ZM10.624 21.125C10.624 20.5625 10.8218 20.0879 11.2173 19.7012C11.6216 19.3057 12.1094 19.1079 12.6807 19.1079H15.7261C16.9829 19.1079 18.0947 19.3188 19.0615 19.7407C20.0371 20.1538 20.8853 20.8174 21.606 21.7314L30.0435 32.2783C30.5972 32.9727 31.1992 33.4824 31.8496 33.8076C32.5 34.1328 33.291 34.2954 34.2227 34.2954H36.4243V31.5664C36.4243 31.083 36.5605 30.7007 36.833 30.4194C37.1055 30.1382 37.479 29.9976 37.9536 29.9976C38.1821 29.9976 38.3843 30.0371 38.5601 30.1162C38.7446 30.1865 38.9072 30.2832 39.0479 30.4062L44.8223 35.2974C45.1826 35.5962 45.3628 35.9609 45.3628 36.3916C45.3628 36.8223 45.1826 37.187 44.8223 37.4858L39.0479 42.3901C38.9072 42.5132 38.7446 42.6099 38.5601 42.6802C38.3843 42.7593 38.1821 42.7988 37.9536 42.7988C37.479 42.7988 37.1055 42.6582 36.833 42.377C36.5605 42.0957 36.4243 41.7134 36.4243 41.23V38.3164H34.1699C32.9043 38.3164 31.7222 38.1011 30.6235 37.6704C29.5249 37.231 28.5625 36.4927 27.7363 35.4556L19.4966 25.146C18.9341 24.4429 18.2925 23.9331 17.5718 23.6167C16.8599 23.3003 16.0381 23.1421 15.1064 23.1421H12.6807C12.1094 23.1421 11.6216 22.9443 11.2173 22.5488C10.8218 22.1533 10.624 21.6787 10.624 21.125Z"/>',
  repeat:
    '<path d="M14.2495 28.9956C13.6519 28.9956 13.1465 28.7891 12.7334 28.376C12.3203 27.9541 12.1138 27.4531 12.1138 26.873V25.3438C12.1138 23.832 12.4565 22.5312 13.1421 21.4414C13.8276 20.3516 14.8076 19.5166 16.082 18.9365C17.3564 18.3477 18.877 18.0532 20.6436 18.0532H30.3599V15.4033C30.3599 14.9111 30.4961 14.5288 30.7686 14.2563C31.041 13.9751 31.4146 13.8345 31.8892 13.8345C32.1177 13.8345 32.3198 13.874 32.4956 13.9531C32.6714 14.0234 32.8296 14.1113 32.9702 14.2168L38.7578 19.1343C39.1182 19.4331 39.2939 19.7979 39.2852 20.2285C39.2852 20.6504 39.1094 21.0107 38.7578 21.3096L32.9702 26.2271C32.8296 26.3501 32.6714 26.4468 32.4956 26.5171C32.3198 26.5874 32.1177 26.6226 31.8892 26.6226C31.4146 26.6226 31.041 26.4819 30.7686 26.2007C30.4961 25.9194 30.3599 25.5415 30.3599 25.0669V22.1929H20.459C19.1846 22.1929 18.1826 22.5269 17.4531 23.1948C16.7236 23.8628 16.3589 24.7812 16.3589 25.9502V26.873C16.3589 27.4531 16.1523 27.9541 15.7393 28.376C15.3262 28.7891 14.8296 28.9956 14.2495 28.9956ZM41.7505 26.7017C42.3306 26.7017 42.8271 26.9082 43.2402 27.3213C43.6621 27.7344 43.873 28.2354 43.873 28.8242V30.3535C43.873 31.8652 43.5303 33.166 42.8447 34.2559C42.1592 35.3457 41.1792 36.1851 39.9048 36.7739C38.6304 37.354 37.1055 37.644 35.3301 37.644H25.627V40.2676C25.627 40.751 25.4907 41.1333 25.2183 41.4146C24.9458 41.6958 24.5723 41.8364 24.0977 41.8364C23.8691 41.8364 23.6626 41.7969 23.478 41.7178C23.3022 41.6475 23.1484 41.5552 23.0166 41.4409L17.2158 36.5366C16.873 36.2466 16.6973 35.8862 16.6885 35.4556C16.6885 35.0249 16.8643 34.6558 17.2158 34.3481L23.0166 29.4307C23.1484 29.3164 23.3022 29.2241 23.478 29.1538C23.6626 29.0835 23.8691 29.0483 24.0977 29.0483C24.5723 29.0483 24.9458 29.189 25.2183 29.4702C25.4907 29.7427 25.627 30.125 25.627 30.6172V33.4912H35.5278C36.8022 33.4912 37.8042 33.1616 38.5337 32.5024C39.2632 31.8345 39.6279 30.916 39.6279 29.7471V28.8242C39.6279 28.2354 39.8301 27.7344 40.2344 27.3213C40.6475 26.9082 41.1528 26.7017 41.7505 26.7017Z"/>',
  star: '<path fill="none" stroke="currentColor" stroke-width="1.7" stroke-linejoin="round" stroke-linecap="round" d="M12 2.8 14.07 9.15 20.75 9.16 15.35 13.09 17.41 19.44 12 15.52 6.59 19.44 8.65 13.09 3.25 9.16 9.93 9.15Z"/>',
  starOn:
    '<path d="M12 2.8 14.07 9.15 20.75 9.16 15.35 13.09 17.41 19.44 12 15.52 6.59 19.44 8.65 13.09 3.25 9.16 9.93 9.15Z"/>'
};

const playerStore = usePlayerStore();
const { isFavorite, toggleFavorite } = useFavorite();
const { isPlaying, playMusicEvent, handleNext, handlePrev } = usePlaybackControl();

const canvasRef = ref<HTMLCanvasElement | null>(null);
const artRef = ref<HTMLElement | null>(null);
const barRef = ref<HTMLElement | null>(null);
const viewportRef = ref<HTMLElement | null>(null);
const lineRefs = ref<Array<HTMLElement | null>>([]);
const hideCursor = ref(false);
const dragProgress = ref<number | null>(null);

let cursorTimer: ReturnType<typeof setTimeout> | null = null;
let bgRaf = 0;
let lyricRaf = 0;
let bgObserver: ResizeObserver | null = null;
const springs: Spring[] = [];
let lastLyricIndex = -1;
const activeIndexRef = { current: -1 };

const cover = computed(() => getImgUrl(playMusic.value?.picUrl, '800y800'));
const artStyle = computed(() => ({
  backgroundImage: cover.value ? `url("${cover.value}")` : 'none'
}));
const barInnerStyle = computed(() => ({
  width: `${progressRatio.value * 100}%`
}));
const title = computed(() => playMusic.value?.name || '');
const albumName = computed(
  () =>
    playMusic.value?.al?.name ||
    playMusic.value?.album?.name ||
    playMusic.value?.song?.album?.name ||
    ''
);
const artistName = computed(() => {
  const names = artistList.value?.map((item) => item.name).filter(Boolean) || [];
  return names.join(' / ') || playMusic.value?.ar?.[0]?.name || '';
});
const artistLine = computed(() =>
  albumName.value ? `${artistName.value} — ${albumName.value}` : artistName.value
);
const duration = computed(() => {
  const sound = audioService.getCurrentSound();
  const fromAudio = sound?.duration;
  if (fromAudio && Number.isFinite(fromAudio) && fromAudio > 0) return fromAudio;
  const song = playMusic.value;
  const ms = song?.duration || song?.dt || 0;
  return ms > 1000 ? ms / 1000 : 0;
});
const progress = computed(() => dragProgress.value ?? nowTime.value);
const progressRatio = computed(() => {
  if (!duration.value) return 0;
  return Math.min(1, Math.max(0, progress.value / duration.value));
});
const shuffleOn = computed(() => playerStore.playMode === 2);
const repeatOn = computed(() => playerStore.playMode === 1);
const lines = computed<LyricLine[]>(() => {
  if (!lrcArray.value.length) return [];
  return lrcArray.value.map((line, index) => {
    const start = lrcTimeArray.value[index];
    const hasTime = typeof start === 'number' && start >= 0;
    return {
      startTime: hasTime ? Math.round(start * 1000) : null,
      text: line.text || ''
    };
  });
});
const activeIndex = computed(() => {
  const list = lines.value;
  if (!list.length || list[0].startTime == null) return -1;
  const currentMs = nowTime.value * 1000;
  let index = 0;
  for (let i = 0; i < list.length; i++) {
    if (list[i].startTime != null && currentMs >= (list[i].startTime as number)) index = i;
    else break;
  }
  return index;
});
activeIndexRef.current = activeIndex.value;
watch(activeIndex, (value) => {
  activeIndexRef.current = value;
});

const formatTime = (sec: number) => {
  if (!Number.isFinite(sec) || sec < 0) sec = 0;
  const total = Math.floor(sec);
  const minutes = Math.floor(total / 60);
  const seconds = total % 60;
  return `${minutes}:${seconds.toString().padStart(2, '0')}`;
};

const setLineRef = (el: unknown, index: number) => {
  lineRefs.value[index] = (el as HTMLElement) || null;
};

const hideCursorSoon = () => {
  hideCursor.value = false;
  if (cursorTimer) clearTimeout(cursorTimer);
  cursorTimer = setTimeout(() => {
    hideCursor.value = true;
  }, 2000);
};

const applyPlayMode = (mode: number) => {
  const wasRandom = playerStore.playMode === 2;
  playerStore.playMode = mode;
  if (mode === 2 && !wasRandom) playerStore.shufflePlayList();
  if (mode !== 2 && wasRandom) playerStore.restoreOriginalOrder();
};

const toggleShuffle = () => {
  applyPlayMode(playerStore.playMode === 2 ? 0 : 2);
};

const toggleRepeat = () => {
  applyPlayMode(playerStore.playMode === 1 ? 0 : 1);
};

const seekTo = (next: number) => {
  audioService.seek(next);
  nowTime.value = next;
};

const seekFromEvent = (event: MouseEvent, commit = false) => {
  const bar = barRef.value;
  if (!bar || !duration.value) return 0;
  const rect = bar.getBoundingClientRect();
  const ratio = Math.min(1, Math.max(0, (event.clientX - rect.left) / rect.width));
  const next = ratio * duration.value;
  dragProgress.value = next;
  if (commit) {
    seekTo(next);
    dragProgress.value = null;
  }
  return next;
};

const onBarMouseDown = (event: MouseEvent) => {
  seekFromEvent(event);
  const move = (e: MouseEvent) => seekFromEvent(e);
  const up = (e: MouseEvent) => {
    seekFromEvent(e, true);
    window.removeEventListener('mousemove', move);
    window.removeEventListener('mouseup', up);
  };
  window.addEventListener('mousemove', move);
  window.addEventListener('mouseup', up);
};

const seekLine = (line: LyricLine) => {
  if (line.startTime == null) return;
  seekTo(line.startTime / 1000);
};

const loadCoverImage = (url: string) =>
  new Promise<HTMLImageElement | null>((resolve) => {
    if (!url) {
      resolve(null);
      return;
    }
    const image = new Image();
    image.onload = () => resolve(image);
    image.onerror = () => resolve(null);
    image.src = url;
  });

const startFluidBackground = (src: string) => {
  const canvas = canvasRef.value;
  if (!canvas || !src) return;
  const ctx = canvas.getContext('2d', { alpha: false });
  if (!ctx) return;
  const scene = document.createElement('canvas');
  const sceneCtx = scene.getContext('2d', { alpha: false });
  if (!sceneCtx) return;

  let image: HTMLImageElement | null = null;
  let last = performance.now();
  let time = 0;
  const rot = [
    Math.random() * Math.PI * 2,
    Math.random() * Math.PI * 2,
    Math.random() * Math.PI * 2,
    Math.random() * Math.PI * 2
  ];
  const flowSpeed = 1;
  const fpsInterval = 1000 / 24;

  const resize = () => {
    const scale = 0.42;
    const width = Math.max(2, Math.round(canvas.clientWidth * scale));
    const height = Math.max(2, Math.round(canvas.clientHeight * scale));
    canvas.width = width;
    canvas.height = height;
    scene.width = width;
    scene.height = height;
  };
  resize();
  bgObserver?.disconnect();
  bgObserver = new ResizeObserver(resize);
  bgObserver.observe(canvas);
  loadCoverImage(src).then((loaded) => {
    image = loaded;
  });

  const drawSprite = (x: number, y: number, size: number, rotation: number) => {
    if (!image) return;
    sceneCtx.save();
    sceneCtx.translate(x, y);
    sceneCtx.rotate(rotation);
    sceneCtx.drawImage(image, -size / 2, -size / 2, size, size);
    sceneCtx.restore();
  };

  const tick = (now: number) => {
    bgRaf = requestAnimationFrame(tick);
    if (now - last < fpsInterval) return;
    const delta = Math.min(2.2, (now - last) / 16.667);
    last = now;
    time += delta * flowSpeed;
    const width = canvas.width;
    const height = canvas.height;
    const maxSize = Math.max(width, height);
    sceneCtx.fillStyle = '#111';
    sceneCtx.fillRect(0, 0, width, height);
    if (image) {
      rot[0] += (delta / 1000) * flowSpeed;
      rot[1] -= (delta / 500) * flowSpeed;
      rot[2] += (delta / 1000) * flowSpeed;
      rot[3] -= (delta / 750) * flowSpeed;
      drawSprite(width / 2, height / 2, maxSize * Math.SQRT2, rot[0]);
      drawSprite(width / 2.5, height / 2.5, maxSize * 0.8, rot[1]);
      drawSprite(
        width / 2 + (width / 4) * Math.cos((time / 1000) * 0.75),
        height / 2 + (width / 4) * Math.cos((time / 1000) * 0.75),
        maxSize * 0.5,
        rot[2]
      );
      drawSprite(
        width / 2 + (width / 4) * 0.1 + Math.cos(time * 0.006 * 0.75),
        height / 2 + (width / 4) * 0.1 + Math.cos(time * 0.006 * 0.75),
        maxSize * 0.25,
        rot[3]
      );
    }
    ctx.filter = 'blur(28px) saturate(1.2) brightness(0.58) contrast(0.82)';
    ctx.drawImage(scene, 0, 0, width, height);
    ctx.filter = 'none';
  };
  cancelAnimationFrame(bgRaf);
  bgRaf = requestAnimationFrame(tick);
};

const createSpring = (mass = 0.9, damping = 15, stiffness = 90): Spring => {
  let pos = 0;
  let vel = 0;
  let target = 0;
  let pending: number | null = null;
  let unlockAt = 0;
  return {
    setTarget(next, delay = 0) {
      if (delay > 0) {
        pending = next;
        unlockAt = performance.now() + delay * 1000;
        return;
      }
      if (pending != null && performance.now() < unlockAt) {
        pending = next;
        return;
      }
      target = next;
    },
    snap(next) {
      pos = next;
      vel = 0;
      target = next;
      pending = null;
    },
    update(dt, now = performance.now()) {
      if (pending != null && now >= unlockAt) {
        target = pending;
        pending = null;
      }
      const acc = (stiffness * (target - pos) - damping * vel) / mass;
      vel += acc * dt;
      pos += vel * dt;
      return pos;
    },
    current() {
      return pos;
    }
  };
};

const computeLineBlur = (index: number, currentIndex: number) => {
  if (currentIndex < 0 || index === currentIndex) return 0;
  let blurLevel = 1;
  if (index < currentIndex) blurLevel += Math.abs(currentIndex - index) + 1;
  else blurLevel += Math.abs(index - currentIndex);
  return blurLevel;
};

const rebuildSprings = (list: LyricLine[]) => {
  springs.length = 0;
  list.forEach((_, i) => {
    const prev = list[i - 1];
    const interval =
      prev && list[i].startTime != null && prev.startTime != null
        ? (list[i].startTime as number) - (prev.startTime as number)
        : 400;
    const clamped = Math.min(800, Math.max(100, interval));
    const ratio = (1 - (clamped - 100) / 700) ** 0.2;
    const stiffness = 170 + ratio * 50;
    springs.push(createSpring(0.9, Math.sqrt(stiffness) * 2.2, stiffness));
  });
  lastLyricIndex = -1;
};

const startLyricLoop = () => {
  cancelAnimationFrame(lyricRaf);
  let last = performance.now();
  const tick = (now: number) => {
    const dt = Math.min(0.033, (now - last) / 1000);
    last = now;
    const currentIndex = activeIndexRef.current;
    const viewport = viewportRef.value;
    const art = artRef.value;
    const count = lines.value.length;
    const nodes: HTMLElement[] = [];
    for (let i = 0; i < count; i++) {
      const node = lineRefs.value[i];
      if (!node) {
        lyricRaf = requestAnimationFrame(tick);
        return;
      }
      nodes.push(node);
    }
    if (viewport && nodes.length === count && springs.length === count) {
      const heights = nodes.map((node) => Math.max(node.offsetHeight, 1));
      const prefixes = [0];
      for (let i = 0; i < heights.length; i++) prefixes.push(prefixes[i] + heights[i]);
      const focusIndex = Math.max(0, currentIndex);
      const viewBox = viewport.getBoundingClientRect();
      const artBox = art?.getBoundingClientRect();
      const focusY =
        artBox && artBox.height
          ? artBox.top + artBox.height / 2 - viewBox.top
          : viewBox.height * 0.42;
      const activeMid = prefixes[focusIndex] + heights[focusIndex] / 2;
      const layoutReady = viewBox.height > 0 && heights.every((h) => h > 1);
      const snap = lastLyricIndex < 0;
      const indexChanged = lastLyricIndex !== currentIndex;
      let delay = 0;
      let baseDelay = snap || !indexChanged ? 0 : 0.05;
      for (let i = 0; i < count; i++) {
        const spring = springs[i];
        const y = focusY - activeMid + prefixes[i];
        if (snap || !layoutReady) spring.snap(y);
        else spring.setTarget(y, indexChanged ? delay : 0);
        const current = spring.update(dt, now);
        const node = nodes[i];
        const blur = computeLineBlur(i, currentIndex);
        node.style.transform = `translate3d(0, ${current}px, 0) scale(${i === currentIndex ? 1 : 0.97})`;
        node.style.opacity = i === currentIndex ? '1' : '0.22';
        node.style.filter = blur > 0 ? `blur(${blur}px)` : 'none';
        if (indexChanged && !snap && y + heights[i] >= 0) {
          delay += baseDelay;
          if (i >= focusIndex) baseDelay /= 1.05;
        }
      }
      if (layoutReady) lastLyricIndex = currentIndex;
    }
    lyricRaf = requestAnimationFrame(tick);
  };
  lyricRaf = requestAnimationFrame(tick);
};

const onKeydown = (event: KeyboardEvent) => {
  if (event.key === 'Escape') emit('exit');
};

watch(
  cover,
  (src) => {
    if (src) startFluidBackground(src);
  },
  { immediate: false }
);

watch(
  lines,
  async (list) => {
    lineRefs.value = [];
    rebuildSprings(list);
    await nextTick();
    startLyricLoop();
  },
  { immediate: false }
);

onMounted(async () => {
  hideCursorSoon();
  document.addEventListener('keydown', onKeydown);
  if (cover.value) startFluidBackground(cover.value);
  rebuildSprings(lines.value);
  await nextTick();
  startLyricLoop();
});

onBeforeUnmount(() => {
  document.removeEventListener('keydown', onKeydown);
  if (cursorTimer) clearTimeout(cursorTimer);
  cancelAnimationFrame(bgRaf);
  cancelAnimationFrame(lyricRaf);
  bgObserver?.disconnect();
});
</script>

<style scoped>
#amnp-root {
  position: fixed;
  inset: 0;
  z-index: 10000;
  overflow: hidden;
  cursor: default;
  font-family:
    -apple-system, BlinkMacSystemFont, 'SF Pro Display', 'SF Pro Text', 'Helvetica Neue', system-ui,
    sans-serif;
  color: #fff;
  user-select: none;
}
#amnp-root.hide-cursor {
  cursor: none;
}
#amnp-bg {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  display: block;
  z-index: 0;
  pointer-events: none;
}
#amnp-bg-dim {
  position: absolute;
  inset: 0;
  background: rgba(0, 0, 0, 0.12);
  z-index: 1;
}
#amnp-layout {
  position: relative;
  z-index: 2;
  height: 100%;
  display: grid;
  grid-template-columns: 50% 50%;
  align-items: center;
}
#amnp-left {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 6vh 0;
  width: 100%;
  min-width: 0;
}
#amnp-art {
  width: min(50vh, 38vw);
  aspect-ratio: 1;
  border-radius: max(2%, 8px);
  background-size: cover;
  background-position: center;
  box-shadow: 0 1em 1.2em rgba(0, 0, 0, 0.19);
  transform: scale(1);
  transition:
    transform 0.5s cubic-bezier(0.3, 0.2, 0.2, 1.4),
    box-shadow 0.5s ease;
}
#amnp-art.paused {
  transform: scale(0.94);
  box-shadow: 0 0.8em 0.8em rgba(0, 0, 0, 0.19);
}
#amnp-meta {
  width: min(50vh, 38vw);
  margin-top: 28px;
  mix-blend-mode: plus-lighter;
}
#amnp-meta-row {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 10px;
}
#amnp-title {
  font-size: 16px;
  font-weight: 600;
  letter-spacing: 0.2px;
  line-height: 1.2;
  color: rgba(255, 255, 255, 0.94);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
#amnp-artist {
  margin-top: 3px;
  font-size: 12px;
  font-weight: 400;
  letter-spacing: 0.2px;
  color: rgba(255, 255, 255, 0.45);
  line-height: 1.25;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
#amnp-heart {
  background: none;
  border: 0;
  padding: 0;
  width: 28px;
  height: 28px;
  color: rgba(255, 255, 255, 0.55);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  margin-top: -1px;
}
#amnp-heart.on {
  color: #fff;
}
#amnp-heart svg {
  display: block;
}
#amnp-progress-wrap {
  display: flex;
  flex-direction: column;
  margin-top: 28px;
  width: 100%;
}
#amnp-bar {
  width: 100%;
  height: 18px;
  display: flex;
  align-items: center;
  cursor: pointer;
}
#amnp-bar-track {
  width: 100%;
  height: 4px;
  border-radius: 100px;
  background: rgba(255, 255, 255, 0.22);
  overflow: hidden;
  transition: height 0.22s ease;
}
#amnp-bar:hover #amnp-bar-track {
  height: 7px;
}
#amnp-bar-inner {
  height: 100%;
  border-radius: 100px;
  background: #fff;
}
#amnp-times {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 8px;
  font-size: 11px;
  font-weight: 500;
  font-variant-numeric: tabular-nums;
  color: rgba(255, 255, 255, 0.5);
  letter-spacing: 0.01em;
}
#amnp-controls {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
  margin-top: 22px;
}
#amnp-controls button {
  background: none;
  border: 0;
  padding: 0;
  width: 48px;
  height: 48px;
  color: #fff;
  cursor: pointer;
  opacity: 0.92;
  display: flex;
  align-items: center;
  justify-content: center;
}
#amnp-controls button.edge {
  opacity: 0.5;
  width: 44px;
  height: 44px;
}
#amnp-controls button.edge.on {
  opacity: 1;
}
#amnp-controls button.repeat-one {
  position: relative;
}
#amnp-controls button.repeat-one::after {
  content: '1';
  position: absolute;
  right: 3px;
  bottom: 4px;
  font-size: 10px;
  font-weight: 700;
  line-height: 1;
}
#amnp-controls button.play {
  opacity: 1;
}
#amnp-controls button:active svg {
  transform: scale(0.88);
}
#amnp-controls svg {
  display: block;
}
#amnp-right {
  height: 100%;
  min-height: 100%;
  align-self: stretch;
  min-width: 0;
  position: relative;
  overflow: hidden;
}
#amnp-lyrics-viewport {
  position: absolute;
  inset: 0;
  padding: 0 8vw 0 0;
  overflow: hidden;
  mix-blend-mode: plus-lighter;
}
#amnp-lyrics {
  position: relative;
  height: 100%;
}
.amnp-line {
  position: absolute;
  left: 0;
  right: 0;
  top: 0;
  font-size: max(6.2vh, 3.2vw);
  font-weight: 700;
  letter-spacing: -0.03em;
  line-height: 1.2;
  text-align: left;
  color: #fff;
  padding: 0.42em 0;
  cursor: pointer;
  transform-origin: left center;
  will-change: transform, filter, opacity;
  transition:
    opacity 0.4s ease,
    filter 0.4s ease;
}
.amnp-line.empty {
  opacity: 0 !important;
  filter: none !important;
  pointer-events: none;
  height: 0.9em;
  padding: 0;
}
#amnp-empty-lyrics {
  height: 100%;
  display: flex;
  align-items: center;
  color: rgba(255, 255, 255, 0.28);
  font-size: 28px;
  font-weight: 500;
  padding-left: 0;
}
</style>
