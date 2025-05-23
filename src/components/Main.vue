<template>
  <v-app id="main">
    <v-navigation-drawer v-model="drawer">
      <v-list>
        <v-list-item v-for="(note, i) in notes" :key="i" :subtitle="toHHMMSS(note.time)"
          :title="removeTitle(firstLine(note.text))" link @click="selectAndGotoNote(note)">

        </v-list-item>
      </v-list>
    </v-navigation-drawer>
    <v-main>
      <!-- 视频和右侧的笔记框 -->
      <v-row class="px-4 justify-center pt-3">
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
        {{ toHHMMSS(currentTime) }} / {{ toHHMMSS(videoLength) }} [{{ speedList[currentSpeedIndex] }}x]
      </v-row>


      <!-- 时间轴 -->
      <v-row>
        <div ref="videoTimeline" class="flex-grow-1 cursor-move" @wheel="onTimelineScroll" color="grey-lighten-2">
          <v-sheet :height="timelineHeight" @mousedown.left="startDragTimeline" color="grey-lighten-3"
            @click.right.prevent.stop :width="Math.max(windowWidth, videoLength * timeScale)">

            <!-- 表示当前播放时间的线，始终在 timeline 中间 -->
            <v-divider :thickness="3" class="center-line border-opacity-75" vertical :height="timelineHeight"
              color="red-lighten-1"
              :style="{ 'position': 'absolute', 'left': windowWidth / 2 + 'px', 'height': timelineHeight }"></v-divider>

            <!-- 表示视频开头的线 -->
            <v-divider :thickness="3" class="border-opacity-50" vertical :height="timelineHeight"
              :style="{ 'position': 'absolute', 'left': - offsetX + 'px', 'height': timelineHeight }"></v-divider>

            <!-- 表示视频结尾的线 -->
            <v-divider :thickness="3" class="border-opacity-50" vertical :height="timelineHeight"
              :style="{ 'position': 'absolute', 'left': videoLength * timeScale - offsetX + 'px', 'height': timelineHeight }"></v-divider>
            <!-- 时间戳 -->
            <div class="d-inline-flex timestamp no-select-text" v-for="note in notes"
              :style="{ 'position': 'absolute', 'left': note.time * timeScale - offsetX + 'px', 'height': timelineHeight }">
              <v-divider vertical></v-divider>
              <v-card :max-width="250" class="mx-3 my-8 d-flex align-center px-3 overflow-x-hidden"
                color="grey-lighten-1" height="36" @click="selectAndGotoNote(note)" style="float:left"
                @mousedown.left.stop @mousedown.right.stop="startDragTimestamp($event, note)"
                @click.middle.prevent.stop="setNoteToCurrentTime(note)" @click.right.prevent.stop>
                {{ removeTitle(firstLine(note.text)) }}
              </v-card>
            </div>
          </v-sheet>
        </div>
      </v-row>
      <!-- 控制播放的一堆按钮 -->
      <v-row class="justify-center mt-5">
        <v-btn @click="drawer = !drawer" icon="mdi-pencil"></v-btn>
        <v-btn @click="addDefaultNote()" icon="mdi-pencil"></v-btn>
        <v-btn @click="deleteCurrentNote()" icon="mdi-delete"></v-btn>
        <v-btn @click="saveNoteJSON()" icon="mdi-download-outline"></v-btn>
        <v-btn @click="debug()">debug</v-btn>
        <v-btn @click="play()" color="blue" icon="mdi-play" size="large"></v-btn>
      </v-row>
      <!-- 载入视频文件和笔记文件 -->
      <v-row>
        <v-col :cols="2">
          <v-file-input ref="videoFileInput" label="拖拽加载视频文件..." accept=".mp4,.mov,.webm"
            @change="updateVideoSrc"></v-file-input>
        </v-col>
        <v-col :cols="2">
          <v-file-input :height="20" ref="noteFileInput" label="笔记文件" accept=".txt"></v-file-input>
        </v-col>
      </v-row>
      <!-- 一些 debug 用的信息 -->
      <!-- <v-row class="px-6">
        {{ offsetX }} / {{ windowWidth }}
      </v-row> -->

    </v-main>
  </v-app>
