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
        <v-col :cols="8">
          <v-sheet class="d-flex" height="70vh" :elevation="8">
            <video ref="videoPlayer" id="my-player" class="video-js" controls preload="auto">
            </video>
            <div id="md-content" class="pa-10" v-if="!showVideo" v-html="mdContent"></div>
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
      <v-row>
        <v-col class="pt-0 pl-7">
          {{ videoFileName }}
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
              ℹ️ DL: "{{ videoFileName }}.txt"
            </template>
          </template>
          <template v-else>
            ℹ️ DL: "{{ videoFileName }}.txt"。
          </template>
        </v-col>
      </v-row>


      <!-- 时间轴 -->
      <v-row>
        <div ref="videoTimeline" class="flex-grow-1 cursor-move" @wheel.prevent="onTimelineScroll"
          color="grey-lighten-2">
          <v-sheet :height="timelineHeight" @mousedown.left="startDragTimeline" color="grey-lighten-3"
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
        <v-col :cols="8">
          <v-row>
            <v-col :cols="3">
            </v-col>
            <!-- 一堆按钮 -->
            <v-col :cols="6" class="d-flex align-center justify-center">
              <!-- 用 mousedown.prevent 阻止按钮在点击后获取焦点，否则按下空格时聚焦的按钮也会被触发 -->
              <v-btn @mousedown.prevent @click="drawer = !drawer" class="mx-1" icon="mdi-list-box-outline"
                size="x-large"></v-btn>
              <v-btn @mousedown.prevent @click="showVideo = !showVideo" class="mx-1"
                :icon="showVideo ? 'mdi-television-off' : 'mdi-television'" size="x-large"></v-btn>
              <v-divider vertical class="mx-5"></v-divider>
              <v-btn @mousedown.prevent @click="seekPreviousNote()" class="mx-1" icon="mdi-skip-backward-outline"
                size="x-large"></v-btn>
              <v-btn @mousedown.prevent @click="play()" class="mx-1"
                :icon="isPlaying ? 'mdi-pause' : 'mdi-play-outline'" size="x-large"></v-btn>
              <v-btn @mousedown.prevent @click="seekNextNote()" class="mx-1" icon="mdi-skip-forward-outline"
                size="x-large"></v-btn>
              <v-divider vertical class="mx-5"></v-divider>
              <v-btn @mousedown.prevent @click="addDefaultNote()" class="mx-1" icon="mdi-text-box-plus-outline"
                size="x-large"></v-btn>
              <v-btn @mousedown.prevent @click="deleteCurrentNote()" class="mx-1" icon="mdi-delete-outline"
                size="x-large"></v-btn>
              <v-divider vertical class="mx-5"></v-divider>
              <v-btn @mousedown.prevent @click="saveNoteJSON()" class="mx-1" icon="mdi-content-save-outline"
                size="x-large"></v-btn>
              <v-btn @mousedown.prevent @click="loadFromURL()" class="mx-1" icon="mdi-link-plus"
                size="x-large"></v-btn>
            </v-col>
            <v-col>
            </v-col>
          </v-row>
        </v-col>

        <!-- BPM 与偏移量调整 -->
        <v-col :cols="4">
          <v-row class="">
            <v-col :cols="8" class="mt-5">
              <v-row class="px-10">
                <v-number-input class="w-25" v-model="tlMarkerBPM" label="BPM" control-variant="stacked" :min="30"
                  :max="300" :precision="1"></v-number-input>
                <v-number-input class="w-25" v-model="tlMarkerBeat" label="Beat" control-variant="stacked" :min="1"
                  :max="20"></v-number-input>
              </v-row>
              <v-row class="w-100 pl-7 my-0">
                <v-slider v-model="tlMarkerOffset" :min="0" :max="tlMarkerBeat"></v-slider>
              </v-row>
            </v-col>
            <v-col>
              <v-btn @mousedown.prevent @click="useMetronome = !useMetronome" class="mx-1" icon="mdi-metronome"
                size="x-large"></v-btn>
            </v-col>
          </v-row>
        </v-col>
      </v-row>
      <!-- 一些 debug 用的信息 -->
      <v-row v-if="debugMode" class="px-6">
        <v-btn @click="debug()">debug</v-btn>
        {{ offsetX }} / {{ windowWidth }} / {{ tlMarkerDensity }} / {{ tlBigMarkerDensity }} / {{ tlBigMarkerCull }}<br>
        timescale: {{ timeScale }} / showVideo {{ showVideo }} {{ mdContent }}<br>
        timelineMarkerCount: {{ tlMarkerCount }} bpm {{ tlMarkerBPM }} offset {{ tlMarkerOffset }}

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

