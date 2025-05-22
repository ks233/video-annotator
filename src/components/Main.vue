<template>
  <v-app id="main">
    <v-navigation-drawer v-model="drawer">
      <v-list>
        <v-list-item color="primary" v-for="note in notes" :subtitle="toHHMMSS(note.time)"
          :title="removeTitle(firstLine(note.text))" link @click="selectAndGotoNote(note)"></v-list-item>
      </v-list>
    </v-navigation-drawer>

    <v-app-bar>
      <v-app-bar-nav-icon @click="drawer = !drawer"></v-app-bar-nav-icon>

      <v-app-bar-title>Application

        <v-col :cols="2">
          <v-file-input ref="videoFileInput" label="拖拽加载视频文件..." accept=".mp4,.mov,.webm"
            @change="updateVideoSrc"></v-file-input>
        </v-col>
        <!-- <v-col :cols="2">
          <v-file-input label="笔记文件"></v-file-input>
        </v-col> -->
      </v-app-bar-title>
    </v-app-bar>

    <v-main>
      <!-- 视频和左侧的笔记框 -->
      <v-row class="px-4">
        <!-- 视频 -->
        <v-col :cols="8">
          <v-sheet color="grey-lighten-2" height="70vh">
            <video ref="videoPlayer" id="my-player" class="video-js" controls preload="auto">
            </video>
          </v-sheet>
        </v-col>
        <!-- Markdown 笔记框 -->
        <v-col :cols="4" height="70vh">
          <v-sheet height="70vh" class="d-flex" color="grey-lighten-2">

            <MdEditor id="md-editor" v-model="selectedNote.text" class="h-100" :toolbars="mdEditorToolbar"
              :preview="false" />

          </v-sheet>
        </v-col>
      </v-row>

      <!-- 显示当前时间 -->
      <v-row class="justify-center">
        {{ toHHMMSS(currentTime) }} / {{ toHHMMSS(videoLength) }}
      </v-row>


      <!-- 时间轴 -->
      <v-row>
        <div ref="videoTimeline" class="flex-grow-1 cursor-move" @wheel="onTimelineScroll" color="grey-lighten-2">
          <v-sheet height="120px" @mousedown.left="startDragTimeline" @mouseup="endDragTimeline"
            @mousemove="dragTimeline" @mouseleave="endDragTimeline" color="grey-lighten-3"
            :width="Math.max(windowWidth, videoLength * timeScale)">

            <!-- 表示当前播放时间的线，始终在 timeline 中间 -->
            <v-divider :thickness="3" class="center-line border-opacity-75" vertical :height="120" color="red-lighten-1"
              :style="{ 'position': 'absolute', 'left': windowWidth / 2 + 'px', 'height': '120px' }"></v-divider>

            <!-- 表示视频开头的线 -->
            <v-divider :thickness="3" class="border-opacity-50" vertical :height="120"
              :style="{ 'position': 'absolute', 'left': - offsetX + 'px', 'height': '120px' }"></v-divider>

            <!-- 表示视频结尾的线 -->
            <v-divider :thickness="3" class="border-opacity-50" vertical :height="120"
              :style="{ 'position': 'absolute', 'left': videoLength * timeScale - offsetX + 'px', 'height': '120px' }"></v-divider>
            <!-- 时间戳 -->
            <div class="d-inline-flex timestamp no-select-text" v-for="note in notes"
              :style="{ 'position': 'absolute', 'left': note.time * timeScale - offsetX + 'px', 'height': '120px' }">
              <v-divider vertical></v-divider>
              <v-card :max-width="250" class="mx-3 my-11 d-flex align-center px-3 overflow-x-hidden"
                color="grey-lighten-1" height="36" @click="selectAndGotoNote(note)" style="float:left" @mousedown.left.stop
                @mousedown.middle.stop="startDragTimestamp($event, note)" @mouseup.stop="endDragTimestamp"
                @mousemove.stop="dragTimestamp" @click.right.prevent.stop="setNoteToCurrentTime(note)"
                @mouseleave="endDragTimestamp">
                {{ removeTitle(firstLine(note.text)) }}
              </v-card>
            </div>
          </v-sheet>
        </div>
      </v-row>
      <!-- 控制播放的一堆按钮 -->
      <v-row class="justify-center">
        <v-btn @click="debug()">debug</v-btn>
        <v-btn @click="AddNote()">Add Note</v-btn>
        <v-btn @click="DeleteCurrentNote()">Delete Current Note</v-btn>
        <v-btn @click="play()">play</v-btn>
      </v-row>
      <!-- 载入视频文件和笔记文件 -->
      <v-row>

      </v-row>
      <!-- 一些 debug 用的信息 -->
      <v-row class="px-6">
        {{ offsetX }} / {{ windowWidth }}
      </v-row>

    </v-main>
  </v-app>
