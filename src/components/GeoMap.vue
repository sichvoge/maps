<template>
  <div class="geomap-container">
    <div v-if="loading" class="loading">Loading map data...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else class="map-wrapper">
      <div class="pulse-selector">
        <label for="pulse-mode">Animation Style:</label>
        <select id="pulse-mode" v-model="pulseMode" @change="changePulseMode">
          <option value="smooth-wave">Smooth Wave</option>
          <option value="rapid-pulse">Rapid Pulse</option>
          <option value="heartbeat">Heartbeat</option>
          <option value="gentle-fade">Gentle Fade</option>
          <option value="ripple-wave">Ripple Wave</option>
          <option value="static">Static (No Animation)</option>
        </select>
      </div>
      <div ref="mapContainer" class="map-container"></div>
      <div class="legend">
        <h3>Top Countries by Requests</h3>
        <div class="color-scale">
          <div class="scale-item">
            <span class="color-box" style="background: rgba(220, 38, 38, 0.8)"></span>
            <span class="scale-label">High</span>
          </div>
          <div class="scale-item">
            <span class="color-box" style="background: rgba(249, 115, 22, 0.8)"></span>
          </div>
          <div class="scale-item">
            <span class="color-box" style="background: rgba(251, 191, 36, 0.8)"></span>
          </div>
          <div class="scale-item">
            <span class="color-box" style="background: rgba(34, 197, 94, 0.8)"></span>
          </div>
          <div class="scale-item">
            <span class="color-box" style="background: rgba(59, 130, 246, 0.8)"></span>
            <span class="scale-label">Low</span>
          </div>
        </div>
        <div class="legend-divider"></div>
        <div v-for="item in topCountries" :key="item.country" class="legend-item">
          <span class="country-name">{{ item.name }}</span>
          <span class="country-requests">{{ formatNumber(item.requests) }}</span>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import maplibregl from 'maplibre-gl'
import 'maplibre-gl/dist/maplibre-gl.css'