import { onMounted, ref, computed, watch, onUnmounted } from 'vue'

import { nanoid } from 'nanoid'

import { marked } from 'marked';

const debugMode = ref(true)

const videoFileName = ref('SampleVideo')

const videoPlayer = ref(null)
const player = ref(null)
const drawer = ref(true)

const currentTime = ref(0)
const videoLength = ref(100)

const videoTimeline = ref(null)
const draggingTimeline = ref(false)
const draggingTimestamp = ref(false)

const mdContent = ref('')

const selectedNote = ref({
  id: "default",
  time: 0,
  text: ""
}
)

watch(selectedNote, (note) => {
  if (note != null) {
    mdContent.value = marked.parse(note.text)
  }
}, {
  deep: true
})

// 【时间轴与刻度】
const timelineHeight = ref(100)
// 刻度的高度比例
const tlMarkerRatio = ref(0.15)
const tlMarkerMinorRatio = ref(0.08)
// 用 BPM 定义刻度的间隔
const tlMarkerBPM = ref(60)
const tlMarkerInterval = computed(() => 60.0 / tlMarkerBPM.value)
// 大刻度
const tlMarkerBeat = ref(4)
// 刻度的整体时间偏移
const tlMarkerOffset = ref(0)
// 覆盖整个视频时间需要的刻度数量
const tlMarkerCount = computed(() => {
  return Math.ceil(videoLength.value / tlMarkerInterval.value)
})

// 剔除屏幕外的时间轴刻度、刻度密集时降它们间隔隐藏
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
watch(showVideo, (v) => {
  document.getElementById('my-player').style.setProperty('display', v ? "inline-block" : "none")
})

