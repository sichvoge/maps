<template>
  <div class="app-container">
    <h1>Request Analytics by Country</h1>
    <p class="subtitle">Interactive geographic visualization of request distribution</p>
    <GeoMap />
  </div>
</template>

<script>
import GeoMap from './components/GeoMap.vue'

export default {
  name: 'App',
  components: {
    GeoMap
  }
}
</script>

<style scoped>
.app-container {
  width: 100%;
}

.subtitle {
  color: #888;
  margin-bottom: 2rem;
  font-size: 1.1rem;
}
</style>
