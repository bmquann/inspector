<template>
  <div class="ml-40px flex items-center gap-16px my-16px">
      <!-- <n-select class="w-220px" filterable v-model:value="deviceId"></n-select> -->
      <n-input style="width: 220px !important"  v-model:value="deviceId"></n-input>
       
      <!-- <n-button>Connect</n-button> -->
      <n-button @click="dumpData">
        <template #icon>
          <Icon type="repeat" />
        </template>
        Reload
      </n-button>
    </div>
  <div class="main">
    
    <div id="left">
      <div></div>
      <section id="screen">
        <canvas
          id="fgCanvas"
          ref="fgCanvas"
          class="canvas-fg"
          v-bind:style="canvasStyle"
        ></canvas>
        <canvas id="bgCanvas" ref="bgCanvas" class="canvas-bg" v-bind:style="canvasStyle"></canvas>
        <span class="finger finger-0" style="transform: translate3d(200px, 100px, 0px)"></span>
        <img style="z-index: 10" v-if="loading" :src="loadingImg">
      </section>
    </div>
    <n-split class="h-full overflow-auto" direction="horizontal" :max="0.85" :min="0.15">
      <template #1>
        <n-scrollbar style="max-height: 100%" x-scrollable>
          <n-table :bordered="false" :single-line="false">
            <thead>
              <tr>
                <th>Prop</th>
                <th>Value</th>
              </tr>
            </thead>
            <tbody>
              <tr>
                <td>Activity</td>
                <td>{{ activity }}</td>
              </tr>
              <tr>
                <td>XPathLite</td>
                <td @click="copyToClipboard(inspectorData.xpathLite)" class="cursor-pointer ">{{ inspectorData.xpathLite }}</td>
              </tr>
              <tr>
                <td>ClassName</td>
                <td>{{ inspectorData.className }}</td>
              </tr>
              <tr v-if="elem.rect">
                  <td># rect</td>
                  <td>{{JSON.stringify(elem.rect)}}</td>
                </tr>
              <tr v-if="inspectorData.node" v-for="(item, index) in keyProps" :key="index">
                <td>{{ item }}</td>
                <td>
                  <n-tag :bordered="false" type="success" v-if="inspectorData.node?.[item] === true">true</n-tag>
                  <n-tag :bordered="false" type="error" v-else-if="inspectorData.node?.[item] === false">false</n-tag>
                  <div v-else>{{ inspectorData.node?.[item] }} </div>
                </td>
              </tr>
             
            </tbody>
          </n-table>
        </n-scrollbar>
      </template>
      <template #2>
        <div>
          <Tree
            :data="treeData"
            @deHoverNode="debounceDeHoverTree"
            @hoverNode="debounceHoverTree"
            @clickNode="clickNode"
            :selectedKey="nodeSelected?._id"
          />
        </div>
      </template>
    </n-split>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import axios from 'axios'
// import { useData } from './useData'
import Tree from './Tree.vue'
import { debounce } from 'lodash'
import loadingImg from '../../assets/loading.svg'

const keyProps = [
  'index',
  'text',
  'resourceId',
  'package',
  'description',
  'checkable',
  'checked',
  'clickable',
  'enabled',
  'focusable',
  'focused',
  'scrollable',
  'longClickable',
  'password',
  'selected',
]

const bgCanvas = ref(null)
const fgCanvas = ref(null)
const imagePool = ref([])
const activity = ref()
const inspectorData = ref({})
const deviceId = ref('988a1637384e545a5530')
const treeData = ref()
const loading= ref(false)

// hover
const nodeHovered = ref(null)
const nodeSelected = ref(null)
const cursor = ref()
const originNodeMaps = ref({})
const originNodes = ref([])
const nodeHoveredList = ref([])
const screen = ref({ bounds: {} })
const useXPathOnly = ref(false)
const mapAttrCount = ref({})

const canvasStyle = reactive({ width: '0px', height: '0px', opacity: '0.1' })
const lastScreenSize = reactive({ canvas: { width: 0, height: 0 }, screen: { width: 0, height: 0 } })

const nextImage = () => {
  return imagePool.value.pop() || new Image()
}

