<template>
  <div id="canvas-wrapper">
    <canvas id="canvas" ref="canvas"
      @pointerdown="onmousedown"
      @pointermove="onmousemove"
      @pointerup="onmouseup"></canvas>
    <div v-show="showProvinceInfo" id="provinceInfo" ref="provinceInfo"
      :style="{'top': `${eventOffset.y}px`, 'left': `${eventOffset.x}px`}">
      {{hoveredProvince && hoveredProvince.userData.date}}<br>
      {{hoveredProvince && hoveredProvince.name}}<br>
      {{currentPollution}}: {{hoveredProvince && hoveredProvince.userData[currentPollution]}}
    </div>
    <div v-show="currentPollution" id="color-bar">
      <div v-for="(color, index) in levelColorArr" :key="index" class="color-bar-row">
        <span class="color-area" :style="{ backgroundColor: color }"></span>
        <span class="color-text">{{levelText(index)}}</span>
      </div>
    </div>
    <button id="reset" @click="onclickreset">Reset</button>
  </div>
</template>

<script>
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
import { FirstPersonControls } from 'three/examples/jsm/controls/FirstPersonControls.js';
import { GUI } from "three/examples/jsm/libs/dat.gui.module";
import * as d3 from 'd3';
import * as TWEEN from '@tweenjs/tween.js'

import chinajson from "assets/json/china.json";
import dateList2013 from "assets/json/date2013.json";
import provincedate2013 from "assets/json/provincedate2013.json";
import dateList2014 from "assets/json/date2014.json";
import provincedate2014 from "assets/json/provincedate2014.json";
import dateList2015 from "assets/json/date2015.json";
import provincedate2015 from "assets/json/provincedate2015.json";

