<template>
  <div class="page">
    <div id="cesiumContainer" class="cesium-container"></div>

    <!-- 控件区 -->
    <div id="controls">
      <button id="btn-geojson" @click="openFile('geojson')">加载 GeoJSON</button>
      <input ref="geojsonInput" type="file" accept=".geojson,application/geo+json" style="display:none" @change="onFileChange($event,'geojson')"/>

      <button id="btn-kml" @click="openFile('kml')">加载 KML/KMZ/GML（KML 结构）</button>
      <input ref="kmlInput" type="file" accept=".kml,.kmz,.gml,.xml" style="display:none" @change="onFileChange($event,'kml')"/>

      <button id="btn-czml" @click="openFile('czml')">加载 CZML</button>
      <input ref="czmlInput" type="file" accept=".czml,.json" style="display:none" @change="onFileChange($event,'czml')"/>

      <hr />

      <!-- 时间控制 -->
      <div class="time-controls">
        <button @click="play">播放</button>
        <button @click="pause">暂停</button>
        <button @click="faster">加速</button>
        <button @click="slower">减速</button>
        <button @click="jumpToStart">跳到开始</button>
        <button @click="jumpToEnd">跳到结束</button>
      </div>

      <hr />

      <!-- 简单轨迹加载示例（从JSON数组） -->
      <button @click="loadSampleTrajectory">加载示例轨迹（北京->上海）</button>

      <div class="info" v-if="selectedInfo">
        <h4>选中实体信息</h4>
        <pre>{{ selectedInfo }}</pre>
      </div>
    </div>
  </div>
</template>

<script setup>
/*
  说明：
  - 把 Cesium 的静态资源目录指向 window.CESIUM_BASE_URL（在项目中配置静态资源）
  - 确保在 index.html 或全局脚本里正确设置 Cesium 的 base url，或在本组件 mounted 时设置
*/

import { ref, onMounted, reactive } from 'vue'
import * as Cesium from 'cesium'
import 'cesium/Build/Cesium/Widgets/widgets.css'

const geojsonInput = ref(null)
const kmlInput = ref(null)
const czmlInput = ref(null)

let viewer = null
const selectedInfo = ref(null)

// time range for trajectory playback (Cesium.JulianDate)
let playbackStart = null
let playbackStop = null

onMounted(() => {
  // 如果需要，设置 Cesium 静态资源基路径
  window.CESIUM_BASE_URL = '/cesium' // 根据你的部署调整

  viewer = new Cesium.Viewer('cesiumContainer', {
    terrain: Cesium.Terrain.fromWorldTerrain(),
    animation: true, // 显示动画控件（可选）
    timeline: true,
    fullscreenButton: false,
    homeButton: false,
    sceneModePicker: false,
    baseLayerPicker: true,
    geocoder: false,
    navigationHelpButton: false,
    creditContainer: document.createElement('div')
  })

  // 点击实体显示属性
  const handler = new Cesium.ScreenSpaceEventHandler(viewer.scene.canvas)
  handler.setInputAction(function (click) {
    const picked = viewer.scene.pick(click.position)
    if (!Cesium.defined(picked) || !picked.id) {
      selectedInfo.value = null
      return
    }
    const ent = picked.id
    // 显示名称与自定义属性（如果有）
    const info = {
      id: ent.id || ent._name || 'entity',
      name: ent.name,
      properties: ent.properties ? ent.properties.getValue(Cesium.JulianDate.now()) : undefined
    }
    selectedInfo.value = JSON.stringify(info, null, 2)
  }, Cesium.ScreenSpaceEventType.LEFT_CLICK)
})

/* ---------- 文件选择控制 ---------- */
function openFile(type) {
  if (type === 'geojson') geojsonInput.value?.click()
  else if (type === 'kml') kmlInput.value?.click()
  else if (type === 'czml') czmlInput.value?.click()
}

async function onFileChange(e, type) {
  const file = e.target.files?.[0]
  if (!file) return
  // reset input so same file can be selected again later
  e.target.value = ''

  const text = await file.text()

  try {
    if (type === 'geojson') {
      // 支持对象或 URL — 这里用解析的对象
      const obj = JSON.parse(text)
      const ds = await Cesium.GeoJsonDataSource.load(obj, {
        clampToGround: true
      })
      viewer.dataSources.add(ds)
      viewer.zoomTo(ds)
    } else if (type === 'kml') {
      // Cesium 能加载 KML/KMZ (也能加载 KML 结构存于 .gml 如果它是 KML 结构)
      // 把文件变成 Blob 并传给 KmlDataSource.load
      const blob = new Blob([text], { type: 'application/vnd.google-earth.kml+xml' })
      const ds = await Cesium.KmlDataSource.load(blob, {
        camera: viewer.camera,
        clampToGround: true
      })
      viewer.dataSources.add(ds)
      viewer.zoomTo(ds)
    } else if (type === 'czml') {
      // CZML 可以是 JSON 文本或数组
      let czmlobj
      try { czmlobj = JSON.parse(text) } catch { czmlobj = text }
      const ds = await Cesium.CzmlDataSource.load(czmlobj)
      viewer.dataSources.add(ds)
      viewer.zoomTo(ds)
    }
  } catch (err) {
    console.error('加载失败', err)
    alert('加载失败: ' + (err.message || err))
  }
}

