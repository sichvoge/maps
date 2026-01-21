<template>
  <div class="geomap-container">
    <div v-if="loading" class="loading">Loading map data...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="controls-bar">
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
      </div>
      <div class="map-wrapper">
        <div class="stats-card">
          <div class="stats-label">TOTAL REQUESTS</div>
          <div class="stats-value">{{ formatNumber(totalRequests) }}</div>
          <div class="stats-change">
            <span class="change-icon">↑</span>
            <span class="change-value">+12.5%</span>
            <span class="change-label">vs last hour</span>
          </div>
        </div>
        <div class="legend-card" :class="{ collapsed: legendCollapsed }">
          <div class="legend-header" @click="toggleLegend">
            <div class="legend-title">
              <span class="legend-icon">📊</span>
              <span>Top 5 Countries</span>
            </div>
            <span class="toggle-icon">{{ legendCollapsed ? '▼' : '▲' }}</span>
          </div>
          <div v-if="!legendCollapsed" class="legend-content">
            <div v-for="(item, index) in topCountries" :key="item.country" class="legend-country">
              <div class="country-rank">{{ index + 1 }}</div>
              <div class="country-flag">{{ getCountryFlag(item.country) }}</div>
              <div class="country-info">
                <div class="country-header">
                  <span class="country-name">{{ item.name }}</span>
                  <span class="country-percent">{{ getPercentage(item.requests) }}%</span>
                </div>
                <div class="country-stats">
                  <span class="country-requests">{{ formatNumber(item.requests) }}</span>
                  <div class="country-sparkline">
                    <svg width="40" height="16" viewBox="0 0 40 16">
                      <polyline
                        :points="generateSparkline(index)"
                        fill="none"
                        :stroke="getSparklineColor(item.requests)"
                        stroke-width="1.5"
                      />
                    </svg>
                  </div>
                </div>
              </div>
              <div class="country-bar">
                <div 
                  class="country-bar-fill" 
                  :style="{ width: getPercentage(item.requests) + '%', background: getBarColor(item.requests) }"
                ></div>
              </div>
            </div>
          </div>
        </div>
        <div ref="mapContainer" class="map-container"></div>
        
        <transition name="slide">
          <div v-if="showDetailPanel && selectedCountry" class="detail-panel">
            <div class="detail-header">
              <div class="detail-title">
                <span class="detail-flag">{{ getCountryFlag(selectedCountry.country) }}</span>
                <div>
                  <h3>{{ selectedCountry.name }}</h3>
                  <span class="detail-code">{{ selectedCountry.country }}</span>
                </div>
              </div>
              <button class="close-btn" @click="closeDetailPanel">✕</button>
            </div>
            
            <div class="detail-content">
              <div class="detail-stat-card">
                <div class="stat-label">Total Requests</div>
                <div class="stat-value">{{ formatNumber(selectedCountry.requests) }}</div>
                <div class="stat-trend" :style="{ color: countryTrends[selectedCountry.country]?.direction === 'up' ? '#22c55e' : '#ef4444' }">
                  {{ countryTrends[selectedCountry.country]?.direction === 'up' ? '↑' : '↓' }}
                  {{ countryTrends[selectedCountry.country]?.percentage }}% vs yesterday
                </div>
              </div>
              
              <div class="detail-stat-card">
                <div class="stat-label">% of Total Traffic</div>
                <div class="stat-value">{{ getPercentage(selectedCountry.requests) }}%</div>
                <div class="progress-bar-full">
                  <div class="progress-fill" :style="{ width: getPercentage(selectedCountry.requests) + '%' }"></div>
                </div>
              </div>
              
              <div class="detail-stat-card">
                <div class="stat-label">Rank</div>
                <div class="stat-value">#{{ getCountryRank(selectedCountry.country) }}</div>
                <div class="stat-info">out of {{ countryData.length }} countries</div>
              </div>
              
              <div class="chart-section">
                <h4>Request History (7 days)</h4>
                <svg class="request-chart" viewBox="0 0 280 70" preserveAspectRatio="none">
                  <defs>
                    <linearGradient id="chartGradient" x1="0%" y1="0%" x2="0%" y2="100%">
                      <stop offset="0%" style="stop-color:#3b82f6;stop-opacity:0.3" />
                      <stop offset="100%" style="stop-color:#3b82f6;stop-opacity:0" />
                    </linearGradient>
                  </defs>
                  <path :d="generateChartPath(selectedCountry.country)" fill="url(#chartGradient)" />
                  <polyline :points="generateChartLine(selectedCountry.country)" fill="none" stroke="#3b82f6" stroke-width="2" />
                </svg>
              </div>
              
              <div class="detail-actions">
                <button class="action-btn primary" @click="zoomToCountry(selectedCountry)">
                  🔍 Zoom to Country
                </button>
                <button class="action-btn" @click="resetZoom">
                  🌍 Reset View
                </button>
              </div>
            </div>
          </div>
        </transition>
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
      currentPopup: null,
      totalRequests: 0,
      legendCollapsed: true,
      countryTrends: {},
      selectedCountry: null,
      showDetailPanel: false
    }
  },
  computed: {
    formattedTotal() {
      return this.formatNumber(this.totalRequests)
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
      
      // Calculate total requests
      this.totalRequests = this.countryData.reduce((sum, item) => sum + item.requests, 0)
      
      // Generate consistent trend data for each country
      this.countryData.forEach(item => {
        this.countryTrends[item.country] = {
          direction: Math.random() > 0.5 ? 'up' : 'down',
          percentage: (Math.random() * 20 + 5).toFixed(1)
        }
      })
      
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
      
      // Hide MapLibre attribution
      const attributionControl = this.map._controls.find(c => c instanceof maplibregl.AttributionControl)
      if (attributionControl) {
        this.map.removeControl(attributionControl)
      }

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
        const { name, requests, country } = feature.properties
        const percentage = this.getPercentage(requests)
        const flag = this.getCountryFlag(country)
        
        // Get consistent trend data for this country
        const trendData = this.countryTrends[country] || { direction: 'up', percentage: '0.0' }
        const trend = trendData.direction === 'up' ? '↑' : '↓'
        const trendColor = trendData.direction === 'up' ? '#22c55e' : '#ef4444'
        const trendPercent = trendData.percentage
        
        // Create popup
        this.currentPopup = new maplibregl.Popup({
          closeButton: false,
          closeOnClick: false,
          offset: 15
        })
          .setLngLat(feature.geometry.coordinates)
          .setHTML(`
            <div style="padding: 14px 16px; min-width: 200px; background: rgba(15, 15, 20, 0.98); border-radius: 8px; border: 1px solid rgba(255, 255, 255, 0.1); box-shadow: 0 8px 16px rgba(0, 0, 0, 0.5);">
              <div style="display: flex; align-items: center; gap: 10px; margin-bottom: 10px;">
                <span style="font-size: 28px; line-height: 1;">${flag}</span>
                <div style="flex: 1;">
                  <strong style="font-size: 16px; color: #fff; display: block; line-height: 1.2;">${name}</strong>
                  <span style="font-size: 11px; color: #666; text-transform: uppercase; letter-spacing: 0.5px;">${country}</span>
                </div>
              </div>
              <div style="border-top: 1px solid rgba(255, 255, 255, 0.1); padding-top: 10px; display: flex; flex-direction: column; gap: 8px;">
                <div style="display: flex; justify-content: space-between; align-items: center;">
                  <span style="color: #888; font-size: 12px;">Total Requests</span>
                  <strong style="color: #fff; font-size: 16px; font-weight: 600;">${this.formatNumber(requests)}</strong>
                </div>
                <div style="display: flex; justify-content: space-between; align-items: center;">
                  <span style="color: #888; font-size: 12px;">% of Total</span>
                  <strong style="color: #22c55e; font-size: 14px; font-weight: 600;">${percentage}%</strong>
                </div>
                <div style="display: flex; justify-content: space-between; align-items: center;">
                  <span style="color: #888; font-size: 12px;">Trend (24h)</span>
                  <span style="color: ${trendColor}; font-size: 14px; font-weight: 600;">
                    <span style="font-size: 16px;">${trend}</span> ${trendPercent}%
                  </span>
                </div>
                <div style="margin-top: 6px; padding-top: 8px; border-top: 1px solid rgba(255, 255, 255, 0.05);">
                  <span style="color: #555; font-size: 10px;">Last updated: just now</span>
                </div>
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

      // Add click handler for bubbles
      this.map.on('click', 'country-bubbles', (e) => {
        const feature = e.features[0]
        const { name, requests, country } = feature.properties
        
        this.selectedCountry = { name, requests, country }
        this.showDetailPanel = true
        
        // Optional: zoom to country on click
        this.zoomToCountry({ name, requests, country })
      })
    },
    formatNumber(num) {
      return new Intl.NumberFormat().format(num)
    },
    toggleLegend() {
      this.legendCollapsed = !this.legendCollapsed
    },
    getCountryFlag(countryCode) {
      const flags = {
        'US': '🇺🇸',
        'DE': '🇩🇪',
        'GB': '🇬🇧',
        'FR': '🇫🇷',
        'CA': '🇨🇦',
        'JP': '🇯🇵',
        'AU': '🇦🇺',
        'BR': '🇧🇷',
        'IN': '🇮🇳',
        'ES': '🇪🇸',
        'IT': '🇮🇹',
        'NL': '🇳🇱',
        'SE': '🇸🇪',
        'CH': '🇨🇭',
        'MX': '🇲🇽'
      }
      return flags[countryCode] || '🌍'
    },
    getPercentage(requests) {
      return ((requests / this.totalRequests) * 100).toFixed(1)
    },
    generateSparkline(index) {
      // Generate fake sparkline data for demo (simulating trend over 7 days)
      const points = []
      const variance = [0.8, 0.9, 1.1, 0.95, 1.05, 1.15, 1.0]
      
      for (let i = 0; i < 7; i++) {
        const x = (i / 6) * 40
        const y = 16 - (variance[i] * 8 + Math.random() * 4)
        points.push(`${x},${y}`)
      }
      
      return points.join(' ')
    },
    getSparklineColor(requests) {
      const maxValue = Math.max(...this.countryData.map(d => d.requests))
      const ratio = requests / maxValue
      
      if (ratio >= 0.7) return '#dc2626'
      if (ratio >= 0.5) return '#f97316'
      if (ratio >= 0.3) return '#facc15'
      if (ratio >= 0.1) return '#22c55e'
      return '#3b82f6'
    },
    getBarColor(requests) {
      const maxValue = Math.max(...this.countryData.map(d => d.requests))
      const ratio = requests / maxValue
      
      if (ratio >= 0.7) return 'linear-gradient(90deg, #dc2626, #ef4444)'
      if (ratio >= 0.5) return 'linear-gradient(90deg, #f97316, #fb923c)'
      if (ratio >= 0.3) return 'linear-gradient(90deg, #facc15, #fde047)'
      if (ratio >= 0.1) return 'linear-gradient(90deg, #22c55e, #4ade80)'
      return 'linear-gradient(90deg, #3b82f6, #60a5fa)'
    },
    closeDetailPanel() {
      this.showDetailPanel = false
      this.selectedCountry = null
    },
    getCountryRank(countryCode) {
      const sorted = [...this.countryData].sort((a, b) => b.requests - a.requests)
      return sorted.findIndex(c => c.country === countryCode) + 1
    },
    zoomToCountry(country) {
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
      
      const coords = countryCoordinates[country.country]
      if (coords && this.map) {
        this.map.flyTo({
          center: coords,
          zoom: 4.5,
          duration: 2000
        })
      }
    },
    resetZoom() {
      if (this.map) {
        this.map.flyTo({
          center: [15, 30],
          zoom: 1.8,
          duration: 2000
        })
      }
      this.closeDetailPanel()
    },
    generateChartLine(countryCode) {
      // Generate fake 7-day data
      const points = []
      const baseValue = this.countryData.find(c => c.country === countryCode)?.requests || 1000
      
      for (let i = 0; i < 7; i++) {
        const x = (i / 6) * 280
        const variance = 0.8 + Math.random() * 0.4
        const y = 70 - ((baseValue * variance) / (baseValue * 1.2)) * 56
        points.push(`${x},${y}`)
      }
      
      return points.join(' ')
    },
    generateChartPath(countryCode) {
      const line = this.generateChartLine(countryCode)
      return `M 0,70 L ${line} L 280,70 Z`
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
/* Global styles for MapLibre popups and attribution - not scoped */
.maplibregl-popup-content {
  background: transparent !important;
  padding: 0 !important;
  box-shadow: none !important;
  border: none !important;
}

.maplibregl-popup-tip {
  display: none !important;
}

.maplibregl-ctrl-attrib,
.maplibregl-ctrl-bottom-left,
.maplibregl-ctrl-bottom-right {
  display: none !important;
}
</style>

<style scoped>
.geomap-container {
  width: 100%;
  position: relative;
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

.controls-bar {
  display: flex;
  justify-content: flex-start;
  padding: 1rem 0;
  margin-bottom: 1rem;
}

.pulse-selector {
  background: rgba(30, 30, 30, 0.95);
  padding: 12px 16px;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
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

.map-wrapper {
  position: relative;
  width: 100%;
  height: 70vh;
}

.stats-card {
  position: absolute;
  top: 20px;
  left: 20px;
  z-index: 10;
  background: rgba(20, 20, 25, 0.95);
  padding: 16px 20px;
  border-radius: 10px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.4);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.1);
  min-width: 200px;
}

.stats-label {
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 0.5px;
  color: #888;
  text-transform: uppercase;
  margin-bottom: 8px;
}

.stats-value {
  font-size: 32px;
  font-weight: 700;
  color: #fff;
  line-height: 1;
  margin-bottom: 8px;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
}

.stats-change {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 13px;
}

.change-icon {
  color: #22c55e;
  font-size: 14px;
  font-weight: bold;
}

.change-value {
  color: #22c55e;
  font-weight: 600;
}

.change-label {
  color: #666;
  margin-left: 2px;
}

.map-container {
  width: 100%;
  height: 100%;
  border-radius: 8px;
  overflow: hidden;
  position: relative;
  background: linear-gradient(135deg, #1a1d29 0%, #0f1419 50%, #1a1d29 100%);
}

.map-container::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-image: 
    linear-gradient(rgba(255, 255, 255, 0.02) 1px, transparent 1px),
    linear-gradient(90deg, rgba(255, 255, 255, 0.02) 1px, transparent 1px);
  background-size: 50px 50px;
  pointer-events: none;
  z-index: 1;
}

.map-container canvas {
  position: relative;
  z-index: 2;
}

.legend-card {
  position: absolute;
  top: 20px;
  right: 50px;
  z-index: 10;
  background: rgba(20, 20, 25, 0.95);
  border-radius: 10px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.4);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.1);
  min-width: 300px;
  max-width: 320px;
  transition: all 0.3s ease;
  display: flex;
  flex-direction: column;
}

.legend-card.collapsed {
  min-width: 180px;
}

.legend-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 14px 16px;
  cursor: pointer;
  user-select: none;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.legend-card.collapsed .legend-header {
  border-bottom: none;
}

.legend-header:hover {
  background: rgba(255, 255, 255, 0.03);
}

.legend-title {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 14px;
  font-weight: 600;
  color: #fff;
}

.legend-icon {
  font-size: 16px;
}

.toggle-icon {
  color: #888;
  font-size: 12px;
  transition: transform 0.3s;
}

.legend-content {
  padding: 12px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.legend-country {
  display: grid;
  grid-template-columns: 24px 32px 1fr;
  grid-template-rows: auto auto;
  gap: 8px;
  align-items: center;
  padding: 10px;
  background: rgba(255, 255, 255, 0.03);
  border-radius: 6px;
  transition: all 0.2s;
}

.legend-country:hover {
  background: rgba(255, 255, 255, 0.06);
  transform: translateX(-2px);
}

.country-rank {
  grid-row: 1 / 3;
  font-size: 14px;
  font-weight: 700;
  color: #666;
  text-align: center;
}

.country-flag {
  grid-row: 1 / 3;
  font-size: 24px;
  line-height: 1;
}

.country-info {
  grid-column: 3;
  display: flex;
  flex-direction: column;
  gap: 4px;
  min-width: 0;
}

.country-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 8px;
}

.country-name {
  font-size: 13px;
  font-weight: 500;
  color: #fff;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.country-percent {
  font-size: 12px;
  font-weight: 600;
  color: #22c55e;
  white-space: nowrap;
}

.country-stats {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.country-requests {
  font-size: 11px;
  color: #888;
  font-weight: 500;
}

.country-sparkline {
  opacity: 0.8;
}

.country-bar {
  grid-column: 1 / 4;
  height: 3px;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 2px;
  overflow: hidden;
}

.country-bar-fill {
  height: 100%;
  border-radius: 2px;
  transition: width 0.6s ease;
}

/* Detail Panel Styles */
.detail-panel {
  position: absolute;
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 100;
  background: rgba(15, 15, 20, 0.98);
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 12px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.6);
  backdrop-filter: blur(20px);
  width: 90%;
  max-width: 450px;
  max-height: calc(100vh - 180px);
  overflow-y: auto;
}

.detail-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.detail-title {
  display: flex;
  align-items: center;
  gap: 12px;
}

.detail-flag {
  font-size: 40px;
  line-height: 1;
}

.detail-title h3 {
  font-size: 18px;
  margin: 0;
  color: #fff;
  line-height: 1.2;
}

.detail-code {
  font-size: 12px;
  color: #666;
  text-transform: uppercase;
  letter-spacing: 1px;
}

.close-btn {
  background: rgba(255, 255, 255, 0.1);
  border: none;
  color: #fff;
  width: 32px;
  height: 32px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 18px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s;
}

.close-btn:hover {
  background: rgba(255, 255, 255, 0.2);
}

.detail-content {
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.detail-stat-card {
  background: rgba(255, 255, 255, 0.03);
  padding: 12px 14px;
  border-radius: 8px;
  border: 1px solid rgba(255, 255, 255, 0.05);
}

.stat-label {
  font-size: 12px;
  color: #888;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 8px;
}

.stat-value {
  font-size: 28px;
  font-weight: 700;
  color: #fff;
  line-height: 1;
  margin-bottom: 6px;
}

.stat-trend {
  font-size: 14px;
  font-weight: 600;
}

.stat-info {
  font-size: 13px;
  color: #666;
  margin-top: 4px;
}

.progress-bar-full {
  height: 6px;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 3px;
  overflow: hidden;
  margin-top: 8px;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #3b82f6, #22c55e);
  border-radius: 3px;
  transition: width 0.8s ease;
}

.chart-section {
  background: rgba(255, 255, 255, 0.03);
  padding: 12px 14px;
  border-radius: 8px;
  border: 1px solid rgba(255, 255, 255, 0.05);
}

.chart-section h4 {
  font-size: 13px;
  color: #fff;
  margin: 0 0 8px 0;
  font-weight: 500;
}

.request-chart {
  width: 100%;
  height: 70px;
}

.detail-actions {
  display: flex;
  gap: 8px;
  margin-top: 4px;
}

.action-btn {
  flex: 1;
  padding: 10px 14px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  background: rgba(255, 255, 255, 0.05);
  color: #fff;
  border-radius: 8px;
  cursor: pointer;
  font-size: 13px;
  font-weight: 500;
  transition: all 0.2s;
}

.action-btn:hover {
  background: rgba(255, 255, 255, 0.1);
  border-color: rgba(255, 255, 255, 0.3);
}

.action-btn.primary {
  background: #3b82f6;
  border-color: #3b82f6;
}

.action-btn.primary:hover {
  background: #2563eb;
}

/* Slide transition */
.slide-enter-active, .slide-leave-active {
  transition: all 0.3s ease;
}

.slide-enter-from {
  opacity: 0;
  transform: translateX(-50%) translateY(-20px);
}

.slide-leave-to {
  opacity: 0;
  transform: translateX(-50%) translateY(-20px);
}

@media (max-width: 1024px) {
  .map-wrapper {
    height: 60vh;
  }
  
  .stats-card {
    top: 10px;
    left: 10px;
    padding: 12px 16px;
    min-width: 180px;
  }
  
  .stats-value {
    font-size: 28px;
  }
  
  .legend-card {
    top: 10px;
    right: 50px;
    min-width: 260px;
    max-width: 280px;
  }
}

@media (max-width: 768px) {
  .map-wrapper {
    height: 500px;
  }
  
  .controls-bar {
    flex-direction: column;
    gap: 0.5rem;
  }
  
  .stats-card {
    position: relative;
    top: auto;
    left: auto;
    margin-bottom: 1rem;
    width: 100%;
  }
  
  .legend-card {
    position: relative;
    top: auto;
    right: auto;
    margin-bottom: 1rem;
    width: 100%;
    max-width: 100%;
  }
}
</style>
