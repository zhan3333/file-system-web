<template>
  <div id="app">
    <home></home>
  </div>
</template>

<script>
import home from './pages/home.vue'

export default {
  name: 'app',
  components: {
    home
  }
}
</script>

<style>
#app {
  font-family: "Helvetica Neue",Helvetica,"PingFang SC","Hiragino Sans GB","Microsoft YaHei","微软雅黑",Arial,sans-serif;
}
</style>