</template>
<script setup>
import videojs from 'video.js';
import "video.js/dist/video-js.css"

import { MdEditor } from 'md-editor-v3';
import 'md-editor-v3/lib/preview.css';
import 'md-editor-v3/lib/style.css';

import { onMounted, onUnmounted, ref, computed, watch } from 'vue'

import { nanoid } from 'nanoid'

const timelineHeight = '100px'

const videoPlayer = ref(null)
const player = ref(null)
const drawer = ref(true)

const currentTime = ref(0)
const videoLength = ref(100)

const videoTimeline = ref(null)
const draggingTimeline = ref(false)
const draggingTimestamp = ref(false)

const windowWidth = ref(0)

const offsetX = ref(0)

const videoFileInput = ref(null)

const selectedNote = ref({
  time: 0,
  text: ""
}
)

onMounted(() => {
  player.value = videojs(videoPlayer.value, {
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
    playbackRates: speedList,
    userActions: {
      doubleClick: false
    }
  }, () => {
    seek(0)
  })
  player.value.on('timeupdate', () => {
    updateTime()
  })

  updateWindowSize()
  window.addEventListener('resize', updateWindowSize)

  document.addEventListener('dragover', (e) => {
    e.preventDefault()
  });

  document.addEventListener('drop', (e) => {
    videoFileInput.value.files = e.dataTransfer.files;
    console.log(e.dataTransfer.files)
    e.preventDefault()
  });

  // 避免 markdown 编辑器在不聚焦时响应 Ctrl-Z
  document.addEventListener('keydown', function (event) {
    if (event.ctrlKey && event.key === 'z') {
      if (notUsingInput.value) {
        event.preventDefault();
      }
    }
  });

  // 拖拽时间轴和时间戳，只有 mousedown 不是全局

  document.addEventListener('mousemove', function (event) {
    if (draggingTimeline.value) {
      dragTimeline(event)
      event.preventDefault()
    }
    if (draggingTimestamp.value) {
      dragTimestamp(event)
      event.preventDefault()
    }
  });

  document.addEventListener('mouseup', function (event) {
    if (draggingTimeline.value) {
      stopDragTimeline(event)
    }
    if (draggingTimestamp.value) {
      stopDragTimestamp(event)
    }
  });
})

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
)

const keys = useMagicKeys()

// 【播放速度控制】

const speedList = [0.5, 0.75, 1, 1.5, 2, 3]
const defaultSpeedIndex = 2
const currentSpeedIndex = ref(2)
const storedSpeedIndex = ref(4)

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
  console.log(activeElement.value)
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

// 注册指令添加到栈中
function startRegisterMove(note) {
  let time = note.time
  let id = note.id
  moveCmd = {
    undo: () => setNoteTimeByID(id, time),
    redo: null
  }
}

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

// 根据 id 找到要操作的元素，避免 add delete 之后丢失引用

function findNoteByID(id) {
  return notes.value.find(note => note.id == id)
}

function deleteNoteByID(id) {
  let note = findNoteByID(id)
  if (note) {
    deleteNote(note, false)
  }
}

function setNoteTimeByID(id, time) {
  let note = findNoteByID(id)
  if (note) {
    note.time = time
  }
  sortNotes()
}

// Markdown 编辑器（md-editor-v3）工具栏的按钮布局
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
  'preview',
  'previewOnly',
]

//
function updateVideoSrc(event) {
  var URL = window.URL || window.webkitURL
  var file = videoFileInput.value.files[0]
  var fileURL = URL.createObjectURL(file)
  console.log(fileURL)
  player.value.src({ type: "video/mp4", src: fileURL })
  seek(0)
}