</template>
<script setup>
import videojs from 'video.js';
import "video.js/dist/video-js.css"

import { MdEditor } from 'md-editor-v3';
import 'md-editor-v3/lib/preview.css';
import 'md-editor-v3/lib/style.css';

import { onMounted, onUnmounted, ref, computed } from 'vue'

const id = 'preview-only';
const scrollElement = document.documentElement;
const videoPlayer = ref(null)
const myPlayer = ref(null)
const drawer = ref(null)

const currentTime = ref(0)
const videoLength = ref(100)

const videoTimeline = ref(null)
const draggingTimeline = ref(false)
const draggingTimestamp = ref(false)

const windowWidth = ref(0)

const offsetX = ref(0)

const videoFileInput = ref(null)

const mdText = ref('# hello')

const selectedNote = ref({
  time: 0,
  text: ""
}
)



const mdEditorToolbar = ['bold',
  'underline',
  'italic',
  '-',
  'strikeThrough',
  'quote',
  'unorderedList',
  'orderedList',
  'task',
  '=',
  'previewOnly',
  // 'preview',
]

onMounted(() => {
  myPlayer.value = videojs(videoPlayer.value, {
    controls: true,
    sources: [
      {
        src: "//vjs.zencdn.net/v/oceans.mp4",
        type: 'video/mp4',
      }
    ],
    controlBar: {
      // remainingTimeDisplay: false,
      // fullscreenToggle: false,
      pictureInPictureToggle: false,
      // volumePanel: false,
      // currentTimeDisplay: false,
      // timeDivider: false,
      // durationDisplay: false
    },
    playbackRates: [0.25, 0.5, 1, 1.5, 2, 3],
    userActions: {
      doubleClick: false
    }
  }, () => {

  })
  myPlayer.value.on('load', () => {
    updateTime()
  })
  myPlayer.value.on('timeupdate', () => {
    updateTime()
  })

  updateWindowSize()
  window.addEventListener('resize', updateWindowSize)

  // document.addEventListener('keyup', globalHotkey);

})
let speed = 1;

function updateVideoSrc(event) {
  var URL = window.URL || window.webkitURL
  var file = videoFileInput.value.files[0]
  var fileURL = URL.createObjectURL(file)
  console.log(fileURL)
  myPlayer.value.src({ type: "video/mp4", src: fileURL })
}

function updateWindowSize() {
  windowWidth.value = window.innerWidth
}

function onTimelineScroll(event) {
  if (event.deltaY > 0) {
    timeScale.value -= 10;
  } else {
    timeScale.value += 10;
  }
  timeScale.value = Math.max(timeScale.value, 5)
  offsetX.value = timeToOffset(currentTime.value);
}

function debug() {
  console.log(notes.value)
}

function AddNote() {
  let newNote = { time: currentTime.value, text: "# New Note" }
  notes.value.push(newNote)
  sortNotes()
  selectedNote.value = newNote
}

