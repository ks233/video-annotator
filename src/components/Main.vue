<template>
  <v-app id="main">
    <v-navigation-drawer v-model="drawer">
      <v-list>
        <v-list-item v-for="note in notes" :key="note.id" :subtitle="note.timeString" :title="note.title"
          @click="selectAndSeekNote(note)" color="primary" :variant="note.isSelcted ? 'outlined' : 'flat'">
          <v-card class="h-100 w-100" color="grey"></v-card>
        </v-list-item>
      </v-list>
    </v-navigation-drawer>
    <v-main>
      <!-- 视频和右侧的笔记框 -->
      <v-row class="px-4 justify-center pt-3">
        <!-- 视频 -->
        <v-col>
          <v-sheet class="d-flex" height="65vh" :elevation="2" style="position: relative">
            <!-- 叠在视频上方的绘图层 -->
            <div id="drawing-layer" v-show="showVideo && videoInfo != null" ref="drawingLayer" class="no-select-text"
              :class="keyCtrl && polylineDrawable ? 'drawing' : ''">
              <svg width="100%" height="100%" style="justify-content: center;" @mousedown.left="startDrawing"
                @mousemove="drawing" @mousedown.right="isErasing = true" @mouseup.right="isErasing = false"
                @click.right.prevent>
                <template v-if="polylineDrawable" v-for="polyline in selectedNote.polylines">
                  <polyline :points="polyline.polyString"
                    style="fill:none;stroke:white;stroke-width:8;fill-rule:evenodd;"
                    @mousemove="removeIfErasing(polyline)" />
                  <polyline :points="polyline.polyString" style="fill:none;stroke:red;stroke-width:3;fill-rule:evenodd;"
                    @mousemove="removeIfErasing(polyline)" />
                </template>
              </svg>
            </div>
            <video ref="videoPlayer" id="my-player" class="video-js"></video>
            <div class="md-display pa-10" v-if="!showVideo || videoInfo == null" v-html="mdContent"></div>
          </v-sheet>
        </v-col>
        <!-- Markdown 笔记框 -->
        <v-col v-show="editMode || (showVideo && videoInfo != null)" :cols="4" height="65vh">
          <v-sheet height="65vh" class="d-flex">
            <MdEditor ref="myMdEditor" v-show="editMode" id="md-editor" v-model="selectedNote.text" class="h-100"
              :toolbars="mdEditorToolbar" :preview="false" :theme="theme.global.name.value">
              <template #defToolbars>
                <Emoji :emojis="emojis" :selectAfterInsert="false" />
              </template>
            </MdEditor>
            <div class="md-display pa-5" v-if="!editMode" v-html="mdContent"></div>
          </v-sheet>
        </v-col>
      </v-row>

      <!-- 显示当前时间 -->
      <v-row class="mt-1">
        <v-col class="pt-0 pl-7">
          <template v-if="videoInfo == null">No Video</template>
          <template v-else>
            <template v-if="!videoInfo.isLocal">
              <a :href="videoInfo.src"> {{ videoInfo.filename }}</a>
            </template>
            <template v-else>
              {{ videoInfo.filename }}
            </template>
          </template>
        </v-col>
        <v-col class="pt-0 font-size-large time-text">
          {{ toHHMMSS(currentTime) }} / {{ toHHMMSS(videoLength) }} [{{ speedList[currentSpeedIndex] }}x]
        </v-col>
        <v-col class="pt-0 pl-0">
          <template v-if="supportFSHandle">
            <template v-if="noteFileHandle != null">
              ✅ RW: "{{ noteFileHandle.name }}"
            </template>
            <template v-else>
              ℹ️ DL: "{{ videoInfo == null ? 'Note' : videoInfo.filename }}.txt"
            </template>
          </template>
          <template v-else>
            🔷 DL: "{{ videoInfo == null ? 'Note' : videoInfo.filename }}.txt"。
          </template>
        </v-col>
      </v-row>


      <!-- 时间轴 -->
      <v-row>
        <div id="timeline" ref="videoTimeline" class="mt-n2 flex-grow-1 cursor-move" @wheel.prevent="onTimelineScroll">
          <v-sheet :height="timelineHeight" @mousedown.left="startDragTimeline" color="rgb(var(--v-theme-timeline))"
            @click.right.prevent.stop :width="Math.max(windowWidth, videoLength * timeScale)">

            <!-- 表示当前播放时间的线，始终在 timeline 中间 -->
            <v-divider :thickness="3" class="center-line border-opacity-75" vertical color="red-lighten-1"
              :style="{ 'position': 'absolute', 'left': windowWidth / 2 + 'px', 'height': timelineHeight + 'px' }"></v-divider>

            <!-- 表示视频开头的线 -->
            <v-divider :thickness="3" class="border-opacity-50" vertical
              :style="{ 'position': 'absolute', 'left': - offsetX + 'px', 'height': timelineHeight + 'px' }"></v-divider>

            <!-- 表示视频结尾的线 -->
            <v-divider :thickness="3" class="border-opacity-50" vertical
              :style="{ 'position': 'absolute', 'left': videoLength * timeScale - offsetX + 'px', 'height': timelineHeight + 'px' }"></v-divider>
            <!-- 时间戳 -->
            <div class="d-inline-flex timestamp no-select-text" v-for="note in notes" :key="note.id"
              :style="{ 'position': 'absolute', 'left': note.time * timeScale - offsetX + 'px', 'height': timelineHeight + 'px' }">
              <v-divider vertical :thickness="2"></v-divider>
              <v-card :max-width="250" :min-width="48" class="mx-3 my-8 d-flex align-center px-3 overflow-x-hidden"
                color="grey-lighten-1" height="36" @click="onTimestampClick($event, note)" style="float:left"
                @mousedown.left.stop="startDragTimestamp($event, note)"
                @click.middle.prevent.stop="setNoteToCurrentTime(note)"
                @click.right.prevent.stop="deleteNote(note, true)">
                {{ removeTitle(firstLine(note.text)) }}
              </v-card>
            </div>
            <!-- 刻度 -->
            <template v-for="i in tlMarkerCount">
              <template v-if="VisibleOnTimeline((i - 1 + tlMarkerOffset) * tlMarkerInterval)">
                <!-- 大 -->
                <v-divider v-if="i % tlMarkerBeat == 1 && parseInt(i / tlMarkerBeat) % tlBigMarkerCull == 0"
                  :thickness="2" class="border-opacity-25" vertical :style="{
                    'position': 'absolute', 'left': ((i - 1 + tlMarkerOffset) * tlMarkerInterval * timeScale - offsetX) + 'px',
                    'height': timelineHeight * tlMarkerRatio + 'px',
                    transform: `translateY(${timelineHeight * (1 - tlMarkerRatio)}px)`
                  }"></v-divider>
                <!-- 小 -->
                <v-divider v-else-if="tlMarkerDensity < 300" :thickness="1" class="border-opacity-25" vertical :style="{
                  'position': 'absolute', 'left': ((i - 1 + tlMarkerOffset) * tlMarkerInterval * timeScale - offsetX) + 'px',
                  'height': timelineHeight * tlMarkerMinorRatio + 'px',
                  transform: `translateY(${timelineHeight * (1 - tlMarkerMinorRatio)}px)`
                }"></v-divider>
              </template>
            </template>
          </v-sheet>
        </div>
      </v-row>
      <v-row class="justify-center mt-5" justify="center">
        <v-col :cols="9">
          <v-row>
            <v-col :cols="3">
            </v-col>
            <!-- 一堆按钮 -->
            <v-col :cols="6" class="d-flex align-center justify-center">
              <!-- 用 mousedown.prevent 阻止按钮在点击后获取焦点，否则按下空格时聚焦的按钮也会被触发 -->
              <round-btn @click="drawer = !drawer" icon="mdi-list-box-outline"></round-btn>
              <round-btn @click="showVideo = !showVideo" :icon="showVideo ? 'mdi-television-off' : 'mdi-television'">
              </round-btn>
              <round-btn @click="editMode = !editMode" :icon="editMode ? 'mdi-eye-outline' : 'mdi-pen'">
              </round-btn>
              <v-divider vertical class="mx-5"></v-divider>
              <round-btn @click="seekPreviousNote()" icon="mdi-skip-backward-outline"></round-btn>
              <round-btn @click="play()" :icon="isPlaying ? 'mdi-pause' : 'mdi-play-outline'"></round-btn>
              <round-btn @click="seekNextNote()" icon="mdi-skip-forward-outline"></round-btn>
              <v-divider vertical class="mx-5"></v-divider>
              <round-btn @click="addDefaultNote()" icon="mdi-text-box-plus-outline"></round-btn>
              <round-btn @click="deleteCurrentNote()" icon="mdi-delete-outline"></round-btn>
              <round-btn @click="imageDatabaseDialog = true" icon="mdi-image-multiple-outline"></round-btn>
              <v-divider vertical class="mx-5"></v-divider>
              <v-speed-dial location="bottom center" transition="fade-transition">
                <template v-slot:activator="{ props: activatorProps }">
                  <!-- <v-fab v-bind="activatorProps" size="large" icon="$vuetify"></v-fab> -->
                  <v-btn v-bind="activatorProps" icon="mdi-file-plus-outline" class="mx-1" size="x-large"></v-btn>
                </template>
                <v-btn :key="1" @click="loadFromFileInput()" icon="mdi-upload"></v-btn>
                <v-btn :key="2" @click="loadFromURLPrompt()" icon="mdi-link-plus"></v-btn>
              </v-speed-dial>
              <v-file-input ref="fileInputBox" @change="onFileInputBox" v-show="false" multiple> </v-file-input>
              <round-btn @click="saveNoteJSON()" icon="mdi-content-save-outline"></round-btn>
            </v-col>
            <v-col>
            </v-col>
          </v-row>
        </v-col>

        <!-- 提示 -->
        <v-snackbar v-model="snackbarSaveSuccess" :timeout="2000">
          💾 Saved: "{{ noteFileHandle != null ? noteFileHandle.name : 'ERROR' }}".
        </v-snackbar>

        <!-- BPM 与偏移量调整 -->
        <v-col :cols="3">
          <v-row>
            <v-col :cols="3" class="d-flex flex-column align-end">
              <v-btn @mousedown.prevent @click="toggleMetronome" class="mx-1 my-2" icon="mdi-metronome"
                :color="useMetronome ? 'blue-lighten-1' : '--v-theme-surface'"></v-btn>
              <v-btn @mousedown.prevent @click="snapToMarker = !snapToMarker" class="mx-1" icon="mdi-magnet"
                :color="snapToMarker ? 'pink-lighten-1' : '--v-theme-surface'"></v-btn>
            </v-col>
            <v-col :cols="8" class="mt-5">
              <v-row>
                <v-number-input class="w-25" v-model="tlMarkerBPM" label="BPM" control-variant="stacked" :min="30"
                  :max="300" :precision="2"></v-number-input>
                <v-number-input class="w-25" v-model="tlMarkerBeat" label="Beat" control-variant="stacked" :min="1"
                  :max="20"></v-number-input>
              </v-row>
              <v-row class="w-100 my-0">
                <v-slider class="ml-0 mr-n6" label="Offset" v-model="tlMarkerOffset" :min="0"
                  :max="tlMarkerBeat"></v-slider>
              </v-row>
            </v-col>
          </v-row>
        </v-col>
      </v-row>

      <!-- 图床窗口 -->
      <v-dialog v-model="imageDatabaseDialog" width="auto">
        <v-card width="40vw" max-height="60vh" min-height="30vh" prepend-icon="mdi-update"
          text="Ctrl-V to add image. Click images to insert into note." title="Image Database">
          <v-row class="pa-5">
            <v-col v-for="(base64, name) in imageDatabase" :key="key"
              class="d-flex child-flex flex-column text-center py-1 px-2" cols="4">
              <v-img :src="base64" aspect-ratio="1.5" class="bg-grey-lighten-2" cover @click="insertImage(name)">
                <template v-slot:placeholder>
                  <v-row align="center" class="fill-height ma-0" justify="center">
                    <v-progress-circular color="grey-lighten-5" indeterminate></v-progress-circular>
                  </v-row>
                </template>
              </v-img>
              <p @click="renameImage(name)">{{ name }}</p>
              <v-btn @click="deleteImage(name)">Delete</v-btn>
            </v-col>
          </v-row>
          <template v-slot:actions>
            <v-btn class="ms-auto" text="Ok" @click="imageDatabaseDialog = false"></v-btn>
          </template>
        </v-card>
      </v-dialog>

      <!-- 一些 debug 用的信息 -->
      <v-row v-if="debugMode" class="px-6">
        <v-btn @click="debug()">debug</v-btn>
        <!-- {{ offsetX }} / {{ windowWidth }} / {{ tlMarkerDensity }} / {{ tlBigMarkerDensity }} / {{ tlBigMarkerCull }}<br>
        timescale: {{ timeScale }} / showVideo {{ showVideo }} {{ mdContent }}<br>
        timelineMarkerCount: {{ tlMarkerCount }} bpm {{ tlMarkerBPM }} offset {{ tlMarkerOffset }} -->
      </v-row>

      <v-btn @mousedown.prevent @click="toggleTheme()" id="toggle-theme" class="mx-1" prepend-icon="mdi-brightness-6"
        size="small">{{ theme.global.name.value }}</v-btn>

    </v-main>
  </v-app>