const drawBlobImageToScreen = (blob) => {
  return new Promise((resolve, reject) => {
    const URL = window.URL || window.webkitURL
    const BLANK_IMG = 'data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw=='
    const img = nextImage()
    const url = URL.createObjectURL(blob)

    img.onload = () => {
      const bgCanvasEl = bgCanvas.value
      const fgCanvasEl = fgCanvas.value
      const ctx = bgCanvasEl.getContext('2d')

      fgCanvasEl.width = bgCanvasEl.width = img.width
      fgCanvasEl.height = bgCanvasEl.height = img.height

      ctx.drawImage(img, 0, 0, img.width, img.height)

      resizeScreen(img)

      // Cleanup
      img.onload = img.onerror = null
      img.src = BLANK_IMG
      URL.revokeObjectURL(url)

      resolve()
    }

    img.onerror = () => {
      // Cleanup
      img.onload = img.onerror = null
      img.src = BLANK_IMG
      URL.revokeObjectURL(url)

      reject(new Error('Failed to load image'))
    }

    img.src = url
  })
}
function copyToClipboard(text) {
  navigator.clipboard.writeText(text).then(
    function () {
      // window.$message.success('Copy to clipboard successfully')
    },
    function (err) {
      window.$message.error(err.message)
    },
  )
}
const resizeScreen = (img) => {
  if (img) {
    if (lastScreenSize.canvas.width === img.width && lastScreenSize.canvas.height === img.height) {
      return
    }
  } else {
    img = lastScreenSize.canvas
    if (!img.width || !img.height) {
      return
    }
  }
  const screenDiv = document.getElementById('screen')
  lastScreenSize.canvas.width = img.width
  lastScreenSize.canvas.height = img.height
  lastScreenSize.screen.width = screenDiv.clientWidth
  lastScreenSize.screen.height = screenDiv.clientHeight

  const canvasRatio = img.width / img.height
  const screenRatio = screenDiv.clientWidth / screenDiv.clientHeight

  if (canvasRatio > screenRatio || screenDiv.clientHeight === 0) {
    canvasStyle.width = `${Math.floor(screenDiv.clientWidth)}px`
    canvasStyle.height = `${Math.floor(screenDiv.clientWidth / canvasRatio)}px`
  } else {
    canvasStyle.width = `${Math.floor(screenDiv.clientHeight * canvasRatio)}px`
    canvasStyle.height = `${Math.floor(screenDiv.clientHeight)}px`
  }
}

const b64toBlob = (b64Data, contentType = '', sliceSize = 512) => {
  const byteCharacters = atob(b64Data)
  const byteArrays = []

  for (let offset = 0; offset < byteCharacters.length; offset += sliceSize) {
    const slice = byteCharacters.slice(offset, offset + sliceSize)

    const byteNumbers = new Array(slice.length)
    for (let i = 0; i < slice.length; i++) {
      byteNumbers[i] = slice.charCodeAt(i)
    }

    const byteArray = new Uint8Array(byteNumbers)

    byteArrays.push(byteArray)
  }

  const blob = new Blob(byteArrays, { type: contentType })
  return blob
}
function drawAllNode() {
  var canvas = fgCanvas.value
  var ctx = canvas.getContext('2d')
  ctx.clearRect(0, 0, canvas.width, canvas.height)
  originNodes.value.forEach(function (node) {
    // ignore some types
    if (['Layout'].includes(node.type)) {
      return
    }
    drawNode(node, 'black', true)
  })
}
function drawNode(node, color, dashed) {
  if (!node || !node.rect) {
    return
  }
  var x = node.rect.x,
    y = node.rect.y,
    w = node.rect.width,
    h = node.rect.height
  color = color || 'black'
  var ctx = fgCanvas.value.getContext('2d')
  var rectangle = new Path2D()
  rectangle.rect(x, y, w, h)
  if (dashed) {
    ctx.lineWidth = 3
    ctx.setLineDash([8, 10])
  } else {
    ctx.lineWidth = 5
    ctx.setLineDash([])
  }
  ctx.strokeStyle = color
  ctx.stroke(rectangle)
}

function drawAllNodeFromSource(source) {
  let nodeMaps = (originNodeMaps.value = {})

  function sourceToNodes(source) {
    let node = Object.assign({}, source) //, { children: undefined });
    nodeMaps[node._id] = node
    let nodes = [node]
    if (source.children) {
      source.children.forEach(function (s) {
        s._parentId = node._id
        nodes = nodes.concat(sourceToNodes(s))
      })
    }
    return nodes
  }
  originNodes.value = sourceToNodes(source) //ret.nodes;
  drawAllNode()
  canvasStyle.opacity = 1.0
}
function drawRefresh() {
  drawAllNode()
  if (nodeHovered.value) {
    drawNode(nodeHovered.value, 'blue')
  }
  if (nodeSelected.value) {
    drawNode(nodeSelected.value, 'red')
  }
}