export default {
  name: 'GeoMap',
  data() {
    return {
      map: null,
      loading: true,
      error: null,
      countryData: [],
      topCountries: [],
      pulseMode: 'smooth-wave',
      animationFrameId: null,
      currentPopup: null
    }
  },
  async mounted() {
    try {
      await this.loadData()
      this.loading = false
      await this.$nextTick()
      await this.initMap()
    } catch (err) {
      this.error = `Failed to load map: ${err.message}`
      this.loading = false
    }
  },
  beforeUnmount() {
    if (this.animationFrameId) {
      cancelAnimationFrame(this.animationFrameId)
    }
    if (this.map) {
      this.map.remove()
    }
  },
  methods: {
    async loadData() {
      const response = await fetch('/country-data.json')
      if (!response.ok) {
        throw new Error('Failed to load country data')
      }
      const data = await response.json()
      this.countryData = data.data
      
      this.topCountries = [...this.countryData]
        .sort((a, b) => b.requests - a.requests)
        .slice(0, 5)
    },
    async initMap() {
      // Initialize the map
      this.map = new maplibregl.Map({
        container: this.$refs.mapContainer,
        style: 'https://basemaps.cartocdn.com/gl/dark-matter-gl-style/style.json',
        center: [15, 30],
        zoom: 1.8
      })

      // Add navigation controls
      this.map.addControl(new maplibregl.NavigationControl(), 'top-right')

      // Wait for the map to load
      this.map.on('load', () => {
        console.log('Map loaded successfully')
        this.addCountryLayer()
      })

      this.map.on('error', (e) => {
        console.error('Map error:', e)
      })
    },
    async addCountryLayer() {
      // Calculate max value first
      const maxValue = Math.max(...this.countryData.map(d => d.requests))

      // Country coordinates (approximate centers)
      const countryCoordinates = {
        'US': [-95.7129, 37.0902],
        'DE': [10.4515, 51.1657],
        'GB': [-3.4360, 55.3781],
        'FR': [2.2137, 46.2276],
        'CA': [-106.3468, 56.1304],
        'JP': [138.2529, 36.2048],
        'AU': [133.7751, -25.2744],
        'BR': [-51.9253, -14.2350],
        'IN': [78.9629, 20.5937],
        'ES': [-3.7492, 40.4637],
        'IT': [12.5674, 41.8719],
        'NL': [5.2913, 52.1326],
        'SE': [18.6435, 60.1282],
        'CH': [8.2275, 46.8182],
        'MX': [-102.5528, 23.6345]
      }

      // Create GeoJSON features for each country
      const features = this.countryData
        .filter(item => countryCoordinates[item.country])
        .map(item => {
          const coords = countryCoordinates[item.country]
          return {
            type: 'Feature',
            geometry: {
              type: 'Point',
              coordinates: coords
            },
            properties: {
              country: item.country,
              name: item.name,
              requests: item.requests
            }
          }
        })

      console.log('Adding country bubbles:', features.length, 'countries')
      console.log('Max requests:', maxValue)
      console.log('Sample feature:', features[0])

      // Add source with point data
      this.map.addSource('country-points', {
        type: 'geojson',
        data: {
          type: 'FeatureCollection',
          features: features
        }
      })

      // Add outer transparent ring (largest)
      this.map.addLayer({
        id: 'country-bubbles-outer',
        type: 'circle',
        source: 'country-points',
        paint: {
          'circle-radius': [
            'interpolate',
            ['linear'],
            ['get', 'requests'],
            0, 20,
            maxValue * 0.3, 38,
            maxValue * 0.6, 55,
            maxValue, 75
          ],
          'circle-color': [
            'interpolate',
            ['linear'],
            ['/', ['get', 'requests'], maxValue],
            0, '#3b82f6',
            0.2, '#22c55e',
            0.4, '#facc15',
            0.6, '#f97316',
            0.8, '#dc2626'
          ],
          'circle-opacity': 0.15,
          'circle-blur': 0
        }
      })

      // Add pulsing middle ring
      this.map.addLayer({
        id: 'country-bubbles-pulse',
        type: 'circle',
        source: 'country-points',
        paint: {
          'circle-radius': [
            'interpolate',
            ['linear'],
            ['get', 'requests'],
            0, 15,
            maxValue * 0.3, 29,
            maxValue * 0.6, 42,
            maxValue, 60
          ],
          'circle-color': [
            'interpolate',
            ['linear'],
            ['/', ['get', 'requests'], maxValue],
            0, '#3b82f6',
            0.2, '#22c55e',
            0.4, '#facc15',
            0.6, '#f97316',
            0.8, '#dc2626'
          ],
          'circle-opacity': 0.35,
          'circle-blur': 0
        }
      })

      // Add solid center bubble
      this.map.addLayer({
        id: 'country-bubbles',
        type: 'circle',
        source: 'country-points',
        paint: {
          'circle-radius': [
            'interpolate',
            ['linear'],
            ['get', 'requests'],
            0, 10,
            maxValue * 0.3, 20,
            maxValue * 0.6, 30,
            maxValue, 45
          ],
          'circle-color': [
            'interpolate',
            ['linear'],
            ['/', ['get', 'requests'], maxValue],
            0, '#3b82f6',      // Blue - low
            0.2, '#22c55e',    // Green
            0.4, '#facc15',    // Yellow
            0.6, '#f97316',    // Orange
            0.8, '#dc2626'     // Red - high
          ],
          'circle-opacity': 1.0,
          'circle-stroke-width': 0,
          'circle-stroke-color': '#ffffff',
          'circle-stroke-opacity': 0
        }
      })

      console.log('Starting pulse animation')
      // Animate the pulse ring
      this.animatePulse()

      // Add hover effect
      this.map.on('mouseenter', 'country-bubbles', () => {
        this.map.getCanvas().style.cursor = 'pointer'
      })

      this.map.on('mousemove', 'country-bubbles', (e) => {
        // Remove existing popup
        if (this.currentPopup) {
          this.currentPopup.remove()
        }
        
        const feature = e.features[0]
        const { name, requests } = feature.properties
        
        // Create popup
        this.currentPopup = new maplibregl.Popup({
          closeButton: false,
          closeOnClick: false,
          offset: 15
        })
          .setLngLat(feature.geometry.coordinates)
          .setHTML(`
            <div style="padding: 12px; min-width: 160px; background: rgba(0, 0, 0, 0.9); border-radius: 6px;">
              <strong style="font-size: 15px; color: #fff; display: block; margin-bottom: 6px;">${name}</strong>
              <div style="display: flex; justify-content: space-between; align-items: center;">
                <span style="color: #aaa; font-size: 13px;">Requests:</span>
                <strong style="color: #22c55e; font-size: 15px; margin-left: 8px;">${this.formatNumber(requests)}</strong>
              </div>
            </div>
          `)
          .addTo(this.map)
      })

      this.map.on('mouseleave', 'country-bubbles', () => {
        this.map.getCanvas().style.cursor = ''
        if (this.currentPopup) {
          this.currentPopup.remove()
          this.currentPopup = null
        }
      })
    },
    formatNumber(num) {
      return new Intl.NumberFormat().format(num)
    },
    changePulseMode() {
      // Cancel existing animation
      if (this.animationFrameId) {
        cancelAnimationFrame(this.animationFrameId)
      }
      // Start new animation based on selected mode
      this.animatePulse()
    },
    animatePulse() {
      if (this.pulseMode === 'static') {
        // No animation, hide middle ring and show only center and smaller, thin outer ring
        if (this.map && this.map.getLayer('country-bubbles-pulse')) {
          this.map.setPaintProperty('country-bubbles-pulse', 'circle-opacity', 0)
        }
        if (this.map && this.map.getLayer('country-bubbles-outer')) {
          const maxValue = Math.max(...this.countryData.map(d => d.requests))
          // Smaller outer ring for static mode
          const smallerOuterRadius = [
            'interpolate',
            ['linear'],
            ['get', 'requests'],
            0, 15,
            maxValue * 0.3, 28,
            maxValue * 0.6, 40,
            maxValue, 55
          ]
          this.map.setPaintProperty('country-bubbles-outer', 'circle-radius', smallerOuterRadius)
          this.map.setPaintProperty('country-bubbles-outer', 'circle-opacity', 0.1)
        }
        return
      } else {
        // Restore outer ring opacity and radius for other modes
        if (this.map && this.map.getLayer('country-bubbles-outer')) {
          const maxValue = Math.max(...this.countryData.map(d => d.requests))
          const normalOuterRadius = [
            'interpolate',
            ['linear'],
            ['get', 'requests'],
            0, 20,
            maxValue * 0.3, 38,
            maxValue * 0.6, 55,
            maxValue, 75
          ]
          this.map.setPaintProperty('country-bubbles-outer', 'circle-radius', normalOuterRadius)
          this.map.setPaintProperty('country-bubbles-outer', 'circle-opacity', 0.15)
        }
      }

      let opacity = 0.35
      let growing = false
      let time = 0
      let rippleProgress = 0 // 0 to 1, for ripple wave
      let rippleDirection = 1 // 1 for outward, -1 for inward
      
      const pulse = () => {
        if (!this.map || !this.map.getLayer('country-bubbles-pulse')) {
          return
        }

        switch (this.pulseMode) {
          case 'smooth-wave':
            // Current implementation - smooth wave
            if (growing) {
              opacity += 0.01
              if (opacity >= 0.55) growing = false
            } else {
              opacity -= 0.01
              if (opacity <= 0.2) growing = true
            }
            this.map.setPaintProperty('country-bubbles-pulse', 'circle-opacity', opacity)
            break

          case 'rapid-pulse':
            // Faster pulsing
            if (growing) {
              opacity += 0.025
              if (opacity >= 0.6) growing = false
            } else {
              opacity -= 0.025
              if (opacity <= 0.15) growing = true
            }
            this.map.setPaintProperty('country-bubbles-pulse', 'circle-opacity', opacity)
            break

          case 'heartbeat':
            // Double pulse pattern like a heartbeat
            time += 0.05
            if (time < 0.3) {
              opacity = 0.2 + Math.sin(time * 10) * 0.25
            } else if (time < 0.6) {
              opacity = 0.2 + Math.sin((time - 0.3) * 10) * 0.25
            } else if (time < 1.5) {
              opacity = 0.2
            } else {
              time = 0
            }
            this.map.setPaintProperty('country-bubbles-pulse', 'circle-opacity', opacity)
            break

          case 'gentle-fade':
            // Very slow, gentle fade
            if (growing) {
              opacity += 0.005
              if (opacity >= 0.45) growing = false
            } else {
              opacity -= 0.005
              if (opacity <= 0.25) growing = true
            }
            this.map.setPaintProperty('country-bubbles-pulse', 'circle-opacity', opacity)
            break

          case 'ripple-wave':
            // Ripple wave - moves from center to outer, then restarts
            rippleProgress += 0.005
            
            if (rippleProgress >= 1) {
              rippleProgress = 0 // Reset to start from center again
            }

            // Get max value for scaling
            const maxValue = Math.max(...this.countryData.map(d => d.requests))
            
            // Calculate radius based on ripple progress
            // Progress 0 = at center bubble edge, Progress 1 = at outer ring
            const radiusExpression = [
              'interpolate',
              ['linear'],
              ['get', 'requests'],
              0, 10 + (rippleProgress * 10),  // Small bubbles: 10 to 20
              maxValue * 0.3, 20 + (rippleProgress * 18),  // Medium bubbles: 20 to 38
              maxValue * 0.6, 30 + (rippleProgress * 25),  // Large bubbles: 30 to 55
              maxValue, 45 + (rippleProgress * 30)  // Largest bubbles: 45 to 75
            ]
            
            // Opacity based on position (more visible in the middle of the journey)
            const opacityValue = 0.2 + Math.sin(rippleProgress * Math.PI) * 0.35
            
            this.map.setPaintProperty('country-bubbles-pulse', 'circle-radius', radiusExpression)
            this.map.setPaintProperty('country-bubbles-pulse', 'circle-opacity', opacityValue)
            break
        }

        this.animationFrameId = requestAnimationFrame(pulse)
      }

      pulse()
    }
  }
}
</script>