</template>
<script setup>
import videojs from 'video.js';
import "video.js/dist/video-js.css"

import "videojs-youtube"

import { MdEditor } from 'md-editor-v3';
import 'md-editor-v3/lib/preview.css';
import 'md-editor-v3/lib/style.css';

import { onMounted, ref, computed, watch, onUnmounted, nextTick } from 'vue'

import { nanoid } from 'nanoid'

import { marked } from 'marked';

import { useTheme } from 'vuetify'

import RoundBtn from './RoundBtn.vue';

const debugMode = ref(false)

const videoFileName = ref('')
const videoSrc = ref('')


const videoPlayer = ref(null)
const player = ref(null)
const drawer = ref(true)

// 如果没有视频加载，则所有播放操作都与 player 无关，使用更新时间
const videoLoaded = ref(false)

class VideoInfo {
  constructor(isLocal, src, duration) {
    this.isLocal = isLocal
    this.src = src
    this.duration = duration
    this.height = 1080
    this.width = 1920
  }
  get filename() {
    // 如果是本地文件，src 就是文件名
    if (this.isLocal) return this.src;
    // 如果是 URL，则分割字符串得到文件名
    return this.src.split('/').pop()
  }
}

const videoInfo = ref(null)

const fakeVideoLength = ref(273)
const fakeStartTime = ref(0)
let fakeTimerId = null
function playFake() {
  fakeStartTime.value = Date.now() / 1000 - currentTime.value
  isPlaying.value = true;
  fakeTimerId = setInterval(fakeVideoTimer, 10);
}
function pauseFake() {
  clearInterval(fakeTimerId);
  isPlaying.value = false;
}