function DeleteCurrentNote() {
  var index = notes.value.indexOf(selectedNote.value);
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


function globalHotkey(event) {
  // z
  if (event.keyCode === 90) {
    changeSpeed()
  }
  // space
  if (event.keyCode === 32) {
    play()
  }
}

function selectAndGotoNote(note) {
  if (timestampMoved) return;
  selectedNote.value = note
  setCurrentTime(note.time)
}

let prevX = 0

function startDragTimeline(event) {
  draggingTimeline.value = true
  prevX = event.clientX
}

function dragTimeline(event) {
  if (!draggingTimeline.value) return;
  let delta = prevX - event.clientX;
  prevX = event.clientX
  // videoTimeline.value.scrollLeft += delta;
  offsetX.value += delta;
  offsetX.value = Math.max(offsetX.value, -windowWidth.value / 2);
  offsetX.value = Math.min(offsetX.value, timeToOffset(videoLength.value))
  currentTime.value = offsetToTime(offsetX.value)
}

function endDragTimeline(event) {
  if (!draggingTimeline.value) return;
  draggingTimeline.value = false
  setCurrentTime(offsetToTime(offsetX.value))
}

const timestampOffsetX = ref(0)

let prevXts = 0

let timestampMoved = false;

function startDragTimestamp(event, note) {
  selectedNote.value = note
  draggingTimestamp.value = true
  timestampMoved = false;
  prevXts = event.clientX
  timestampOffsetX.value = timeToOffset(selectedNote.value.time)
}

function dragTimestamp(event) {
  if (!draggingTimestamp.value) return;
  timestampMoved = true;
  let delta = prevXts - event.clientX;
  prevXts = event.clientX
  // videoTimeline.value.scrollLeft += delta;
  timestampOffsetX.value -= delta;
  timestampOffsetX.value = Math.max(timestampOffsetX.value, -windowWidth.value / 2);
  timestampOffsetX.value = Math.min(timestampOffsetX.value, timeToOffset(videoLength.value))
  selectedNote.value.time = offsetToTime(timestampOffsetX.value)
}

function endDragTimestamp(event) {
  if (!draggingTimestamp.value) return;
  draggingTimestamp.value = false
  selectedNote.value.time = offsetToTime(timestampOffsetX.value)
  sortNotes()
}

function setNoteToCurrentTime(note) {
  note.time = currentTime.value
  sortNotes()
}

function sortNotes() {
  notes.value.sort((a, b) => a.time - b.time)
  console.log('sort')
}

function offsetToTime(offset) {
  return (offset + windowWidth.value / 2) / timeScale.value
}
function timeToOffset(time) {
  return -windowWidth.value / 2 + time * timeScale.value
}

function updateTime() {
  if (draggingTimeline.value) return;
  currentTime.value = myPlayer.value.currentTime()
  videoLength.value = myPlayer.value.duration()
  offsetX.value = timeToOffset(currentTime.value);
}


function changeSpeed() {
  speed = speed === 1 ? 2 : 1;
  myPlayer.value.playbackRate(speed);
}

function play() {
  if (myPlayer.value.paused()) {
    myPlayer.value.play();
  } else {
    myPlayer.value.pause();
  }
}

function setCurrentTime(time) {
  myPlayer.value.currentTime(time);
}


function nextFrame() {

}


const timeScale = ref(100)
const notes = ref([
  {
    "time": 0.946316,
    "text": "# 这是一个示例视频\n\n一只鸟"
  },
  {
    "time": 5.456315,
    "text": "# 一群鱼"
  },
  {
    "time": 12.204623,
    "text": "# 一群鸟抓鱼"
  },
  {
    "time": 34.311123,
    "text": "# 鲨鱼"
  },
  {
    "time": 42.747532,
    "text": "# 鲸鱼"
  }
])


function firstLine(str) {
  return str.split("\n")[0];
}

function removeTitle(str) {
  while (str.startsWith("#")) {
    str = str.substring(1);
  }
  return str
}

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
</style>