<style>
/* Global styles for MapLibre popups - not scoped */
.maplibregl-popup-content {
  background: transparent !important;
  padding: 0 !important;
  box-shadow: none !important;
  border: none !important;
}

.maplibregl-popup-tip {
  display: none !important;
}
</style>

<style scoped>
.geomap-container {
  width: 100%;
  max-height: 80vh;
  position: relative;
  overflow: hidden;
}

.loading,
.error {
  padding: 2rem;
  text-align: center;
  font-size: 1.2rem;
}

.error {
  color: #ef4444;
}

.map-wrapper {
  position: relative;
  width: 100%;
  height: 70vh;
}

.pulse-selector {
  position: absolute;
  top: 20px;
  left: 20px;
  z-index: 10;
  background: rgba(30, 30, 30, 0.95);
  padding: 12px 16px;
  border-radius: 8px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.3);
  backdrop-filter: blur(10px);
  display: flex;
  align-items: center;
  gap: 10px;
}

.pulse-selector label {
  color: #fff;
  font-size: 14px;
  font-weight: 500;
  white-space: nowrap;
}

.pulse-selector select {
  background: rgba(50, 50, 50, 0.9);
  color: #fff;
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 6px;
  padding: 8px 12px;
  font-size: 14px;
  cursor: pointer;
  outline: none;
  transition: all 0.2s;
}