function fakeVideoTimer() {
  if (draggingTimeline.value) return;
  currentTime.value = Date.now() / 1000 - fakeStartTime.value
  offsetX.value = timeToOffset(currentTime.value);
  selectNoteByCurrentTime()
}

const currentTime = ref(0)
const videoLength = computed(() => {
  console.log("compute")
  return videoInfo.value == null ? fakeVideoLength.value : videoInfo.value.duration;
})

const videoTimeline = ref(null)
const draggingTimeline = ref(false)
const draggingTimestamp = ref(false)

const mdContent = ref('')

const selectedNote = ref({
  id: "default",
  time: 0,
  text: "",
})

const theme = useTheme()

function toggleTheme() {
  theme.global.name.value = theme.global.current.value.dark ? 'light' : 'dark'
}

watch(selectedNote, (note) => {
  if (note != null) {
    mdContent.value = marked.parse(embedImage(note.text), { breaks: true, sanitize: true })
  }
}, {
  deep: true
})

// 【时间轴与刻度】
const timelineHeight = ref(90)
// 刻度的高度比例
const tlMarkerRatio = ref(0.15)
const tlMarkerMinorRatio = ref(0.08)
// 用 BPM 定义刻度的间隔
const tlMarkerBPM = ref(60)
const tlMarkerInterval = computed(() => 60.0 / tlMarkerBPM.value)
const tlMarkerIntervalStretched = computed(() => tlMarkerInterval.value / currentSpeed.value)
// 大刻度
const tlMarkerBeat = ref(4)
// 整体偏移的刻度数量，单位不是时间而是刻度，代表的时间随 BPM 变化
const tlMarkerOffset = ref(0)
// 覆盖整个视频时间需要的刻度数量
const tlMarkerCount = computed(() => {
  return Math.ceil(videoLength.value / tlMarkerInterval.value)
})

// 剔除屏幕外的时间轴刻度、刻度密集时将它们间隔隐藏
function VisibleOnTimeline(time) {
  if (timeToOffset(time) < offsetX.value - windowWidth.value * 0.5)
    return false;
  if (timeToOffset(time) > offsetX.value + windowWidth.value * 0.5)
    return false;
  return true
}

// 根据刻度间隔，计算屏幕中的刻度数量
const tlMarkerDensity = computed(() => {
  return windowWidth.value / tlMarkerInterval.value / timeScale.value
})

// 屏幕中大刻度的数量
const tlBigMarkerDensity = computed(() => {
  return tlMarkerDensity.value / tlMarkerBeat.value
})

// 大刻度每 n 个显示一个
const tlBigMarkerCull = computed(() => {
  return Math.ceil(tlBigMarkerDensity.value / 300)
})

const windowWidth = ref(0)

// 刻度根据视频时间整体偏移
const offsetX = ref(0)

// 在 video 的 play 和 pause 事件中更新该值，仅用于显示播放器按钮的图标
const isPlaying = ref(false)

const showVideo = ref(true)

// 控制播放器隐藏和显示，感觉太啰嗦了，但没找到别的方法
watch(showVideo, (v) => {
  document.getElementById('my-player').style.setProperty('display', (v && videoInfo.value != null) ? "inline-block" : "none")
})
watch(videoInfo, (v) => {
  document.getElementById('my-player').style.setProperty('display', (showVideo.value && v != null) ? "inline-block" : "none")
})

const useMetronome = ref(false)
const metronomeStart = ref(Date.now())

const snapToMarker = ref(false)

onMounted(async () => {
  player.value = videojs(videoPlayer.value, {
    controls: true,
    controlBar: {
      // remainingTimeDisplay: false,
      // fullscreenToggle: false,
      pictureInPictureToggle: false,
      // volumePanel: false,
      // currentTimeDisplay: false,
      // timeDivider: false,
      // durationDisplay: false
    },
    playbackRates: speedList,
    userActions: {
      doubleClick: false
    },
    // children: ["MediaLoader"],
    techOrder: ["html5", "youtube"],
    youtube: {
      ytControls: 2,
      iv_load_policy: 1
    }
  })


  player.value.on('timeupdate', () => {
    onPlayerTimeUpdate()
  })
  player.value.on('pause', () => {
    isPlaying.value = false
    seeking.value = false
  })
  player.value.on('play', () => {
    console.log('play')
    isPlaying.value = true
    seeking.value = false;
    if (useMetronome.value == true) {
      // 触发 seeked 事件，重新对准节拍器
      player.value.currentTime(currentTime.value)
    }
  })
  player.value.on('seeked', () => {
    console.log('seeked')
    seeking.value = false
    updateMetronomeStart()
  })
  player.value.on('sourceset', () => {
    console.log('sourceset')
    player.value.pause()
    isPlaying.value = false
  })
  // 油管速度到三倍速的时候不会触发
  player.value.on('ratechange', () => {
    console.log('ratechange')
    currentSpeedIndex.value = speedList.findIndex(n => n == player.value.playbackRate())
    if (useMetronome.value == true) {
      // 触发 seeked 事件，重新对准节拍器
      // 只有在 seeked 事件里更新 metronome 才准，直接在其它地方更新会有延迟，太怪了
      player.value.currentTime(currentTime.value)
    }
  })

  player.value.on('loadedmetadata', () => {
    console.log('loadedmetadata', player.value.src())
    videoInfo.value.duration = player.value.duration()
    videoInfo.value.width = player.value.videoWidth() // 油管视频的 width 和 height 都为 0
    videoInfo.value.height = player.value.videoHeight()
    console.log(videoInfo.value)
  })

  updateWindowSize()

  window.addEventListener('resize', updateWindowSize)

  document.addEventListener('dragover', prevent);
  document.addEventListener('drop', dropFile);

  // 避免 markdown 编辑器在不聚焦时响应 Ctrl-Z
  // 屏蔽浏览器 Ctrl-S 保存网页
  document.addEventListener('keydown', preventHotkeys);
  document.addEventListener('keyup', preventHotkeys);
  // 拖拽时间轴和时间戳，只有 mousedown 不是全局
  document.addEventListener('mousemove', globalMouseMove);
  document.addEventListener('mouseup', globalMouseUp);
  document.addEventListener('paste', pasteImage);
  // 节拍器设置
  let prevTime = 0
  let delta = 0
  let nextBeat = 0
  let nextBeatTime = 0
  // 节拍器计时器，一直在跑，不知道对性能有多大影响，目前好像不太卡
  setInterval(() => {
    delta = (Date.now() - metronomeStart.value) / 1000 - tlMarkerOffset.value * tlMarkerIntervalStretched.value - 0.01; // 往前偏移一丢丢，不然第一拍不响

    // delta = player.value.currentTime() - tlMarkerOffset.value * tlMarkerIntervalStretched.value - 0.01;

    nextBeat = Math.ceil(prevTime / tlMarkerIntervalStretched.value)
    nextBeatTime = nextBeat * tlMarkerIntervalStretched.value
    if (delta > nextBeatTime) {
      if (useMetronome.value && isPlaying.value) {
        playMetronome(nextBeat % tlMarkerBeat.value == 0)
      }
    }
    prevTime = delta
    // alternatively just show wall clock time:
  }, 5); // 每 5ms 更新一下计时器

  // Check to see if Media-Queries are supported
  if (window.matchMedia) {
    // Check if the dark-mode Media-Query matches
    if (window.matchMedia('(prefers-color-scheme: dark)').matches) {
      theme.global.name.value = 'dark'
    } else {
      theme.global.name.value = 'light'
    }
  } else {
    // Default (when Media-Queries are not supported)
  }

  offsetX.value = timeToOffset(currentTime.value)

  console.log(urlParamNoteSrc, urlParamVideoSrc)
  if (urlParamNoteSrc != null) {
    if (await getUrlType(urlParamNoteSrc, (o) => { }) == INVALID_URL) {
      removeQueryString()
    }
    loadFromURL(urlParamNoteSrc)
  } else if (urlParamVideoSrc != null) {
    if (await getUrlType(urlParamVideoSrc, (o) => { }) == INVALID_URL) {
      removeQueryString()
    }
    loadFromURL(urlParamVideoSrc)
  }
  document.getElementById('my-player').style.setProperty('display', (showVideo.value == true && videoInfo.value != null) ? "inline-block" : "none")
  reset()
})

