<template>
  <div id="app">
    <pollution-map></pollution-map>
  </div>
</template>

<script>
// import LineMap from "./assets/js/LineMap.js"
import PollutionMap from "@/PollutionMap.vue";

export default {
  name: 'App',
  components: {
    PollutionMap
  },
  methods:{
    
  },
  mounted(){
    // const line = new LineMap();
    // line.init();
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
</style>