.pulse-selector select:hover {
  border-color: rgba(255, 255, 255, 0.4);
  background: rgba(60, 60, 60, 0.9);
}

.pulse-selector select:focus {
  border-color: #3b82f6;
}

.pulse-selector select option {
  background: #2a2a2a;
  color: #fff;
  padding: 8px;
}

.map-container {
  width: 100%;
  height: 100%;
  border-radius: 8px;
  overflow: hidden;
}

.legend {
  position: absolute;
  top: 20px;
  right: 20px;
  min-width: 250px;
  max-width: 280px;
  padding: 1.2rem;
  background: rgba(30, 30, 30, 0.95);
  border-radius: 8px;
  text-align: left;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.3);
  backdrop-filter: blur(10px);
}

.legend h3 {
  font-size: 1.1rem;
  margin-bottom: 1rem;
  color: #fff;
}

.color-scale {
  display: flex;
  flex-direction: column;
  gap: 4px;
  margin-bottom: 0.5rem;
}

.scale-item {
  display: flex;
  align-items: center;
  gap: 8px;
}

.color-box {
  width: 40px;
  height: 16px;
  border-radius: 2px;
  border: 1px solid rgba(255, 255, 255, 0.2);
}

.scale-label {
  font-size: 0.85rem;
  color: #aaa;
}

.legend-divider {
  height: 1px;
  background: rgba(255, 255, 255, 0.2);
  margin: 1rem 0;
}

.legend-item {
  display: flex;
  justify-content: space-between;
  padding: 0.6rem 0;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
  gap: 1rem;
}

.legend-item:last-child {
  border-bottom: none;
}

.country-name {
  font-weight: 500;
  font-size: 0.95rem;
  color: #fff;
}

.country-requests {
  color: #888;
  font-weight: 600;
  font-size: 0.95rem;
  white-space: nowrap;
}

@media (max-width: 1024px) {
  .map-wrapper {
    height: 60vh;
  }
  
  .legend {
    top: 10px;
    right: 10px;
    min-width: 220px;
    padding: 1rem;
  }
}

@media (max-width: 768px) {
  .map-wrapper {
    height: 500px;
  }
  
  .legend {
    position: static;
    margin-top: 1rem;
    width: 100%;
    max-width: 100%;
  }
}
</style>