onUnmounted(() => {
  window.removeEventListener('resize', updateWindowSize)

  document.removeEventListener('dragover', prevent);
  document.removeEventListener('drop', dropFile);

  document.removeEventListener('keydown', preventHotkeys);
  document.removeEventListener('keyup', preventHotkeys);

  document.removeEventListener('mousemove', globalMouseMove);
  document.removeEventListener('mouseup', globalMouseUp);
})

// 【绘图层】

import { useElementSize } from '@vueuse/core';
const drawingLayer = ref(null)

const isDrawing = ref(false)

let counter = 0
const polylineDrawable = computed(() => {
  let timeDiff = currentTime.value - selectedNote.value.time
  return -0.1 < timeDiff && timeDiff < 1;
})

const layerSize = ref(
  useElementSize(
    drawingLayer,
    { width: 0, height: 0 },
    { box: 'border-box' },
  ),
)

class Polyline {
  /**
   * @param aw 绘制时的容器宽度，用来在容器大小变化时计算坐标
   * @param ah 绘制时的容器高度
   */
  constructor() {
    // 当前容器大小
    this.w = layerSize.value.width
    this.h = layerSize.value.height
    // console.log(this.sw, this.sh)
    this.points = []
  }
  addPoint(x, y) {
    this.points.push([x, y])
  }

  // 由当前容器大小，计算变换后的点坐标，输出为字符串
  get polyString() {
    let w = layerSize.value.width
    let h = layerSize.value.height
    let str = ''
    let aRatio = this.w / this.h
    let bRatio = w / h
    let vRatio = 16 / 9
    if (videoInfo.value.height != 0) {
      vRatio = videoInfo.value.width / videoInfo.value.height
    }
    let ba = 0
    if (aRatio > vRatio && bRatio > vRatio) { // AB 都扁，比容器高度
      ba = h / this.h
    } else if (aRatio < vRatio && bRatio < vRatio) { // AB 都窄，比容器宽度
      ba = w / this.w
    } else if (aRatio > vRatio && bRatio < vRatio) { // A 扁 B 窄，用 B 的容器宽度比 A 的视频宽度
      ba = w / (this.h * vRatio)
    } else if (aRatio < vRatio && bRatio > vRatio) { // A 窄 B 扁，用 B 的容器高度比 A 的视频高度
      ba = h / (this.w / vRatio)
    }
    for (const point of this.points) {
      let x = point[0] - this.w * 0.5
      let y = point[1] - this.h * 0.5
      x *= ba
      y *= ba
      x += w * 0.5
      y += h * 0.5
      str += x + ',' + y + ' '
    }
    return str
  }
}

let polylineDrawing = ref(null)
/**
 *
 * @param {MouseEvent} event
 */
function startDrawing(event) {
  isDrawing.value = true
  polylineDrawing.value = new Polyline()
  selectedNote.value.polylines.push(polylineDrawing.value)
  counter = 0
}

function drawing(event) {
  if (!isDrawing.value) return;
  counter++;
  if (counter % 4 != 0) return;
  polylineDrawing.value.addPoint(event.offsetX, event.offsetY)
}

function finishDrawing(event) {
  if (polylineDrawing.value.points.length <= 1) {
    selectedNote.value.polylines = selectedNote.value.polylines.filter(p => p != polylineDrawing.value)
  }
  isDrawing.value = false
}

const isErasing = ref(false)

function removeIfErasing(polyline) {
  if (!isErasing.value) return;
  selectedNote.value.polylines = selectedNote.value.polylines.filter(p => p != polyline)
}


//【全局事件】

/**
 * @param {MouseEvent} event
 */
function globalMouseMove(event) {
  if (draggingTimeline.value) {
    dragTimeline(event)
    event.preventDefault()
  }
  if (draggingTimestamp.value) {
    dragTimestamp(event)
    event.preventDefault()
  }
}

/**
 * @param {MouseEvent} event
 */
function globalMouseUp(event) {
  if (draggingTimeline.value) {
    finishDragTimeline(event)
  }
  if (draggingTimestamp.value) {
    finishDragTimestamp(event)
  }
  if (isDrawing.value) {
    finishDrawing(event)
  }
}

/**
 * @param {KeyboardEvent} event
 */
function preventHotkeys(event) {
  if (event.ctrlKey && (event.key === 'z' || event.key === 's')) {
    if (notUsingInput.value) {
      event.preventDefault();
    }
  }
  if (event.ctrlKey && event.key === 'e') {
    event.preventDefault();
  }
}

const noteFileHandle = ref(null)

/**
 * @param {DragEvent} event
 */
async function dropFile(event) {
  const items = event.dataTransfer.items;
  Array.from(event.dataTransfer.files).forEach(file => {
    loadFile(file)
  });
  event.preventDefault()
  // 如果浏览器支持使用 FileSystemHandle，记录笔记文件的 handle，以便保存时直接保存到原文件中
  // 按理来说不太旧的 Chrome 或者 Edge 应该都行
  if (!supportFSHandle.value) return;
  for (const item of items) {
    console.log(item, typeof item.getAsFileSystemHandle)
    if (item.kind === 'file' && item.type == 'text/plain' && item.getAsFileSystemHandle) {
      const handle = await item.getAsFileSystemHandle();
      console.log(handle)
      if (handle.kind === 'file') {
        noteFileHandle.value = handle;
      } else {
        noteFileHandle.value = null
      }
    }
  }
}

/**
 *
 * @param {Event} event
 */
function prevent(event) {
  event.preventDefault()
}


// 全局快捷键
import { useActiveElement, useMagicKeys, whenever } from '@vueuse/core'
import { logicAnd } from '@vueuse/math'

// 编辑文本框时不触发全局快捷键
const activeElement = useActiveElement()
const notUsingInput = computed(() =>
  activeElement.value?.tagName !== 'INPUT'
  && activeElement.value?.tagName !== 'TEXTAREA'
  // Markdown 编辑器（md-editor-v3）的输入框有 spellcheck 的属性，其它没有
  // 如果之后其它元素也有 spellcheck，可能导致快捷键失效
  && !activeElement.value?.hasAttribute("spellcheck")
  && !imageDatabaseDialog.value
)

const keys = useMagicKeys()


// Ctrl-S 保存，这个可以在编辑器里触发
const keyCtrlS = keys['ctrl+s']
whenever(keyCtrlS, (v) => {
  saveNoteJSON();
})

// 读取链接的 query string 比如 xxx.com/?n=xxx.txt&v=xxx.mp4
const urlParams = new URLSearchParams(window.location.search);
const urlParamNoteSrc = urlParams.get('n');
const urlParamVideoSrc = urlParams.get('v');

// 【播放速度控制】

const speedList = [0.5, 0.75, 1, 1.5, 2, 3]
const defaultSpeedIndex = 2
const currentSpeedIndex = ref(2)
const storedSpeedIndex = ref(4)

const currentSpeed = computed(() => speedList[currentSpeedIndex.value])