function updateWindowSize() {
  windowWidth.value = window.innerWidth
}

// 滚轮调整时间轴的横向缩放
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

// 【添加和删除笔记】

function addNote(newNote, regUndo = false) {
  notes.value.push(newNote)
  if (regUndo) {
    let noteCopy = { ...newNote }
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
  addNote({ id: nanoid(), time: currentTime.value, text: "# New Note" }, true)
}

function deleteNote(note, regUndo = false) {
  var index = notes.value.indexOf(note);
  if (regUndo) {
    console.log({ ...note })
    let noteCopy = { ...note }
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

function selectAndGotoNote(note) {
  selectedNote.value = note
  seek(note.time)
}

// 拖拽时间轴

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

function stopDragTimeline(event) {
  if (!draggingTimeline.value) return;
  draggingTimeline.value = false
  seek(offsetToTime(offsetX.value))
  selectNoteByCurrentTime()
}

// 拖拽时间戳

const timestampOffsetX = ref(0)

let prevXts = 0

let draggedNote = null;

function startDragTimestamp(event, note) {
  draggedNote = note
  draggingTimestamp.value = true
  prevXts = event.clientX
  timestampOffsetX.value = timeToOffset(draggedNote.time)
  startRegisterMove(note)
}

function dragTimestamp(event) {
  if (!draggingTimestamp.value) return;
  let delta = prevXts - event.clientX;
  prevXts = event.clientX
  // videoTimeline.value.scrollLeft += delta;
  timestampOffsetX.value -= delta;
  timestampOffsetX.value = Math.max(timestampOffsetX.value, -windowWidth.value / 2);
  timestampOffsetX.value = Math.min(timestampOffsetX.value, timeToOffset(videoLength.value))
  draggedNote.time = offsetToTime(timestampOffsetX.value)
}

function stopDragTimestamp(event) {
  if (!draggingTimestamp.value) return;
  draggingTimestamp.value = false
  draggedNote.time = offsetToTime(timestampOffsetX.value)
  finishRegisterMove(draggedNote)
  sortNotes()
  selectNoteByCurrentTime()
}

function setNoteToCurrentTime(note) {
  note.time = currentTime.value
  sortNotes()
}

function sortNotes() {
  notes.value.sort((a, b) => a.time - b.time)
  console.log('sort')
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

function offsetToTime(offset) {
  return (offset + windowWidth.value / 2) / timeScale.value
}
function timeToOffset(time) {
  return -windowWidth.value / 2 + time * timeScale.value
}

function updateTime() {
  if (draggingTimeline.value) return;
  currentTime.value = player.value.currentTime()
  videoLength.value = player.value.duration()
  offsetX.value = timeToOffset(currentTime.value);
  selectNoteByCurrentTime()
}

function play() {
  if (player.value.paused()) {
    player.value.play();
  } else {
    player.value.pause();
  }
}

function seek(time) {
  player.value.currentTime(time.clamp(0, videoLength.value));
}

const timeScale = ref(100)

const notes = ref([
  {
    "id": '1',
    "time": 0.946316,
    "text": "# 这是一个示例视频\n\n一只鸟"
  },
  {
    "id": '2',
    "time": 5.456315,
    "text": "# 一群鱼"
  },
  {
    "id": '3',
    "time": 12.204623,
    "text": "# 一群鸟抓鱼"
  },
  {
    "id": '4',
    "time": 34.311123,
    "text": "# 鲨鱼"
  },
  {
    "id": '5',
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

function download(content, fileName, contentType) {
  var a = document.createElement("a");
  var file = new Blob([content], { type: contentType });
  a.href = URL.createObjectURL(file);
  a.download = fileName;
  a.click();
}

function saveNoteJSON() {
  download(JSON.stringify(notes.value), 'note.txt', 'text/plain');
}

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