const useMetronome = ref(false)
const metronomeStart = ref(Date.now())


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

  player.value.on('pause', () => {
    isPlaying.value = false
  })
  player.value.on('play', () => {
    isPlaying.value = true
    metronomeStart.value = Date.now() - player.value.currentTime() * 1000
  })
  player.value.on('ratechange', () => {
    currentSpeedIndex.value = speedList.findIndex(n => n == player.value.playbackRate())
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

  var prevTime = 0
  var delta = 0
  var nextBeat = 0
  var nextBeatTime = 0
  setInterval(function () {
    delta = (Date.now() - metronomeStart.value) / 1000 - tlMarkerOffset.value * tlMarkerInterval.value;
    nextBeat = Math.ceil(prevTime / tlMarkerInterval.value)
    nextBeatTime = nextBeat * tlMarkerInterval.value
    if (delta > nextBeatTime) {
      if (useMetronome.value && isPlaying.value) {
        playMetronome(nextBeat % tlMarkerBeat.value == 0)
      }
    }
    prevTime = delta
    // alternatively just show wall clock time:
  }, 10); // 每 0.01s 更新一下计时器
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

function globalMouseUp(event) {
  if (draggingTimeline.value) {
    finishDragTimeline(event)
  }
  if (draggingTimestamp.value) {
    finishDragTimestamp(event)
  }
}

function preventHotkeys(event) {
  if (event.ctrlKey && (event.key === 'z' || event.key === 's')) {
    if (notUsingInput.value) {
      event.preventDefault();
    }
  }
}

const noteFileHandle = ref(null)

async function dropFile(event) {
  const items = event.dataTransfer.items;
  Array.from(event.dataTransfer.files).forEach(file => {
    loadFile(file)
  });
  event.preventDefault()
  // 如果浏览器支持使用 FileSystemHandle，记录笔记文件的 handle，以便保存时直接保存到原文件中
  for (const item of items) {
    console.log(item)
    if (item.kind === 'file' && item.type == 'text/plain' && item.getAsFileSystemHandle) {
      const handle = await item.getAsFileSystemHandle();
      console.log(handle)
      if (handle.kind === 'file') {
        noteFileHandle.value = handle;
        console.log(noteFileHandle.value)
        // await handleFile(handle);
      } else {
        noteFileHandle.value = null
      }
    }
  }
}

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
)

const keys = useMagicKeys()


// Ctrl-S 保存，这个可以在编辑器里触发
const keyCtrlS = keys['ctrl+s']
whenever(keyCtrlS, (v) => {
  saveNoteJSON();
})



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
  metronomeStart.value = Date.now() - player.value.currentTime() * 1000
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

function updateWindowSize() {
  windowWidth.value = window.innerWidth
}

const tsList = [0.2, 0.3, 0.5, 1, 2, 3, 4, 5, 10, 20, 30, 40, 50, 80, 150, 300]

// 滚轮调整时间轴的横向缩放
function onTimelineScroll(event) {

  if (event.deltaY > 0) {
    timeScale.value = tsList.findLast(ts => ts < timeScale.value) ?? tsList[0]
  } else {
    timeScale.value = tsList.find(ts => ts > timeScale.value) ?? tsList[tsList.length - 1]
  }
  // timeScale.value = timeScale.value.clamp(1, 300)
  offsetX.value = timeToOffset(currentTime.value);
}

function debug() {
  console.log(notes.value)
  var audio = new Audio('metronome_down.wav');
  audio.play();
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
  addNote(new Note(currentTime.value, "# " + (notes.value.length + 1)), true)
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

function onTimestampClick(event, note) {
  if (event.clientX != startDragXts) {
    return;
  } else {
    selectAndSeekNote(note)
  }
}

function selectAndSeekNote(note) {
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

function finishDragTimeline(event) {
  if (!draggingTimeline.value) return;
  draggingTimeline.value = false
  seek(offsetToTime(offsetX.value))
  selectNoteByCurrentTime()
}

// 拖拽时间戳

const timestampOffsetX = ref(0)

let prevXts = 0

const draggedNote = ref(null);
let startDragXts = 0

function startDragTimestamp(event, note) {
  draggedNote.value = note
  draggingTimestamp.value = true
  prevXts = event.clientX
  startDragXts = event.clientX
  timestampOffsetX.value = timeToOffset(draggedNote.value.time)
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
  draggedNote.value.time = offsetToTime(timestampOffsetX.value)
}

function finishDragTimestamp(event) {
  if (!draggingTimestamp.value) return;
  draggingTimestamp.value = false
  if (startDragXts == event.clientX) return;
  draggedNote.time = offsetToTime(timestampOffsetX.value)
  finishRegisterMove(draggedNote)
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

function playMetronome(firstBeat) {
  var audio = new Audio(firstBeat ? 'metronome_up.wav' : 'metronome_down.wav');
  audio.play();
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
  metronomeStart.value = Date.now() - time * 1000
}

const timeScale = ref(100)

const notes = ref([])


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

// 【文件的读取和保存】

function loadFile(file) {
  var URL = window.URL || window.webkitURL
  var fileURL = URL.createObjectURL(file)
  if (file.type == "text/plain") {
    loadNoteJSON(file)
  } else {
    console.log(file)
    videoFileName.value = file.name
    player.value.src({ type: file.type, src: fileURL })
    seek(0)
  }
}

function download(content, fileName, contentType) {
  var a = document.createElement("a");
  var file = new Blob([content], { type: contentType });
  a.href = URL.createObjectURL(file);
  a.download = fileName;
  a.click();
}

async function saveNoteJSON() {
  let saveData = makeSaveData()
  let json = JSON.stringify(saveData, null, 4)

  if (noteFileHandle.value != null) {
    console.log('write')
    const writable = await noteFileHandle.value.createWritable();
    await writable.write(json);
    await writable.close();
  } else {
    download(json, videoFileName.value + '.txt', 'text/plain');
  }
}

function loadNoteJSON(file) {
  var reader = new FileReader();
  reader.onload = event => {
    var saveData = JSON.parse(event.target.result);
    loadSaveData(saveData)
    seek(0)
    selectNoteByCurrentTime()
  };
  reader.readAsText(file);
}

function loadFromURL(){

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

class Note {
  constructor(time, text) {
    this.id = nanoid()
    this.time = time
    this.text = text
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

function makeSaveData() {
  const saveData = {
    markers: {
      bpm: tlMarkerBPM.value,
      beat: tlMarkerBeat.value,
      offset: tlMarkerOffset.value
    },
    notes: notes.value
  }
  return saveData
}

function loadSaveData(saveData) {
  tlMarkerBPM.value = saveData.markers.bpm
  tlMarkerBeat.value = saveData.markers.beat
  tlMarkerOffset.value = saveData.markers.offset

  saveData.notes.forEach(note => Object.setPrototypeOf(note, Note.prototype))
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
  font-size: 1.5em;
  padding: 0;
  margin-top: -6px;
}

div#md-content h1 {
  font-size: 5em
}
</style>
