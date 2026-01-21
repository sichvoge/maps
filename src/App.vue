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
  position: relative;
}

.app-container::before {
  content: '';
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: 
    radial-gradient(circle at 20% 50%, rgba(59, 130, 246, 0.03) 0%, transparent 50%),
    radial-gradient(circle at 80% 80%, rgba(34, 197, 94, 0.03) 0%, transparent 50%),
    radial-gradient(circle at 50% 20%, rgba(249, 115, 22, 0.02) 0%, transparent 50%);
  pointer-events: none;
  z-index: 0;
}

.subtitle {
  color: #888;
  margin-bottom: 2rem;
  font-size: 1.1rem;
  position: relative;
  z-index: 1;
}

h1 {
  position: relative;
  z-index: 1;
}
</style>