function findNodesByPosition(pos) {
  function isInside(node, x, y) {
    if (!node.rect) {
      return false
    }
    var lx = node.rect.x,
      ly = node.rect.y,
      rx = node.rect.width + lx,
      ry = node.rect.height + ly
    return lx < x && x < rx && ly < y && y < ry
  }

  function nodeArea(node) {
    return node.rect.width * node.rect.height
  }

  let activeNodes = originNodes.value.filter(function (node) {
    if (!isInside(node, pos.x, pos.y)) {
      return false
    }
    // skip some types
    if (['Layout', 'Sprite'].includes(node.type)) {
      return false
    }
    return true
  })

  activeNodes.sort((node1, node2) => {
    return nodeArea(node1) - nodeArea(node2)
  })
  return activeNodes
}

function drawHoverNode(pos) {
  let hoveredNodes = findNodesByPosition(pos)
  let node = hoveredNodes[0]
  nodeHovered.value = node

  hoveredNodes.forEach((node) => {
    drawNode(node, 'green')
  })
  drawNode(nodeHovered.value, 'blue')
}
const elem = computed(() => {
  return nodeSelected.value || {}
})

function getAttrCount(collectionKey, key) {
  // eg: getAttrCount("resource-id", "tv_scan_text")
  let mapCount = mapAttrCount.value[collectionKey]
  if (!mapCount) {
    return 0
  }
  return mapCount[key] || 0
}
function incrAttrCount(collectionKey, key) {
  if (!mapAttrCount.value.hasOwnProperty(collectionKey)) {
    mapAttrCount.value[collectionKey] = {}
  }
  let count = mapAttrCount.value[collectionKey][key] || 0
  mapAttrCount.value[collectionKey][key] = count + 1
}

function elemXPathLite() {
  // scan nodes
  mapAttrCount.value = {}
  originNodes.value.forEach((n) => {
    incrAttrCount('label', n.label)
    incrAttrCount('resourceId', n.resourceId)
    incrAttrCount('text', n.text)
    incrAttrCount('_type', n._type)
    incrAttrCount('description', n.description)
  })

  let node = elem.value
  const array = []
  while (node && node._parentId) {
    const parent = originNodeMaps.value[node._parentId]
    if (getAttrCount('label', node.label) === 1) {
      array.push(`*[@label="${node.label}"]`)
      break
    } else if (getAttrCount('resourceId', node.resourceId) === 1) {
      array.push(`*[@resource-id="${node.resourceId}"]`)
      break
    } else if (getAttrCount('text', node.text) === 1) {
      array.push(`*[@text="${node.text}"]`)
      break
    } else if (getAttrCount('description', node.description) === 1) {
      array.push(`*[@content-desc="${node.description}"]`)
      break
    } else if (getAttrCount('_type', node._type) === 1) {
      array.push(`${node._type}`)
      break
    } else if (!parent) {
      array.push(`${node._type}`)
    } else {
      let index = 0
      parent.children.some((n) => {
        if (n._type == node._type) {
          index++
        }
        return n._id == node._id
      })
      array.push(`${node._type}[${index}]`)
    }
    node = parent
  }
  return `//${array.reverse().join('/')}`
}
function elemXPathFull() {
  let node = elem.value
  const array = []
  while (node && node._parentId) {
    let parent = originNodeMaps.value[node._parentId]

    let index = 0
    parent.children.some((n) => {
      if (n._type == node._type) {
        index++
      }
      return n._id == node._id
    })

    array.push(`${node._type}[${index}]`)
    node = parent
  }
  return `//${array.reverse().join('/')}`
}