export default {
  data() {
    return {
      canvas: null,
      provinceInfo: null,
      renderer: null,
      scene: new THREE.Scene(),
      camera: new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 5000),
      raycaster: new THREE.Raycaster(),
      map: new THREE.Object3D(),
      dateGroup: undefined,
      mouse: new THREE.Vector2(),
      eventOffset: {
        x: null,
        y: null
      },
      controls: null,
      normalDepth: 1,
      stretchedScale: 10,
      showProvinceInfo: true,
      hoveredProvince: null,
      selectedProvince: null,
      currentDate: null,
      currentPollution: null,
      currentYear: null,
      pollutionArr: ['PM2.5', 'PM10', 'SO2', 'NO2', 'CO', 'O3', 'U', 'V', 'TEMP', 'RH', 'PSFC'],
      yearArr: [
        {
          year: "2013",
          dateList: dateList2013,
          provincedate: provincedate2013
        },
        {
          year: "2014",
          dateList: dateList2014,
          provincedate: provincedate2014
        },
        {
          year: "2015",
          dateList: dateList2015,
          provincedate: provincedate2015
        }
      ],
      currentdateList: null,
      currentprovincedate: null,
      levelColorArr: ["#00cc00", "#66cc00", "#ffff00", "#ff9900", "#ff0000", "#a52a2a"],
      pollutionLevel: {
        "PM2.5": [30, 60, 90, 120, 250],
        "PM10": [50, 100, 250, 350, 430],
        "SO2": [40, 80, 380, 800, 1600],
        "NO2": [40, 80, 180, 280, 400],
        "CO": [1, 2, 10, 17, 34],
        "O3": [50, 100, 168, 208, 748],
      },
      blue: "#02A1E2",
      stretched: false
    }
  },
  computed: {
    levelText() {
      return index => {
        if (!this.currentPollution) return "";
        const levelList = this.pollutionLevel[this.currentPollution];
        if (index === 0) {
          return `0 - ${levelList[index]}`
        } else if (index === levelList.length) {
          return `${levelList[index - 1]} +`
        } else {
          return `${levelList[index - 1]} - ${levelList[index]}`
        }
      }
    },
    originIndex() {
      return this.currentdateList ? Math.floor(this.currentdateList.length / 2) : 0
    }
  },
  methods: {
    animate() {
      // this.controls.update(this.clock.getDelta());

      requestAnimationFrame(this.animate);
      // this.cube.rotation.x += 0.05;
      // this.cube.rotation.y += 0.05;

      // TWEEN.update();

      this.renderer.render(this.scene, this.camera);
    },
    initMap() {
      // 墨卡托投影转换
      const projection = d3.geoMercator().center([104.0, 37.5]).scale(80).translate([0, 0]);

      chinajson.features.forEach(geo => {
        // 定一个省份3D对象
        const province = new THREE.Object3D();
        province.name = geo.properties.name || "";
        province.userData.color = this.blue;

        // 每个的 坐标 数组
        const coordinates = geo.geometry.coordinates;

        // 循环坐标数组
        coordinates.forEach(multiPolygon => {
          multiPolygon.forEach(polygon => {
            const shape = new THREE.Shape();
            const lineMaterial = new THREE.LineBasicMaterial({
              color: 'white'
            });

            const points = [];

            for (let i = 0; i < polygon.length; i++) {
              const [x, y] = projection(polygon[i]);
              if (i === 0) {
                shape.moveTo(x, -y);
              }
              shape.lineTo(x, -y);
              points.push(new THREE.Vector3(x, -y, 0));
            }

            const lineGeometry = new THREE.BufferGeometry().setFromPoints(points);

            const extrudeSettings = {
              depth: this.normalDepth,
              bevelEnabled: false
            };

            const geometry = new THREE.ExtrudeGeometry(shape, extrudeSettings);
            const material = new THREE.MeshBasicMaterial({
              color: province.userData.color,
              transparent: true,
              opacity: 0.6
            });

            const mesh = new THREE.Mesh(geometry, material);
            const line = new THREE.Line(lineGeometry, lineMaterial);

            province.add(mesh);
            province.add(line);
          })
        })

        this.map.add(province);
      });
      this.scene.add(this.map);

      this.currentPollution = this.pollutionArr[0];
      this.currentYear = this.yearArr[0].year;
    },
    setLight() {
      this.scene.add(new THREE.AmbientLight(0xffffff)); // 环境光
    },
    setControls() {
      this.controls = new OrbitControls(this.camera, this.renderer.domElement);
      // this.controls = new FirstPersonControls(this.camera, this.renderer.domElement);

      // this.controls.lookSpeed = 0.1;
      // this.controls.movementSpeed = 100;

      // this.clock = new THREE.Clock();

      /* this.controls.enablePan = false; // 禁止右键拖拽
      this.controls.enableZoom = true; // false-禁止右键缩放
      
      this.controls.maxDistance = 200; // 最大缩放 适用于 PerspectiveCamera
      this.controls.minDistance = 50; // 最大缩放
      this.controls.enableRotate = true; // false-禁止旋转 */

      /* this.controls.minZoom = 0.5; // 最小缩放 适用于OrthographicCamera
      this.controls.maxZoom = 2; // 最大缩放 */
    },
    setGUI() {
      const gui = new GUI();
      const pollutionFolder = gui.addFolder('Pollution');
      pollutionFolder.add({ type: this.currentPollution }, 'type')
        .options(this.pollutionArr)
        .onChange(e => this.currentPollution = e);
      pollutionFolder.add({ year: this.currentYear }, 'year')
        .options(this.yearArr.map(e => e.year))
        .onChange(e => this.currentYear = e);
    },
    onmousedown(e) {
      this.moveMove = false;
    },
    onmousemove(e) {
      this.moveMove = true;
      // calculate mouse position in normalized device coordinates
      // (-1 to +1) for both components
      this.mouse.x = (e.clientX / window.innerWidth) * 2 - 1;
      this.mouse.y = -(e.clientY / window.innerHeight) * 2 + 1;
      this.eventOffset.x = e.clientX + 2;
      this.eventOffset.y = e.clientY + 2;

      this.raycaster.setFromCamera(this.mouse, this.camera);

      // calculate objects intersecting the picking ray
      const intersects = this.raycaster.intersectObjects(this.scene.children, true);
      if (this.hoveredProvince) { // 将上一次选中的恢复颜色
        this.hoveredProvince.children.forEach(obj => {
          if (obj instanceof THREE.Mesh) {
            let color;
            if (this.hoveredProvince.userData.gray) {
              color = 0xeeeeee;
            } else if (obj.scale.z !== 1) {
              color = this.blue;
            } else {
              color = this.hoveredProvince.userData.color;
            }
            obj.material.color.set(color);
          }
        });
      }

      const province = intersects.filter(obj => obj.object instanceof THREE.Mesh && obj.object.parent?.visible && obj.object.parent?.name)[0]?.object.parent;
      if (province) {
        province.children.forEach(e => {
          if (e instanceof THREE.Mesh) {
            e.material.color.set(0xff0000);
          }
        });
      }
      this.hoveredProvince = province;

      this.createProvinceInfo();
    },
    onmouseup(e) {
      if (this.moveMove) return;
      this.mouse.x = (e.clientX / window.innerWidth) * 2 - 1;
      this.mouse.y = -(e.clientY / window.innerHeight) * 2 + 1;
      this.raycaster.setFromCamera(this.mouse, this.camera);

      const lastProvince = this.selectedProvince;
      const intersects = this.raycaster.intersectObjects(this.scene.children, true);
      let province = intersects.filter(obj => obj.object instanceof THREE.Mesh && obj.object.parent?.visible)[0]?.object.parent;
      if (!province?.name || province.userData.gray) return;
      if (this.selectedProvince && province?.userData.date) {
        this.selectedProvince = null;
        this.currentDate = province?.userData.date;
      } else if (!this.selectedProvince) {
        if (province && province.name === lastProvince?.name) {
          province = null;
        }
        this.selectedProvince = province;
      }
    },
    setResize() {
      window.addEventListener('resize', () => {
        this.camera.aspect = window.innerWidth / window.innerHeight;
        this.camera.updateProjectionMatrix();
        this.renderer.setSize(window.innerWidth, window.innerHeight);
      })
    },
    createProvinceInfo() { // 显示省份的信息      
      if (this.hoveredProvince && this.hoveredProvince.visible) {
        if (!this.selectedProvince && this.hoveredProvince.name || this.selectedProvince && this.hoveredProvince.userData.date) {
          // this.provinceInfo.textContent = this.selectedProvince ? this.hoveredProvince.userData.date : this.hoveredProvince.name;
          this.showProvinceInfo = true;
        }
      } else {
        this.showProvinceInfo = false;
      }
    },
    updateMapColor(parent) {
      parent.children.forEach(province => {
        const pollutionLevel = this.pollutionLevel[this.currentPollution];
        if (pollutionLevel) {
          const num = province.userData[this.currentPollution]
          if (num != null) {
            let levelIndex = 0;
            while (num > pollutionLevel[levelIndex] && levelIndex < pollutionLevel.length) {
              levelIndex++;
            }
            province.userData.color = this.levelColorArr[levelIndex] || this.blue;
          } else {
            province.userData.color = this.blue;
          }
        } else {
          province.userData.color = this.blue;
        }
        province.children.forEach(obj => {
          if (obj instanceof THREE.Mesh) {
            let color;
            if (obj.scale.z !== 1) {
              color = this.blue;
            } else {
              color = province.userData.color;
            }
            obj.material.color.set(color);
          }
        });
      });
    },
    updateMapPos() {
      let currentDateIndex = this.currentdateList.findIndex(e => e === this.currentDate); 
      this.map.children.forEach(province => {
        province.visible = true;
        province.userData.gray = false;
        province.children.forEach(obj => {
          if (obj instanceof THREE.Mesh) {
            obj.scale.z = this.stretched ? this.currentdateList.length : 1;
            obj.position.z = this.stretched ? (0 - this.originIndex) * this.normalDepth : (currentDateIndex - this.originIndex) * this.normalDepth;
          } else if (obj instanceof THREE.Line) {
            obj.position.z = this.stretched ? (this.currentdateList.length - this.originIndex) * this.normalDepth : (currentDateIndex - this.originIndex + 1) * this.normalDepth;
          }
        });
      });
    },
    createDateGroup() {
      if (this.dateGroup) {
        this.disposeDateGroup();
      }

      const provinceData = this.currentprovincedate.find(e => e.name === this.selectedProvince.name);
      if (!provinceData) return;

      const dateGroup = new THREE.Object3D();

      this.currentdateList.forEach((date, i) => {
        const province = new THREE.Object3D();
        province.name = provinceData.name;
        const dateData = provinceData.data.find(e => e.date === date);
        if (dateData) {
          dateData.data.forEach((data, index) => {
            province.userData[this.pollutionArr[index]] = data;
          });
        }
        province.userData.date = date;
        province.userData.color = this.blue;

        const pollutionLevel = this.pollutionLevel[this.currentPollution];
        if (pollutionLevel) {
          const num = province.userData[this.currentPollution]
          if (num != null) {
            let levelIndex = 0;
            while (num > pollutionLevel[levelIndex] && levelIndex < pollutionLevel.length) {
              levelIndex++;
            }
            province.userData.color = this.levelColorArr[levelIndex] || this.blue;
          } else {
            province.userData.color = this.blue;
          }
        } else {
          province.userData.color = this.blue;
        }

        this.selectedProvince.children.forEach(object => {
          if (!object instanceof THREE.Mesh) return;
          const material = new THREE.MeshBasicMaterial({
            color: province.userData.color,
            transparent: true,
            opacity: 0.6
          });
          const mesh = new THREE.Mesh(object.geometry, material);
          mesh.position.z = (i - this.originIndex) * this.normalDepth;
          province.add(mesh);
        });
        dateGroup.add(province);
      });
      this.dateGroup = dateGroup;
      this.scene.add(this.dateGroup);
    },
    disposeDateGroup() {
      if (!this.dateGroup) return
      for (let i = 0; i < this.dateGroup.children.length; i++) {
        const province = this.dateGroup.children[i];
        for (let j = 0; j < province.children.length; j++) {
          const object = province.children[j];
          object.geometry.dispose();
          object.material.dispose();
        }
      }
      this.scene.remove(this.dateGroup);
      this.dateGroup = undefined;
    },
    onclickreset() {
      this.currentDate = this.currentdateList[0];
      this.stretched = true;
    }
  },
  mounted() {
    this.canvas = this.$refs.canvas;
    this.provinceInfo = this.$refs.provinceInfo;

    // 渲染器
    this.renderer = new THREE.WebGLRenderer({ canvas: this.canvas });
    this.renderer.setSize(window.innerWidth, window.innerHeight);
    // this.container.appendChild(this.renderer.domElement);

    // 场景
    this.scene.add(new THREE.AxesHelper(100));

    // 相机 透视相机
    this.camera.position.set(0, 0, 300);
    this.camera.lookAt(0, 0, 0);

    this.initMap();

    this.setLight(); // 设置灯光

    this.setControls(); // 设置控制

    this.setGUI();

    this.setResize(); // 绑定浏览器缩放事件

    this.animate();

    console.log(this.renderer.info)
    console.log(this.scene)
  },
  beforeDestroy() {
    this.scene.remove();
    this.renderer.dispose();
    this.renderer.forceContextLoss();
    this.renderer.content = null;
    this.renderer.domElement = null;
    cancelAnimationFrame(this.animate)
  },
  watch: {
    selectedProvince(val) {
      console.log(val?.name)
      const duration = 500;
      const easingFn = TWEEN.Easing.Quadratic.Out;
      const cameraTween = new TWEEN.Tween(this.camera.position)
        .easing(easingFn)
        .onUpdate(() => {
          this.controls.update();
        });
      if (val) {
        // this.map.children.forEach(e => e.visible = false);
        this.map.children.forEach(province => {
          if (val.name === province.name) {
            province.visible = false;
          } else {
            province.userData.gray = true;
          }
          province.children.forEach(obj => {
            if (obj instanceof THREE.Mesh) {
              obj.material.color.set(0xeeeeee);
            }
          });
        });
        cameraTween.to(new THREE.Vector3(-100, 0, 0), duration);

        this.createDateGroup();
      } else {
        // this.map.children.forEach(e => e.visible = true);
        if (this.stretched) {
          this.stretched = false;
        }
        this.map.children.forEach(province => {
          province.visible = true;
          province.userData.gray = false;
          province.children.forEach(obj => {
            if (obj instanceof THREE.Mesh) {
              obj.material.color.set(province.userData.color);
            }
          });
        });
        cameraTween.to(new THREE.Vector3(0, 0, 100), duration);

        this.disposeDateGroup();
      }
      cameraTween.start();
    },
    currentDate(val) {
      if (!val) return;
      this.map.children.forEach(province => {
        province.userData.date = val;
        const provinceData = this.currentprovincedate.find(e => e.name === province.name);
        if (!provinceData) return;
        const dateData = provinceData.data.find(e => e.date === province.userData.date)
        if (dateData) {
          dateData.data.forEach((data, index) => {
            province.userData[this.pollutionArr[index]] = data;
          });
        }
      });
      this.updateMapPos();
      this.updateMapColor(this.map);
      if (this.dateGroup) {
        this.disposeDateGroup();
        this.createDateGroup();
      }
    },
    currentPollution(val) {
      this.updateMapColor(this.map);
      if (this.dateGroup) {
        this.updateMapColor(this.dateGroup);
      }
    },
    currentYear(val) {
      const yearObj = this.yearArr.find(e => e.year === val);
      if (!yearObj) return;
      this.currentdateList = yearObj.dateList;
      this.currentprovincedate = yearObj.provincedate;

      this.stretched = true;

      let dateIndex = -1;
      if (this.currentDate) {
        dateIndex = this.currentdateList.findIndex(e => e.substr(4) === this.currentDate.substr(4));
      }
      this.currentDate = this.currentdateList[dateIndex] || this.currentdateList[0];
    },
    stretched(val) {
      this.updateMapPos();
      this.updateMapColor(this.map);

      if (val) {
        this.selectedProvince = null;
      }
    }
  }
}
</script>

<style lang="scss">
#canvas {
  width: 100%;
  height: 100%;
}

#provinceInfo {
  position: absolute;
  z-index: 2;
  background: white;
  padding: 6px;
  pointer-events: none;
  font-size: 12px;
  line-height: 1.5;
}

#color-bar {
  position: absolute;
  bottom: 10px;
  right: 15px;
  z-index: 2;
  background-color: black;
  padding: 6px;
  pointer-events: none;
  font-size: 12px;
  line-height: 1.5;
  border-radius: 2px;
  
  .color-bar-row {
    height: 20px;
    display: flex;
    flex-direction: row-reverse;
    justify-content: left;
    align-items: center;

    span {
      display: inline-block;
    }

    .color-text {
      color: #ffffff;
      margin-right: 10px;
    }

    .color-area {
      width: 30px;
      height: 14px;
    }
  }
}

#reset {
  position: absolute;
  bottom: 150px;
  right: 15px;
  z-index: 2;
}
</style>