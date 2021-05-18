<template>
  <div id="app">
    <Space />
  </div>
</template>

<script>
import Space from './components/Space.vue'

export default {
  name: 'app',
  components: {
    Space
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}
body { 
  margin: 0;
  overflow: hidden;
}
</style>