// watch(currentSpeed, (v) => {
//   updateMetronomeStart()
// })

// Z X C 变速，和 PotPlayer 类似
// Ctrl-Z 撤销，Ctrl-Shift-Z 重做

const keyZ = keys['z']
const keyX = keys['x']
const keyC = keys['c']
const keyShift = keys['shift']
const keyCtrl = keys['ctrl']

// Z, Ctrl-Z, Ctrl-Shift-Z
whenever(logicAnd(keyZ, notUsingInput), (v) => {
  if (keyCtrl.value) {
    if (keyShift.value) {
      redo();
    } else {
      undo()
    }
    return;
  }
  if (currentSpeedIndex.value == defaultSpeedIndex) {
    currentSpeedIndex.value = storedSpeedIndex.value
  } else {
    storedSpeedIndex.value = currentSpeedIndex.value
    currentSpeedIndex.value = defaultSpeedIndex
  }
})

whenever(logicAnd(keyX, notUsingInput), (v) => {
  currentSpeedIndex.value = Math.max(currentSpeedIndex.value - 1, 0)
})

whenever(logicAnd(keyC, notUsingInput), (v) => {
  currentSpeedIndex.value = Math.min(currentSpeedIndex.value + 1, speedList.length - 1)
})

// 更新播放速度
watch(currentSpeedIndex, (v) => {
  player.value.playbackRate(speedList[v]);
})

// Space 控制播放和暂停
const keySpace = keys['space']

whenever(logicAnd(keySpace, notUsingInput), (v) => {
  play()
})

// D F 控制上一帧下一帧

const keyD = keys['d']
const keyF = keys['f']

whenever(logicAnd(keyD, notUsingInput), (v) => {
  if (!player.value.paused()) return; // 只在视频暂停的时候启用该功能
  seek(currentTime.value - 0.04) // 随便写个很短的时间假装逐帧调整
})

whenever(logicAnd(keyF, notUsingInput), (v) => {
  if (!player.value.paused()) return;
  seek(currentTime.value + 0.04)
})

// 左右控制前进/后退 5s

const keyLeft = keys['left']
const keyRight = keys['right']

whenever(logicAnd(keyLeft, notUsingInput), (v) => {
  seek(currentTime.value - 5)
})

whenever(logicAnd(keyRight, notUsingInput), (v) => {
  seek(currentTime.value + 5)
})

// 【撤销与重做】

// 用栈存储指令，元素中都有对应的撤销和重做操作
let commandStack = []
let commandStackPointer = -1
let moveCmd = {
  undo: () => { },
  redo: () => { }
}

function clearCommandStack() {
  commandStack = []
  commandStackPointer = -1
}

/**
 * 注册指令添加到栈中
 * @param {Note} note
 */
function startRegisterMove(note) {
  let time = note.time
  let id = note.id
  moveCmd = {
    undo: () => setNoteTimeByID(id, time),
    redo: null
  }
}

/**
 * 注册指令添加到栈中
 * @param {Note} note
 */
function finishRegisterMove(note) {
  let time = note.time
  let id = note.id
  moveCmd.redo = () => setNoteTimeByID(id, time)
  registerCmd(moveCmd)
}

function registerCmd(cmd) {
  commandStack = commandStack.splice(0, commandStackPointer + 1)
  commandStack.push(cmd)
  commandStackPointer++
}

// 判断栈指针是否正常
function debug_validateCmdSP() {
  if (commandStackPointer >= commandStack.length || commandStackPointer < -1) {
    console.error("invalid history SP")
  }
}

// 移动栈指针，调用对应的 undo 或 redo
function undo() {
  debug_validateCmdSP()
  // console.log('Undo')
  if (commandStackPointer >= 0) {
    commandStack[commandStackPointer].undo()
    commandStackPointer--
  } else {
    // console.log('no undo')
  }
  console.log(commandStack, commandStackPointer)
}

function redo() {
  debug_validateCmdSP()
  // console.log('Redo')
  if (commandStack.length == 0) return;
  if (commandStackPointer >= commandStack.length - 1) {
    // console.log('no redo')
    return;
  }
  commandStackPointer++
  commandStack[commandStackPointer].redo()
  console.log(commandStack, commandStackPointer)
}

/**
 * 根据 id 找到要操作的元素，避免 add delete 之后丢失引用
 * @param {String} id
 */
function findNoteByID(id) {
  return notes.value.find(note => note.id == id)
}

/**
 * @param {String} id
 */
function deleteNoteByID(id) {
  let note = findNoteByID(id)
  if (note) {
    deleteNote(note, false)
  }
}

/**
 * @param {String} id
 * @param {Number} time
 */
function setNoteTimeByID(id, time) {
  /**
   * @type {Note}
   */
  let note = findNoteByID(id)
  if (note) {
    note.time = time
  }
  sortNotes()
}

import { Emoji } from '@vavt/v3-extension'
import '@vavt/v3-extension/lib/asset/Emoji.css';

// Markdown 编辑器（md-editor-v3）工具栏的按钮布局
const mdEditorToolbar = [
  // 0, '-',
  'bold',
  'underline',
  'italic',
  '-',
  'strikeThrough',
  'quote',
  'unorderedList',
  'orderedList',
  'task',
  'link',
  '=', 0
  // 'preview',
  // 'previewOnly',
]

const emojis = [
  '🔴', '🟠', '🟡', '🟢', '🔵', '🟣',
  '✔️', '❌', '❓', '💬', 'ℹ️', '⚠️',
  '🎼', '🎹', '🎸', '🥁', '🎻', '🎺',]

const editMode = ref(true)

const keyCtrlE = keys['ctrl+e']
const myMdEditor = ref(null)
whenever(keyCtrlE, (v) => {
  editMode.value = !editMode.value
  nextTick(() => {
    if (editMode.value == true) {
      myMdEditor.value.focus()
    }
  })
})


function updateWindowSize() {
  windowWidth.value = window.innerWidth
  offsetX.value = timeToOffset(currentTime.value)
}

async function debug() {
  console.log(notes.value)
  addImage('test', '=base64xxx=')
  addImage('foo', 'werqreqwre')
  console.log(embedImage(
    `
![](__test)
test,test, h![](__foo)aswet
wasdfasd
    `
  ))
}

// 删除链接中的 query string 并刷新页面
function removeQueryString() {
  window.history.replaceState(null, null, window.location.pathname);
  window.location.href = '';
}

// 【添加和删除笔记】

function addNote(newNote, regUndo = false) {
  notes.value.push(newNote)
  if (regUndo) {
    let noteCopy = newNote.clone()
    let id = newNote.id
    registerCmd(
      {
        undo: () => { deleteNoteByID(id) },
        redo: () => { addNote(noteCopy) }
      }
    )
  }
  sortNotes()
  selectNoteByCurrentTime()
}

function addDefaultNote() {
  addNote(new Note(currentTime.value, "# "), true)
}

function deleteNote(note, regUndo = false) {
  if (notes.value.length == 0) return;
  var index = notes.value.indexOf(note);
  if (regUndo) {
    let noteCopy = note.clone()
    let id = note.id
    registerCmd(
      {
        undo: () => { addNote(noteCopy); console.log(noteCopy) },
        redo: () => { deleteNoteByID(id) }
      }
    )
  }
  if (index > -1) {
    notes.value.splice(index, 1);
  }
  if (index > 0) {
    selectedNote.value = notes.value[index - 1]
  } else {
    if (notes.value.length > 0) {
      selectedNote.value = notes.value[0]
    } else {
      selectedNote.value = {
        time: 0,
        text: ""
      }
    }
  }
}

