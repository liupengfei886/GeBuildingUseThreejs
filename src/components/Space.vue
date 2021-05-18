<template>
  <div id="space">
  </div>
</template>

<script>
import * as THREE from 'three'
import { MapControls } from 'three/examples/jsm/controls/OrbitControls'
import Building from '@/assets/building.json'
import * as geolib from 'geolib';
import Stats from 'three/examples/jsm/libs/stats.module'
import { BufferGeometryUtils } from 'three/examples/jsm/utils/BufferGeometryUtils'

export default {
  name: 'Space',
  props: {
    msg: String
  },
  data() {
    return {
      scene: null,
      camera: null,
      renderer:null,
      controls: null,
      raycaster: null, // 光线投射（判断鼠标选中的建筑物）
      Geo_Building: [],
      Mat_Building: null,
      Mat_Road: null,
      Collider_building: [], // 碰撞器
      stats: null, // 调试组件
      center: [-3.189144, 55.9428942],
      FLAG_LINE_ANI: true, // 控制路线动画
      iR: null,
      iR_Line: null,
      Animated_Line_Distances: []
    }
  },
  mounted() {
    this.initScene()

    window.addEventListener('resize', onWindowResize, false)
    const that = this
    function onWindowResize() {
      that.camera.aspect = window.innerWidth / window.innerHeight
      that.camera.updateProjectionMatrix()
      that.renderer.setSize(window.innerWidth, window.innerHeight)
    }
    onWindowResize()
    document.getElementById('space').addEventListener('click', (event) => {
      const mouse = {
        x: ( event.clientX / window.innerWidth ) * 2 - 1,
        y: - ( event.clientY / window.innerHeight ) * 2 + 1
      }
      const hitted = that.Fire(mouse)
    })
    this.getGeoJSON()
  },
  methods: {
    initScene() {
      // 初始化场景scene
      this.scene = new THREE.Scene()
      this.scene.background = new THREE.Color(0x222222)

      // 初始化相机camera
      this.camera = new THREE.PerspectiveCamera(25, window.clientWidth/window.clientHeight, 1, 1000)
      this.camera.position.set(8, 4, 0)

      // 初始化光照
      const light = new THREE.AmbientLight(0xfafafa, 0.25)
      const light2 = new THREE.PointLight(0xfafafa, 0.4)
      light2.position.set(200, 90, 40)
      const light3 = new THREE.PointLight(0xfafafa, 0.4)
      light3.position.set(200, 90, -40)
      this.scene.add(light)
      this.scene.add(light2)
      this.scene.add(light3)

      const gh = new THREE.GridHelper(60, 160, new THREE.Color(0x555555), new THREE.Color(0x333333))
      this.scene.add(gh)

      // 初始化渲染器
      this.renderer = new THREE.WebGLRenderer({antialias: true})
      this.renderer.setPixelRatio(window.devicePixelRatio)
      this.renderer.setSize(window.innerWidth, window.innerHeight)
      const space = document.getElementById('space')
      space.appendChild(this.renderer.domElement)

      this.controls = new MapControls(this.camera, this.renderer.domElement)
      this.controls.enableDamping = true
      this.controls.dampingFactor = 0.25
      this.controls.screenSpacePanning = false
      this.controls.maxDistance = 800
      this.controls.update()
      
      // 添加调试组件
      this.stats = new Stats()
      space.appendChild(this.stats.domElement)

      // 初始化建筑以及公路的材质
      this.Mat_Building = new THREE.MeshPhongMaterial()
      this.Mat_Road = new THREE.LineBasicMaterial({color: 0x2F9BFF})
      this.raycaster = new THREE.Raycaster()

      // 初始化“组”
      this.iR = new THREE.Group()
      this.iR.name = '高速公路'
      this.iR_Line = new THREE.Group()

      this.$nextTick(() => {
        this.scene.add(this.iR)
        this.scene.add(this.iR_Line)
      })
      this.Update()
    },
    Update() {
      requestAnimationFrame(this.Update)
      this.renderer.render(this.scene, this.camera)
      this.controls.update()
      this.stats.update()
      this.UpdateAnimatedLine()
    },
    getGeoJSON() {
      for (let i = 0; i < Building.length; i++) {
        const ele = Building[i]
        if (!ele['properties']) return 
        const info = ele['properties']
        if (info.tags['building']) { 
          this.createBuilding(ele.geometry.coordinates, info, info.tags['building:levels'])
        } 
        // 创建高速路
        else if (info['hightway']) {
          if (ele.geometry.type === 'LineString' && info['highway'] !== 'pedestrian' && info['highway'] !== 'footway' && info['highway'] !== 'path') {
            this.createRoad(ele.geometry.coordinates)
          }
        }
      }
      // 将模型结合到一起后，整体添加到场景中（性能提升将近5倍）
      this.$nextTick(() => {
        const mergeGeometry = BufferGeometryUtils.mergeBufferGeometries(this.Geo_Building)
        const mesh = new THREE.Mesh(mergeGeometry, this.Mat_Building)
        this.scene.add(mesh)
      })
    },
    createBuilding(data, info, height = 1) {
      let shape = null
      const holes = []
      for (let i = 0; i < data.length; i++) {
        const ele = data[i]
        if (i === 0) {
          shape = this.genShape(ele)
        } else {
          holes.push(this.genShape(ele))
        }
      }
      // 添加镂空效果
      for (let j = 0; j < holes.length; j++) {
        shape.holes.push(holes[j])
      }
      const config = {
        curveSegments: 1,
        depth: 0.1 * height,
        bevelEnabled: false
      }
      const geometry = this.genGeometry(shape, config)
      geometry.rotateX(Math.PI / 2)
      geometry.rotateZ(Math.PI)
      // 将所有的模型结合到一起，优化性能
      this.Geo_Building.push(geometry)

      const helper = this.genHelper(geometry)
      if (helper) {
        helper.name = info.tags.name ? info.tags.name : '建筑物'
        helper.info = info
        this.Collider_building.push(helper)
      }
    },
    createRoad(data) {
      const points = []
      for (let i = 0; i < data.length; i++) {
        if (!data[0][1]) return
        const ele = data[i]
        if (!ele[0] || !ele[1]) return
        let elp = [ele[0], ele[1]]
        elp = this.GPSRelativePosition([ele[0], ele[1]], this.center)
        points.push(new THREE.Vector3(elp[0], 0.5, elp[1]))
      }
      let geometry = new THREE.BufferGeometry().setFromPoint(points)
      geometry.rotateZ(Math.PI)
      let line = new THREE.Line(geometry, this.Mat_Road)
      line.computeLineDistances()

      this.iR.add(line)
      line.position.y = 0.5

      if (this.FLAG_LINE_ANI) {
        const lineLength = geometry.attributes.lienDistance.array[geometry.attributes.lienDistance.count - 1]
        if (lineLength > 0.8) {
          const aniLine = this.addAnimatedLine(geometry, lineLength)
          this.iR_Line.add(aniLine)
        }
      }
    },
    // 添加路线动画效果(使用LineDeshedMaterial实现)
    addAnimatedLine(geometry, length) {
      const animatedLine = new THREE.Line(geometry, new THREE.LineDeshedMaterial({color: 0x00FFFF}))
      animatedLine.material.dashSize = 0
      animatedLine.material.gapSize = 1000
      animatedLine.position.y = 0.5
      animatedLine.material.transparent = true
      this.Animated_Line_Distances.push(length)

      return animatedLine
    },
    UpdateAnimatedLine() {
      if (!this.iR_Line.children.length) {
        return
      }
      for (let i = 0; i < this.iR_Line.children.length; i++) {
        const line = this.iR_Line.children[i]
        const dash = parseInt(line.material.dashSize)
        const length = parseInt(this.Animated_Line_Distances[i])

        if (dash > length) {
          // 初始化动画参数
          line.material.dashSize = 0
          line.material.opacity = 0
        } else {
          line.material.dashSize += 0.004
          line.material.opacity = line.material.opacity > 0 ? line.material.opacity - 0.002 : 0
        }
      }
    },
    /**
     * 使用Box3Helper为建筑创建包围盒（边框线）
     * 为了优化渲染性能，将模型整合为一个整体渲染后，如果想要实现单独点击某模型的场景可使用Box3Helper实现
     */
    genHelper(geometry) {
      if (!geometry.boundingBox) {
        geometry.computeBoundingBox()
      }
      const box3 = geometry.boundingBox
      if (!isFinite(box3.max.x)) {
        return false
      }
      const helper = new THREE.Box3Helper( box3, 0xffff00 )
      helper.updateMatrixWorld()
      return helper
    },
    // 创建二维的shape，用以拉伸成建筑
    genShape(points) {
      const shape = new THREE.Shape()
      for (let i = 0; i < points.length; i++) {
        let point = points[i]
        point = this.GPSRelativePosition(point, this.center)
        if (i === 0) {
          shape.moveTo(point[0], point[1])
        } else {
          shape.lineTo(point[0], point[1])
        }
      }
      return shape
    },
    // 使用ExtrudeBufferGeometry挤压缓冲几何体创建建筑模型
    genGeometry(shape, config) {
      const geometry = new THREE.ExtrudeBufferGeometry( shape, config )
      // 重新计算当前几何体对象的立方体界限
      // geometry.computeBoundingBox()
      // geometry.computeBoundingSphere()
      return geometry
    },
    /**
     * 地图是显示在平面上的，因此需要将球面坐标转换为平面坐标，这个转换过程称为投影。
     * 最常见的投影是墨卡托（Mercator）投影，它具有等角性质，即球体上的两点之间的角度方位与平面上的两点之间的角度方位保持不变，因此特别适合用于导航
     * 使用geolib将将球面坐标经度和纬度转换为平面坐标系中的x和y
     * @param objPosi 在地球上的经度和纬度
     * @param centerPosi 在二维地图上的中心点（绘制某块地图中的建筑先确认地图中心点）
     * @returns 二维坐标系中的x和y值
     */
    GPSRelativePosition(objPosi, centerPosi) {
      const dis = geolib.getDistance(objPosi, centerPosi)
      const bearing = geolib.getRhumbLineBearing(objPosi, centerPosi)
      const x = centerPosi[0] + (dis * Math.cos(bearing * Math.PI / 100))
      const y = centerPosi[1] + (dis * Math.sin(bearing * Math.PI / 100))
      return [-x / 100, y / 100]
    },
    Fire(mouse) {
      this.raycaster.setFromCamera(mouse, this.camera)
      const intersects = this.raycaster.intersectObjects(this.Collider_building, true)

      if (intersects && intersects.length) {
        return intersects[0].object
      } else {
        return false
      }
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
  #space {
    height: 100vh;
  }
</style>
