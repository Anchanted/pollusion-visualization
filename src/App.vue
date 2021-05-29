<template>
  <div id="app">
    <div id="provinceInfo"></div>
  </div>
</template>

<script>
import LineMap from "./assets/js/linemap.js"

export default {
  name: 'App',
  methods:{
    
  },
  mounted(){
    let line = new LineMap();
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
  margin-top: 60px;
}

html body {
  height: 100%;
  width: 100%;
  margin: 0;
  padding: 0;
  overflow: hidden;
}

#provinceInfo {
  position: absolute;
  z-index: 2;
  background: white;
  padding: 10px;
  visibility: hidden;
}
</style>