function deleteCurrentNote() {
  deleteNote(selectedNote.value, true)
}

/**
 *
 * @param {PointerEvent} event
 * @param {Note} note
 */
function onTimestampClick(event, note) {
  if (event.clientX != startDragXTs) {
    return;
  } else {
    selectAndSeekNote(note)
  }
}

function selectAndSeekNote(note) {
  selectedNote.value = note
  seek(note.time)
}


// 滚轮调整时间轴的横向缩放

const tsList = [0.2, 0.3, 0.5, 1, 2, 3, 4, 5, 10, 20, 30, 40, 50, 80, 150, 300]

function onTimelineScroll(event) {

  if (event.deltaY > 0) {
    timeScale.value = tsList.findLast(ts => ts < timeScale.value) ?? tsList[0]
  } else {
    timeScale.value = tsList.find(ts => ts > timeScale.value) ?? tsList[tsList.length - 1]
  }
  // timeScale.value = timeScale.value.clamp(1, 300)
  offsetX.value = timeToOffset(currentTime.value);
}

// 拖拽时间轴

let startDragX = 0
let startDragTime = 0

function startDragTimeline(event) {
  draggingTimeline.value = true
  startDragX = event.clientX
  startDragTime = currentTime.value
}

/**
 * @param {MouseEvent} event
 */
function dragTimeline(event) {
  if (!draggingTimeline.value) return;
  let delta = startDragX - event.clientX;
  if (snapToMarker.value) {
    let targetMarker = Math.round((startDragTime + delta / timeScale.value - tlMarkerOffset.value * tlMarkerInterval.value) / tlMarkerInterval.value)
    offsetX.value = timeToOffset((targetMarker + tlMarkerOffset.value) * tlMarkerInterval.value)
  } else {
    offsetX.value = timeToOffset(startDragTime) + delta;
  }
  offsetX.value = Math.max(offsetX.value, -windowWidth.value / 2);
  offsetX.value = Math.min(offsetX.value, timeToOffset(videoLength.value))
  currentTime.value = offsetToTime(offsetX.value)
}

/**
 * @param {MouseEvent} event
 */
function finishDragTimeline(event) {
  if (!draggingTimeline.value) return;
  draggingTimeline.value = false
  seek(offsetToTime(offsetX.value))
  fakeStartTime.value = Date.now() / 1000 - currentTime.value
  selectNoteByCurrentTime()
}

// 拖拽时间戳

const timestampOffsetX = ref(0)

let prevXts = 0

const draggedNote = ref(null);
let startDragXTs = 0
let startDragTimeTs = 0
let startDragNoteTimeTs = 0

/**
 *
 * @param {MouseEvent} event
 * @param {Note} note
 */
function startDragTimestamp(event, note) {
  draggedNote.value = note
  draggingTimestamp.value = true
  prevXts = event.clientX
  startDragXTs = event.clientX
  startDragTimeTs = currentTime.value
  startDragNoteTimeTs = note.time
  timestampOffsetX.value = timeToOffset(draggedNote.value.time)
  startRegisterMove(note)
}

/**
 * @param {MouseEvent} event
 */
function dragTimestamp(event) {
  if (!draggingTimestamp.value) return;
  let delta = event.clientX - startDragXTs;
  // videoTimeline.value.scrollLeft += delta;
  if (snapToMarker.value) {
    let targetMarker = Math.round((offsetToTime(timestampOffsetX.value + delta) + currentTime.value - startDragTimeTs - tlMarkerOffset.value * tlMarkerInterval.value) / tlMarkerInterval.value)
    let newTime = (targetMarker + tlMarkerOffset.value) * tlMarkerInterval.value
    draggedNote.value.time = newTime.clamp(0, videoLength.value)
  } else {
    timestampOffsetX.value = delta + timeToOffset(startDragNoteTimeTs + currentTime.value - startDragTimeTs);
    timestampOffsetX.value = Math.max(timestampOffsetX.value, -windowWidth.value / 2);
    timestampOffsetX.value = Math.min(timestampOffsetX.value, timeToOffset(videoLength.value))
    draggedNote.value.time = offsetToTime(timestampOffsetX.value)
  }
}

/**
 * @param {MouseEvent} event
 */
function finishDragTimestamp(event) {
  if (!draggingTimestamp.value) return;
  draggingTimestamp.value = false
  if (startDragXTs == event.clientX) return;
  draggedNote.time = offsetToTime(timestampOffsetX.value)
  finishRegisterMove(draggedNote.value)
  sortNotes()
  selectNoteByCurrentTime()
}

function setNoteToCurrentTime(note) {
  startRegisterMove(note)
  note.time = currentTime.value
  finishRegisterMove(note)
  sortNotes()
}

function sortNotes() {
  notes.value.sort((a, b) => a.time - b.time)
  console.log('sort')
}

function seekPreviousNote() {
  let note = notes.value.findLast(note => note.time < currentTime.value - 0.02)
  if (note) {
    seek(note.time)
  }
  else {
    seek(0)
  }
}

function seekNextNote() {
  let note = notes.value.find(note => note.time > currentTime.value + 0.02)
  if (note) {
    seek(note.time)
  }
  else {
    seek(videoLength.value)
  }
}

function selectNoteByCurrentTime() {
  let note = notes.value.findLast(note => note.time < currentTime.value + 0.02)
  if (note) {
    selectedNote.value = note;
  } else {
    if (notes.value.length > 0) {
      selectedNote.value = notes.value[0];
    }
  }
}
/**
 * @param {Number} offset 时间轴的像素偏移量
 * @returns {Number} 视频时间
 */
function offsetToTime(offset) {
  return (offset + windowWidth.value / 2) / timeScale.value
}


/**
 * @param {Number} time 视频时间
 * @returns {Number} 时间轴的像素偏移量
 */
function timeToOffset(time) {
  return -windowWidth.value / 2 + time * timeScale.value
}

function onPlayerTimeUpdate() {
  if (draggingTimeline.value) return;
  if (seeking.value) return;
  currentTime.value = player.value.currentTime()
  offsetX.value = timeToOffset(currentTime.value);
  selectNoteByCurrentTime()
}

function updateMetronomeStart() {
  let spd = currentSpeed.value
  metronomeStart.value = Date.now() - (currentTime.value / currentSpeed.value) * 1000
  console.log(currentSpeed.value)
}

/**
 * @param {Boolean} firstBeat 是否为第一拍
 */
function playMetronome(firstBeat) {
  var audio = new Audio(firstBeat ? 'metronome_up.wav' : 'metronome_down.wav');
  audio.play();
}

function toggleMetronome() {
  useMetronome.value = !useMetronome.value
  if (useMetronome.value == true) {
    seek(currentTime.value)
  }
}


function play() {
  if (videoInfo.value) {
    if (player.value.paused()) {
      player.value.play();
    } else {
      player.value.pause();
    }
  } else {
    if (isPlaying.value) {
      pauseFake()
    } else {
      playFake()
      updateMetronomeStart()
    }
  }
}

const seeking = ref(false)

/**
 * 设置当前视频时间，并更新节拍器的时间偏移
 * @param {Number} time 目标时间
 */
function seek(time) {
  // 直到 seeked 事件触发，seeking 变为 false
  // 在 seeking = true 时 onPlayerTimeUpdate 不触发
  seeking.value = true;
  if (videoInfo.value) {
    currentTime.value = time.clamp(0, videoLength.value)
    player.value.currentTime(currentTime.value);
    offsetX.value = timeToOffset(currentTime.value)
  } else {
    currentTime.value = time.clamp(0, videoLength.value)
    offsetX.value = timeToOffset(currentTime.value)
    updateMetronomeStart()
  }
}

