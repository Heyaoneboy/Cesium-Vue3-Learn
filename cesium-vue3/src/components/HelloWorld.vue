<script setup>
import {onMounted} from 'vue'
import * as Cesium from 'cesium'
import 'cesium/Build/Cesium/Widgets/widgets.css'

onMounted(() => {
  window.CESIUM_BASE_URL = '/cesium'

  const viewer = new Cesium.Viewer('cesiumContainer', {
    terrain: Cesium.Terrain.fromWorldTerrain(),// 新写法

    //关闭界面初始元素
    animation: false, //动画控件
    timeline: false, //时间线
    fullscreenButton: false,//全屏按钮
    homeButton: false,//主页按钮
    sceneModePicker: false,//2d 3d切换
    baseLayerPicker: false,//地图选择
    geocoder: false,//搜索框
    navigationHelpButton: false,//帮助按钮
    //底部版权信息
    creditContainer: document.createElement('div')

  })

  // 添加一个点（北京）
  viewer.entities.add({
    name: "北京",
    position: Cesium.Cartesian3.fromDegrees(116.3974, 39.9092),
    point: {
      pixelSize: 12,
      color: Cesium.Color.RED,
      outlineColor: Cesium.Color.WHITE,
      outlineWidth: 2
    },
    label: {
      text: "北京",
      font: "14pt 宋体",
      style: Cesium.LabelStyle.FILL_AND_OUTLINE,
      outlineWidth: 2,
      verticalOrigin: Cesium.VerticalOrigin.BOTTOM,
      pixelOffset: new Cesium.Cartesian2(0, -12)
    }
  });

  //添加第二个点 上海
  viewer.entities.add({
    name: '上海',
    position: Cesium.Cartesian3.fromDegrees(121.4737, 31.2304),
    point: {
      pixelSize: 12,
      color: Cesium.Color.RED,
      outlineColor: Cesium.Color.WHITE,
      outlineWidth: 2,
    },
    label: {
      text: '上海',
      font: "14pt 宋体",
      style: Cesium.LabelStyle.FILL_AND_OUTLINE,
      outlineWidth: 2,
      verticalOrigin: Cesium.VerticalOrigin.BOTTOM,
      pixelOffset: new Cesium.Cartesian2(0, -12)
    }
  });

/*  //广告牌
  viewer.entities.add({
    name: '广告牌',
    position: Cesium.Cartesian3.fromDegrees(121.49491, 31.24169),
    billboard: {
      image: '/img/东方明珠.svg',
      width: 64,
      height: 64,
      sizeInMeters: false,
      scale: 1.0
    },
    label: {
      text: '东方明珠',
      font: '14px sans-serif',
      style: Cesium.LabelStyle.FILL_AND_OUTLINE,
      outlineWidth: 2,
      verticalOrigin: Cesium.VerticalOrigin.BOTTOM,
      pixelOffset: new Cesium.Cartesian2(0, -30)

    }
  })*/

  //连接线
  viewer.entities.add({
    name: '北京-上海',
    polyline: {
      positions: Cesium.Cartesian3.fromDegreesArray([
        116.3974, 39.9092,   // 北京
        121.4737, 31.2304    // 上海
      ]),
      width: 3,
      material: Cesium.Color.BLUE
    }

  });

  //多边形
  viewer.entities.add({
    name: '多边形',
    polygon: {
      hierarchy: Cesium.Cartesian3.fromDegreesArray([
        116.4, 40,
        117.5, 40,
        117.5, 39.5,
        116.4, 39.5
      ]),
      material: Cesium.Color.BLUE.withAlpha(0.4),
      outline: true,
      outlineColor: Cesium.Color.WHITE
    }
  })

/*  //圆形
  viewer.entities.add({
    name: '圆形',
    position: Cesium.Cartesian3.fromDegrees(116.0, 39.9092,),
    ellipse: {
      semiMajorAxis: 30000,//长半径
      semiMinorAxis: 30000,//短半径
      material: new Cesium.ImageMaterialProperty({
        image: '/img/纹理.png',
        transparent: false,//支持透明
        repeat: new Cesium.Cartesian2(128, 128)//重复次数
      })
      //材质
    }
  })*/

/*  //加载三维模型
  viewer.entities.add({
    name: '雪人',
    position: Cesium.Cartesian3.fromDegrees(116.3974, 39.9192, 0),
    model: {
      uri: '/img/snowman.glb',  //不能使用url
      scale: 12.0,
      minimumPixelSize: 640,      // 模型太远时最小显示尺寸
      maximumScale: 2000          // 限制最大缩放倍数
    }
  })*/

  viewer.camera.flyTo({
    destination: Cesium.Cartesian3.fromDegrees(116.3974, 39.9092, 30000.0),
    orientation: {
      headinh: Cesium.Math.toRadians(0.0),
      pitch: Cesium.Math.toRadians(-90.0),
      roll: 0.0
    },
    duration: 2
  })

  //视角切换
  const btn = document.querySelector('#change-btn button')
  let isBeiJing = false
  btn.addEventListener('click', () => {

    const BeiJing = {
      lon: 116.3974,
      lat: 39.9092
    }
    const Shanghai = {
      lon: 121.4737,
      lat: 31.2304
    }

    const target = isBeiJing ? BeiJing : Shanghai
    viewer.camera.flyTo({
      destination: Cesium.Cartesian3.fromDegrees(target.lon, target.lat, 30000.0),
      orientation: {
        headinh: Cesium.Math.toRadians(0.0),
        pitch: Cesium.Math.toRadians(-90.0),
        roll: 0.0
      },
      duration: 2
    })
    isBeiJing = !isBeiJing

  })
  //viewer.flyTo(viewer.entities)
})
</script>

<template>
  <div id="cesiumContainer" class="cesium-container"></div>
  <div id="change-btn">
    <button>切换</button>
  </div>
</template>

<style scoped>
.cesium-container {
  width: 100%;
  height: 100vh;
  position: relative;
}

#change-btn {
  position: absolute;
  top: 20px;
  left: 20px;
  z-index: 10;
}
</style>