/* ---------- 示例轨迹加载与播放控制 ---------- */

/*
  loadSampleTrajectory:
  - 构造一个示例轨迹数组（包含ISO时间、lon,lat,alt）
  - 将其转换为 SampledPositionProperty 并创建实体（模型 + path）
  - 配置 viewer.clock 的 start/stop/currentTime/multiplier
*/
function loadSampleTrajectory() {
  if (!viewer) return

  // 生成示例时间点：每 30 分钟一个样点
  const startIso = '2025-11-09T00:00:00Z'
  const sampleCount = 5
  const secondsBetween = 30 * 60 // 30 min
  const startJD = Cesium.JulianDate.fromIso8601(startIso)

  const sampledPos = new Cesium.SampledPositionProperty()
  const trajectorySamples = []

  // 样例经纬（北京 >> 上海 等线性插值点）
  const lonLat = [
    [116.3974, 39.9092], // 北京
    [117.5, 39.5],
    [118.8, 38.5],
    [120.0, 36.5],
    [121.4737, 31.2304]   // 上海
  ]

  for (let i = 0; i < lonLat.length; i++) {
    const t = Cesium.JulianDate.addSeconds(startJD, i * secondsBetween, new Cesium.JulianDate())
    const [lon, lat] = lonLat[i]
    sampledPos.addSample(t, Cesium.Cartesian3.fromDegrees(lon, lat, 10000)) // 高度示例 10km
    trajectorySamples.push({ time: t, lon, lat, alt: 10000 })
  }

  // 设定播放时间范围
  playbackStart = trajectorySamples[0].time
  playbackStop = trajectorySamples[trajectorySamples.length - 1].time

  // 创建或移除同 id 的实体（便于多次加载）
  const existing = viewer.entities.getById('traj-plane')
  if (existing) viewer.entities.remove(existing)

  // 添加移动模型实体
  const entity = viewer.entities.add({
    id: 'traj-plane',
    name: '示例飞机',
    position: sampledPos,
    // 你可以换为飞机 glTF 模型（确保模型路径可访问）
    billboard: {
      image: undefined,
      show: false
    },
    model: {
      uri: undefined, // 若有 glTF: '/assets/plane.glb'
      scale: 2000,
      minimumPixelSize: 64
    },
    // path 显示轨迹，trailTime 控制历史轨迹长度（秒）
    path: {
      resolution: 1,
      material: new Cesium.PolylineGlowMaterialProperty({
        glowPower: 0.2,
        color: Cesium.Color.ORANGE
      }),
      width: 4,
      leadTime: 0,     // 前导时间（未来路径）可设置
      trailTime: 3600  // 历史路径显示长度（秒），设成足够覆盖
    }
  })

  // 设置 clock
  viewer.clock.startTime = playbackStart.clone()
  viewer.clock.stopTime = playbackStop.clone()
  viewer.clock.currentTime = playbackStart.clone()
  viewer.clock.multiplier = 60 // 时间倍率（1 秒模拟 60 秒）
  viewer.clock.shouldAnimate = false
  viewer.timeline.zoomTo(playbackStart, playbackStop)

  // 缩放到轨迹
  viewer.zoomTo(entity)
}

/* 播放/暂停/速度/跳转 */
function play() {
  if (!viewer) return
  viewer.clock.shouldAnimate = true
}
function pause() {
  if (!viewer) return
  viewer.clock.shouldAnimate = false
}
function faster() {
  if (!viewer) return
  viewer.clock.multiplier = Math.max(1, viewer.clock.multiplier * 2)
}
function slower() {
  if (!viewer) return
  viewer.clock.multiplier = Math.max(0.01, viewer.clock.multiplier / 2)
}
function jumpToStart() {
  if (!viewer || !playbackStart) return
  viewer.clock.currentTime = playbackStart.clone()
}
function jumpToEnd() {
  if (!viewer || !playbackStop) return
  viewer.clock.currentTime = playbackStop.clone()
}

/* 导出到模板可用 */
const api = {
  openFile,
  onFileChange,
  loadSampleTrajectory,
  play,
  pause,
  faster,
  slower,
  jumpToStart,
  jumpToEnd
}
</script>

<style scoped>
.cesium-container {
  width: 100%;
  height: 100vh;
  position: relative;
}
#controls {
  position: absolute;
  top: 10px;
  left: 10px;
  z-index: 999;
  display: flex;
  flex-direction: column;
  gap: 8px;
  background: rgba(255,255,255,0.9);
  padding: 8px;
  border-radius: 6px;
  width: 220px;
}
#controls button {
  padding: 6px 8px;
  font-size: 13px;
}
.time-controls {
  display:flex;
  flex-wrap:wrap;
  gap:6px;
}
.info {
  margin-top: 8px;
  font-size: 12px;
  max-height: 200px;
  overflow:auto;
}
</style>