const timeScale = ref(100)

const notes = ref([])

// 【图床】

const imageDatabaseDialog = ref(false)
const imageDatabase = ref({})

watch(imageDatabase, () => {
  if (selectedNote.value != null) {
    mdContent.value = marked.parse(embedImage(selectedNote.value.text))
  }
}, {
  deep: true
})

/**
 * @param {String} name
 * @param {String} base64 Base64
 */
function addImage(name, base64) {
  imageDatabase.value[name] = base64
}

// 匹配形如 ![](__xxx) 中的 xxx
const imageRegex = /\[\]\(__([^)]+)\)/gm
/**
 * 将 ![](__name) 替换为 ![](base64)
 * 正则搞不懂，找 GPT 写了，能跑就行
 * @param {String} str
 */
function embedImage(str) {
  return str.replace(imageRegex, (match, name) => {
    return imageDatabase.value.hasOwnProperty(name) ? `[](${imageDatabase.value[name]})` : match;
  })
}

function pasteImage(event) {
  if (imageDatabaseDialog.value == false) return
  console.log('paste')
  const items = (event.clipboardData || event.originalEvent.clipboardData).items;
  for (let index in items) {
    const item = items[index];
    if (item.kind === 'file' && item.type.startsWith('image/')) {
      const blob = item.getAsFile();
      const reader = new FileReader();
      reader.onload = function (event) {
        const base64String = event.target.result;
        // Use the base64String (e.g., set it as the src of an image)
        console.log(base64String);
        imageDatabase.value[nanoid()] = base64String
        console.log(imageDatabase.value)
      };
      reader.readAsDataURL(blob);
      return;
    }
  }
}

function deleteImage(name) {
  delete imageDatabase.value[name]
}

function renameImage(name) {
  let newName = prompt('Rename Image:', name)
  if (newName) {
    if (newName != name) {
      if (!imageDatabase.value.hasOwnProperty(newName)) {
        imageDatabase.value[newName] = imageDatabase.value[name]
        deleteImage(name)
      }
    }
  }
}

function insertImage(name) {
  myMdEditor.value.insert((selectedText) => {
    /**
     * @return targetValue    待插入内容
     * @return select         插入后是否自动选中内容，默认：true
     * @return deviationStart 插入后选中内容鼠标开始位置，默认：0
     * @return deviationEnd   插入后选中内容鼠标结束位置，默认：0
     */
    return {
      targetValue: `![](__${name})`,
      select: false,
      deviationStart: 0,
      deviationEnd: 0,
    };
  });
}

// const keyCtrlV = keys['ctrl+v']

// whenever(logicAnd(keyCtrlV, imageDatabaseDialog), (v) => {
//   console.log('paste')
// })

// 【Util】

function firstLine(str) {
  return str.split("\n")[0];
}

function removeTitle(str) {
  while (str.startsWith("#")) {
    str = str.substring(1);
  }
  return str
}

/**
 * @param {Number} num
 */
function toHHMMSS(num) {
  var sec_num = parseInt(num, 10);
  var hours = Math.floor(sec_num / 3600);
  var minutes = Math.floor((sec_num - (hours * 3600)) / 60);
  var seconds = sec_num - (hours * 3600) - (minutes * 60);

  if (hours < 10) { hours = "0" + hours; }
  if (minutes < 10) { minutes = "0" + minutes; }
  if (seconds < 10) { seconds = "0" + seconds; }
  return hours + ':' + minutes + ':' + seconds;
}

// 【文件的读取和保存】

const snackbarSaveSuccess = ref(false)

/**
 * @param {File} file
 */
function loadFile(file) {
  if (file.type == "text/plain") {
    loadNoteJSON(file)
  } else {
    console.log(file)
    loadLocalVideo(file)
  }
}

function download(content, fileName, contentType) {
  var a = document.createElement("a");
  var file = new Blob([content], { type: contentType });
  a.href = URL.createObjectURL(file);
  a.download = fileName;
  a.click();
}

const saveFileName = computed(() => {
  return videoInfo.value == null ? 'Note.txt' : videoInfo.value.filename + '.txt'
})

async function saveNoteJSON() {
  let saveData = makeSaveData()
  let json = JSON.stringify(saveData, replacer, 2)

  // GPT 师傅写的使 polyline 的 points 变成为一行的魔法
  function replacer(key, value) {
    if (key === "points" && Array.isArray(value)) {
      return value.map(pair => `[${pair.join(',')}]`).join(', ');
    }
    return value;
  }
  json = json.replace(/"points": "(\[.*?\])"/g, (match, group) => {
    return `"points": [ ${group} ]`;
  });
  // 直接读写或者下载
  if (noteFileHandle.value != null) {
    console.log('write')
    const writable = await noteFileHandle.value.createWritable();
    await writable.write(json);
    await writable.close();
    snackbarSaveSuccess.value = true
  } else {
    download(json, saveFileName.value, 'text/plain');
  }
}

/**
 * @param {File} file
 */
function loadNoteJSON(file) {
  var reader = new FileReader();
  reader.onload = event => {
    var saveData = JSON.parse(event.target.result);
    loadSaveData(saveData)
    clearCommandStack()
    seek(0)
    selectNoteByCurrentTime()
  };
  reader.readAsText(file);
}

// 链接的类型，V = Video，N = Note
const V_HTML5 = 0
const V_YOUTUBE = 1
const V_FAKE = 2
const N_JSON = 10
const INVALID_URL = -1

/**
 *
 * @param {String} url
 * @param {function(Object):void} jsonCallback
 */
async function getUrlType(url, jsonCallback) {
  let ext = get_url_extension(url)
  // 用简单的字符串匹配视频类型
  if (url.startsWith("https://www.youtube.com/" || "www.youtube.com/" || "youtube.com/")) {
    console.log('youtube')
    return V_YOUTUBE;
  }

  if (['mp4', 'mov', 'webm'].includes(ext)) {
    return V_HTML5
  }

  if (['mp3', 'wav'].includes(ext)) {
    return V_HTML5
  }

  // 尝试请求并 parse json，如果成功则类型为 json
  try {
    const response = await fetch(url);
    // 如果请求失败，直接 invalid
    if (!response.ok) {
      console.log(`Response status: ${response.status}`);
      return INVALID_URL
    }
    let text = await response.text()
    console.log(text)
    try {
      const obj = JSON.parse(text);
      jsonCallback(obj)
      return N_JSON
    } catch (error) {
    }
  } catch (error) {
    console.log(error)
    return INVALID_URL
  }

  return INVALID_URL;
}

