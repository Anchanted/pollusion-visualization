<template>
  <div id="app">
    <canvas id="canvas"></canvas>
    <div id="provinceInfo"></div>
  </div>
</template>

<script>
import LineMap from "./assets/js/LineMap.js"

export default {
  name: 'App',
  methods:{
    
  },
  mounted(){
    const line = new LineMap();
    line.init();
  }
}
</script>

<style lang="scss">
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  width: 100%;
  height: 100%;
}

body {
  height: 100%;
  width: 100%;
  margin: 0;
  padding: 0;
  overflow: hidden;
}

#canvas {
  width: 100%;
  height: 100%;
}

#provinceInfo {
  position: absolute;
  z-index: 2;
  background: white;
  padding: 10px;
  visibility: hidden;
}
</style>