function findNodes(kwargs) {
  return originNodes.value.filter((node) => {
    for (const [k, v] of Object.entries(kwargs)) {
      if (node[k] !== v) {
        return false
      }
    }
    return true
  })
}
function generateNodeSelectorKwargs(node) {
  // iOS: name, label, className
  // Android: text, description, resourceId, className
  let kwargs = {}
  ;['label', 'resourceId', 'name', 'text', 'type', 'tag', 'description', 'className'].some((key) => {
    if (!node[key]) {
      return false
    }
    kwargs[key] = node[key]
    return findNodes(kwargs).length === 1
  })

  const matchedNodes = findNodes(kwargs)
  const nodeCount = matchedNodes.length
  if (nodeCount > 1) {
    kwargs['instance'] = matchedNodes.findIndex((n) => {
      return n._id == node._id
    })
  }
  kwargs['_count'] = nodeCount
  return kwargs
}
function _combineKeyValue(key, value) {
  if (typeof value === 'string') {
    value = `"${value}"`
  }
  return key + '=' + value
}
function generateNodeSelectorCode(node) {
  if (useXPathOnly.value) {
    return `d.xpath('${elemXPathLite}')`
  }
  let kwargs = generateNodeSelectorKwargs(node)
  if (kwargs._count === 1) {
    const array = []
    for (const [key, value] of Object.entries(kwargs)) {
      if (key.startsWith('_')) {
        continue
      }
      array.push(_combineKeyValue(key, value))
    }
    return `d(${array.join(', ')})`
  }
  return `d.xpath('${elemXPathLite}')`
}

function mouseDownListener(e) {
  // Skip secondary click
  if (e.which === 3) {
    return
  }
  e.preventDefault()

  // fakePinch = e.altKey
  calculateBounds()
  // startMousing()

  var pressure = 0.5
  activeFinger(0, e.pageX, e.pageY, pressure)

  if (nodeHovered.value) {
    nodeSelected.value = nodeHovered.value
    drawAllNode()
    // self.drawHoverNode(pos);
    drawNode(nodeSelected.value, 'red')
    var generatedCode = generateNodeSelectorCode(nodeSelected.value)
    inspectorData.value = {
      xpathLite: elemXPathLite(),
      xpathFull: elemXPathFull(),
      className: elem.value._type,
      activity: activity.value,
      node: nodeSelected.value,
    }
    // copyToClipboard(generatedCode)

    // self.$jstree.jstree(true)._open_to('#' + self.nodeHovered._id)
    // document.getElementById(nodeHovered.value._id).scrollIntoView(false)
  }
  // self.touchDown(0, x / screen.bounds.w, y / screen.bounds.h, pressure);

   fgCanvas.value.removeEventListener('mouseleave', mouseHoverLeaveListener)
   fgCanvas.value.removeEventListener('mousemove', mouseHoverListener)
   fgCanvas.value.addEventListener('mousemove', mouseMoveListener)
  document.addEventListener('mouseup', mouseUpListener)
}

function activeFinger(index, x, y, pressure) {
  const scale = 0.5 + pressure
  document.querySelector(`.finger-${index}`).classList.add('active')
  document.querySelector(`.finger-${index}`).style.transform = `translate3d(${x}px, ${y}px, 0)`
}

function deactiveFinger(index) {
  document.querySelector(`.finger-${index}`).classList.remove('active')
}

function mouseMoveListener(event) {
  if (event.which === 3) return // Skip secondary click
  event.preventDefault()
  // activeFinger(0, event.pageX, event.pageY, 0.5)
}

function mouseHoverLeaveListener() {
  nodeHovered.value = null
  drawRefresh()
}

function mouseHoverListener(e) {
  // Skip secondary click
  if (e.which === 3) {
    return
  }
  e.preventDefault()
  // startMousing()

  var pos = coord(e)

  nodeHoveredList.value = findNodesByPosition(pos)
  nodeHovered.value = nodeHoveredList.value[0]

  drawRefresh()

  if (cursor.value?.px) {
    markPosition(cursor.value)
  }
}

function calculateBounds() {
  var el = fgCanvas.value
  screen.value.bounds.w = el.offsetWidth
  screen.value.bounds.h = el.offsetHeight
  screen.value.bounds.x = 0
  screen.value.bounds.y = 0

  while (el.offsetParent) {
    screen.value.bounds.x += el.offsetLeft
    screen.value.bounds.y += el.offsetTop
    el = el.offsetParent
  }
}

function coord(e) {
  calculateBounds()
  var x = e.pageX - screen.value.bounds.x
  var y = e.pageY - screen.value.bounds.y
  var px = x / screen.value.bounds.w
  var py = y / screen.value.bounds.h
  return {
    px: px,
    py: py,
    x: Math.floor(px * fgCanvas.value.width),
    y: Math.floor(py * fgCanvas.value.height),
  }
}