// 来源：https://stackoverflow.com/questions/6997262/how-to-pull-url-file-extension-out-of-url-string-using-javascript
// 如果输入 https://www.youtube.com/watch?v=XXX 它会输出 com/watch
// 暂时够用，但感觉不是很靠谱
function get_url_extension(url) {
  return url.split(/[#?]/)[0].split('.').pop().trim();
}

/**
 *
 * @param {String} url
 */
function loadYoutubeVideo(url) {
  // 先设置 info，再设置 src，用播放器的 callback 设置 videoInfo.duration
  videoInfo.value = new VideoInfo(false, url, 0)
  player.value.src({ type: 'video/youtube', src: url })
  player.value.controls(false)
}

/**
 *
 * @param {String} url
 */
function loadHtmlVideo(url) {
  videoInfo.value = new VideoInfo(false, url, 0)
  player.value.src({ type: 'video/mp4', src: url })
  player.value.controls(true)
  seek(0)
}

/**
 *
 * @param {File} url
 */
function loadLocalVideo(file) {
  var URL = window.URL || window.webkitURL
  var fileURL = URL.createObjectURL(file)
  videoInfo.value = new VideoInfo(true, file.name, 0)
  player.value.controls(true)
  player.value.src({ type: file.type, src: fileURL })
  seek(0)
}

async function loadFromURLPrompt() {
  let url = prompt('输入笔记或视频 URL（请确认当前笔记已保存）', 'https://www.youtube.com/watch?v=dQw4w9WgXcQ')
  if (url) {
    reset()
    await loadFromURL(url)
  }
}

async function loadFromURL(url) {
  const type = await getUrlType(url, (obj) => {
    loadSaveData(obj)
    noteFileHandle = null
  })
  if (type == V_YOUTUBE) {
    loadYoutubeVideo(url)
  } else if (type == V_HTML5) {
    loadHtmlVideo(url)
  }
}

const fileInputBox = ref(null)

function loadFromFileInput() {
  fileInputBox.value.click()
}

function onFileInputBox(event) {
  for (const file of event.target.files) {
    loadFile(file)
  }
}

// 浏览器是否支持直接读写文件
const supportFSHandle = computed(() => {
  return (
    typeof window.showOpenFilePicker === 'function' &&
    typeof window.showSaveFilePicker === 'function' &&
    typeof window.showDirectoryPicker === 'function'
  );
})

/**
 * Returns a number whose value is limited to the given range.
 *
 * Example: limit the output of this computation to between 0 and 255
 * (x * 255).clamp(0, 255)
 *
 * @param {Number} min The lower boundary of the output range
 * @param {Number} max The upper boundary of the output range
 * @returns A number in the range [min, max]
 * @type Number
 */
Number.prototype.clamp = function (min, max) {
  return Math.min(Math.max(this, min), max);
};

/**
 *
 */
class Note {
  /**
   * @type {String}
   */
  id;
  /**
   * @param {Number} time
   * @param {String} text
   */
  constructor(time, text) {
    this.id = nanoid()
    this.time = time
    this.text = text
    this.polylines = []
  }
  get title() {
    let s = firstLine(removeTitle(this.text)).trim()
    if (s == '') {
      return this.timeString
    }
    return s
  }
  get isSelcted() {
    return this.id == selectedNote.value.id
  }
  get timeString() {
    return toHHMMSS(this.time)
  }
  clone() {
    const newNote = { ...this }
    Object.setPrototypeOf(newNote, Note.prototype)
    return newNote
  }
}

function reset() {
  noteFileHandle.value = null
  // 暂停
  player.value.pause()
  seek(0)
  // 清空撤销队列
  clearCommandStack()
  notes.value = []
  selectedNote.value = new Note(0, '')
}

function makeSaveData() {
  const saveData = {
    showVideo: showVideo.value,
    editMode: editMode.value,
    videoInfo: videoInfo.value,
    markers: {
      bpm: tlMarkerBPM.value,
      beat: tlMarkerBeat.value,
      offset: tlMarkerOffset.value
    },
    notes: notes.value,
    imageDatabase: imageDatabase.value
  }
  return saveData
}

function loadSaveData(saveData) {
  if (saveData.notes === undefined) {
    console.log('invalid save data!')
    return
  }
  tlMarkerBPM.value = saveData.markers.bpm
  tlMarkerBeat.value = saveData.markers.beat
  tlMarkerOffset.value = saveData.markers.offset
  imageDatabase.value = saveData.imageDatabase ?? imageDatabase.value
  showVideo.value = saveData.showVideo ?? showVideo.value
  console.log(saveData.editMode)
  editMode.value = saveData.editMode ?? editMode.value

  videoInfo.value = saveData.videoInfo
  Object.setPrototypeOf(videoInfo.value, VideoInfo.prototype)

  if (saveData.videoInfo != null) {
    if (!saveData.videoInfo.isLocal)
      loadFromURL(saveData.videoInfo.src)
  }

  saveData.notes.forEach(note => {
    Object.setPrototypeOf(note, Note.prototype)
    if (note.polylines != null) {
      note.polylines.forEach(polyline => {
        Object.setPrototypeOf(polyline, Polyline.prototype)
      })
    }
  })
  notes.value = saveData.notes
}

</script>

<style scoped>
.video-js {
  /* position: relative !important; */
  width: 100% !important;
  height: 100% !important;
}

.cursor-move {
  cursor: move;
}

.timestamp:hover {
  z-index: 999;
}

.center-line {
  z-index: 99;
}

.md-editor .cm-editor {
  font-size: 24px !important;
  height: 100%;
}

.no-select-text {
  -webkit-user-select: none;
  /* Safari */
  -ms-user-select: none;
  /* IE 10 and IE 11 */
  user-select: none;
  /* Standard syntax */
}

.time-text {
  font-size: 1.2em;
  padding: 0;
  margin-top: -2px;
}

div#md-content h1 {
  font-size: 5em
}

.v-theme--light {
  --v-theme-timeline: 230, 230, 230;
  --md-border: solid;
  --md-border-color: #ccc;
  --a-color: #2d8cf0;
  --bold-color: #000;
}

.v-theme--dark {
  --v-theme-timeline: 34, 34, 34;
  --md-border: solid;
  --md-border-color: #333;
  --a-color: #e5c07b;
  --bold-color: #ff7a7a;
}

.md-editor-dark {
  --md-bk-color: #222;
}

.md-editor-content {
  font-size: 2em;
}

.md-display {
  overflow-y: scroll;
  width: 100%;
  border-style: var(--md-border);
  border-color: var(--md-border-color);
  border-width: 1px;
}

#toggle-theme {
  position: absolute;
  right: 10px;
  bottom: 10px;
}

#drawing-layer {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 10;
  background-color: rgba(255, 255, 255, 0.0);
  pointer-events: none;
  transition: background-color 0.3s;
}

.drawing {
  pointer-events: inherit !important;
  background-color: rgba(108, 144, 255, 0.2) !important;
}

#timeline * {
  -webkit-user-select: none;
  /* Safari */
  -ms-user-select: none;
  /* IE 10 and IE 11 */
  user-select: none;
  /* Standard syntax */
}
</style>

<!-- 没有 scoped 才能影响 Marked 渲染生成的内容 -->
<style>
a {
  color: var(--a-color);
}

a:visited {
  color: var(--a-color);
}

a:hover {
  color: var(--a-color);
}

a:active {
  color: var(--a-color);
}


#md-editor .cm-line {
  font-size: 1.8em;
  line-height: 1.5;
  height: auto;
  /* margin-top: 6px; */
}

#md-editor .cm-content {
  padding-top: 12px;
}

.md-display img {
  max-width: 100%;
  max-height: 400px;
}

.md-display h1 {
  font-size: 2.5em;
  margin-top: 8px;
  margin-bottom: 8px;
}

.md-display h2 {
  font-size: 2em;
  margin-top: 8px;
  margin-bottom: 8px;
}

.md-display strong, u, em {
  color: var(--bold-color);
}

.md-display p {
  font-size: 1.5em;
  margin-top: 8px;
  margin-bottom: 8px;
}

.md-display p {
  font-size: 1.5em;
  margin-top: 8px;
  margin-bottom: 8px;
}

.md-display li {
  margin-left: 1em;
  font-size: 1.5em;
  margin-top: 8px;
  margin-bottom: 8px;
}

.emoji-container .emojis li {
  font-size: 24px;
  height: 48px;
  width: 48px;
  display: flex;
  justify-content: center;
  align-items: center;
}

ol.emojis {
  width: 286px;
  margin: 0px;
}
</style>