function markPosition(pos) {
  var ctx = fgCanvas.value.getContext('2d')
  ctx.fillStyle = '#ff0000' // redmouseUpListener
  ctx.beginPath()
  ctx.arc(pos.x, pos.y, 12, 0, 2 * Math.PI)
  ctx.closePath()
  ctx.fill()

  ctx.fillStyle = '#fff' // white
  ctx.beginPath()
  ctx.arc(pos.x, pos.y, 8, 0, 2 * Math.PI)
  ctx.closePath()
  ctx.fill()
}

function mouseUpListener(e) {
  // Skip secondary click
  if (e.which === 3) {
    return
  }
  e.preventDefault()

  var pos = coord(e)
  // change precision
  pos.px = Math.floor(pos.px * 1000) / 1000
  pos.py = Math.floor(pos.py * 1000) / 1000
  pos.x = Math.floor(pos.px * fgCanvas.value.width)
  pos.y = Math.floor(pos.py * fgCanvas.value.height)
  cursor.value = pos

  nodeHovered.value = null
  markPosition(cursor.value)

  stopMousing()
}

function stopMousing() {
  fgCanvas.value.removeEventListener('mousemove', mouseMoveListener)
  fgCanvas.value.addEventListener('mousemove', mouseHoverListener)
  fgCanvas.value.addEventListener('mouseleave', mouseHoverLeaveListener)
  document.removeEventListener('mouseup', mouseUpListener)
  deactiveFinger(0)
}

const debounceHoverTree = debounce((id) => {
  var node = originNodeMaps.value[id]
  if (node) {
    nodeHovered.value = node
    drawRefresh()
  }
}, 200)

const debounceDeHoverTree = debounce((id) => {
  nodeHovered.value = null
  drawRefresh()
}, 200)

const clickNode = (id) => {
  var node = originNodeMaps.value[id]
  if (node) {
    nodeSelected.value = node
    drawAllNode()
    drawNode(node, 'red')

    inspectorData.value = {
      xpathLite: elemXPathLite(),
      xpathFull: elemXPathFull(),
      className: elem.value._type,
      activity: activity.value,
      node: nodeSelected.value,
    }
    var generatedCode = generateNodeSelectorCode(
      nodeSelected.value
    );
    // copyToClipboard(generatedCode);

    // self.generatedCode = generatedCode;
  }
}


const dumpData =async  () =>{
  loading.value = true;
 await axios.get(`http://localhost:17310/api/v1/devices/Android%3A${deviceId.value}/screenshot`).then((res) => {
    var blob = b64toBlob(res.data.data, 'image/jpeg')
    drawBlobImageToScreen(blob)
  })
 await axios.get(`http://localhost:17310/api/v2/devices/Android%3A${deviceId.value}/hierarchy`).then((res) => {
    activity.value = res.data.activity
    treeData.value = res.data.jsonHierarchy
    drawAllNodeFromSource(res.data.jsonHierarchy)
  })
  loading.value = false
}
onMounted(() => {
  dumpData()
  fgCanvas.value.addEventListener('mousemove', mouseMoveListener)
  fgCanvas.value.addEventListener('mousemove', mouseHoverListener)
  fgCanvas.value.addEventListener('mousedown', mouseDownListener)
  fgCanvas.value.addEventListener('mouseleave', mouseHoverLeaveListener)
})
</script>

<style scoped>
.canvas-bg {
  z-index: 0;
  position: absolute;
}

.canvas-fg {
  z-index: 1;
  position: absolute;
}

.finger {
  position: absolute;
  border-style: solid;
  border-radius: 50%;
  border-color: white;
  border-width: 0mm;
  width: 6mm;
  height: 6mm;
  top: -3mm;
  left: -3mm;
  opacity: 0.7;
  pointer-events: none;
  background: white;
  /*#464646;*/
  /*background: red;*/
  display: none;
}

.finger.active {
  display: block;
  border-color: #464646;
  border-width: 1mm;
}
#screen {
  display: flex;
  flex-direction: row;
  align-items: center;
  justify-content: center;
  flex: 1;
  background-color: gray;
  height: 100%;
  width: 100%;
}

.main {
  width: 100%;
  display: flex;
  flex: 1;
  gap: 40px;
  height: 90vh;

  border: 1px solid rgb(224, 224, 230);
}

#app {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
}
#left {
  width: 40%;
}
</style>
