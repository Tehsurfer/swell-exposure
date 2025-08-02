<template>
  <div class="windy-integration">
    <div class="windy-controls">
      <h4>🌊 Windy Integration</h4>
      <div class="controls-row">
        <label for="data-type">Show:</label>
        <select v-model="selectedDataType" @change="updateWindyData" id="data-type">
          <option value="wind">Wind</option>
          <option value="waves">Waves/Swell</option>
        </select>
        <button @click="toggleWindyOverlay" class="windy-btn">
          {{ showWindyData ? 'Hide' : 'Show' }} {{ selectedDataType }}
        </button>
      </div>
      <div class="info-panel" v-if="showWindyData && windData">
        <div class="data-item">
          <strong>{{ selectedDataType === 'wind' ? 'Wind' : 'Swell' }} Direction:</strong> 
          {{ windData.direction }}°
        </div>
        <div class="data-item">
          <strong>{{ selectedDataType === 'wind' ? 'Wind Speed' : 'Wave Height' }}:</strong> 
          {{ windData.strength }}{{ selectedDataType === 'wind' ? ' kt' : ' m' }}
        </div>
        <div class="data-item" v-if="windData.period">
          <strong>Wave Period:</strong> {{ windData.period }}s
        </div>
        <div class="data-item" v-if="swellWindowMatch">
          <strong>Window Match:</strong> 
          <span :class="swellWindowMatch.class">{{ swellWindowMatch.text }}</span>
        </div>
      </div>
      <div class="loading" v-if="isLoading">
        <div class="spinner"></div>
        Loading Windy data...
      </div>
      <div class="error-panel" v-if="apiError">
        <div class="error-message">
          <strong>⚠️ {{ apiError.title }}</strong><br>
          {{ apiError.message }}
        </div>
      </div>
    </div>
    
    <!-- Windy Map Container -->
    <div v-if="showWindyData" class="windy-map-container">
      <div ref="windyMapContainer" class="windy-map"></div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted, nextTick } from 'vue'

const props = defineProps({
  selectedSpot: Object,
  swellWindows: Array,
  mapBounds: Object
})

const emit = defineEmits(['windy-data-updated'])

const selectedDataType = ref('waves')
const showWindyData = ref(false)
const windData = ref(null)
const isLoading = ref(false)
const windyMapContainer = ref(null)
const apiError = ref(null)
let windyAPI = null

// Load Windy API scripts
const loadWindyAPI = () => {
  return new Promise((resolve, reject) => {
    // Check if already loaded
    if (window.windyInit) {
      resolve()
      return
    }
    
    // Load Leaflet first
    const leafletScript = document.createElement('script')
    leafletScript.src = 'https://unpkg.com/leaflet@1.4.0/dist/leaflet.js'
    leafletScript.onload = () => {
      // Then load Windy API
      const windyScript = document.createElement('script')
      windyScript.src = 'https://api.windy.com/assets/map-forecast/libBoot.js'
      windyScript.onload = resolve
      windyScript.onerror = reject
      document.head.appendChild(windyScript)
    }
    leafletScript.onerror = reject
    document.head.appendChild(leafletScript)
  })
}

// Initialize Windy map
const initWindyMap = async () => {
  if (!props.selectedSpot) return
  
  try {
    isLoading.value = true
    await loadWindyAPI()
    
    const apiKey = import.meta.env.VITE_WINDY_API_KEY
    if (!apiKey || apiKey === 'YOUR_WINDY_API_KEY') {
      console.warn('Windy API key not configured. Using demo mode.')
      // Fall back to demo/mock data
      await simulateWindyData()
      return
    }
    
    const options = {
      // Required, API key
      key: apiKey,
      
      // Optional: Initial state of the map
      lat: props.selectedSpot.coordinates.lat,
      lon: props.selectedSpot.coordinates.lng,
      zoom: 10,
      
      // Start with wind overlay as it's more likely to be available
      overlay: 'wind',
    }
    
    // Initialize windy map API - it will automatically use the #windy div
    window.windyInit(options, (windyAPIInstance) => {
      // windyAPI is ready, and map has been created
      const { map, store, picker } = windyAPIInstance
      
      // Store the API reference
      windyAPI = windyAPIInstance
      
      // Check what overlays are available
      const availableOverlays = store.getAllowed('overlay')
      console.log('Available overlays:', availableOverlays)
      
      // Try to find the best wave overlay if waves are requested
      if (selectedDataType.value === 'waves') {
        const waveOptions = ['waves', 'ecmwfWaves', 'gfsWaves', 'iconEuWaves', 'iconWaves']
        const availableWaveOverlay = waveOptions.find(overlay => availableOverlays.includes(overlay))
        
        if (availableWaveOverlay) {
          store.set('overlay', availableWaveOverlay)
          console.log('Using wave overlay:', availableWaveOverlay)
          apiError.value = null // Clear any previous errors
        } else {
          apiError.value = {
            title: 'Wave Data Not Available',
            message: 'No wave overlays found in your API subscription. Showing wind data instead.'
          }
          selectedDataType.value = 'wind'
          store.set('overlay', 'wind')
        }
      }
      
      // Show the Windy div and move it to our container
      const windyDiv = document.getElementById('windy')
      if (windyDiv && windyMapContainer.value) {
        windyDiv.style.display = 'block'
        windyDiv.style.width = '100%'
        windyDiv.style.height = '300px'
        windyMapContainer.value.appendChild(windyDiv)
      }
      
      // Set up picker for getting weather data at specific coordinates
      picker.on('pickerOpened', async ({ lat, lon }) => {
        await nextTick()
        const data = picker.getParams()
        updateWindDataFromPicker(data)
      })
      
      picker.on('pickerMoved', ({ lat, lon, values, overlay }) => {
        updateWindDataFromPicker({ lat, lon, values, overlay })
      })
      
      // Open picker at the selected spot
      picker.open({
        lat: props.selectedSpot.coordinates.lat,
        lon: props.selectedSpot.coordinates.lng
      })
      
      isLoading.value = false
    })
  } catch (error) {
    console.error('Failed to initialize Windy API:', error)
    // Fall back to demo/mock data
    await simulateWindyData()
  }
}

// Simulate Windy data when API is not available
const simulateWindyData = async () => {
  await new Promise(resolve => setTimeout(resolve, 1000)) // Simulate loading
  
  const mockData = {
    direction: Math.floor(Math.random() * 360),
    strength: selectedDataType.value === 'wind' ? 
      Math.floor(Math.random() * 25) + 5 : 
      (Math.random() * 3 + 0.5).toFixed(1),
    period: selectedDataType.value === 'waves' ? Math.floor(Math.random() * 10) + 6 : undefined,
    type: selectedDataType.value
  }
  
  windData.value = mockData
  emit('windy-data-updated', {
    dataType: selectedDataType.value,
    data: mockData
  })
  
  isLoading.value = false
}

// Update wind data from Windy picker
const updateWindDataFromPicker = (pickerData) => {
  if (!pickerData || !pickerData.values) return
  
  const values = pickerData.values
  let processedData = {}
  
  if (selectedDataType.value === 'wind') {
    // Process wind data
    if (values.wind_u !== undefined && values.wind_v !== undefined) {
      const windSpeed = Math.sqrt(values.wind_u * values.wind_u + values.wind_v * values.wind_v)
      const windDirection = (Math.atan2(values.wind_v, values.wind_u) * 180 / Math.PI + 180) % 360
      
      processedData = {
        direction: Math.round(windDirection),
        strength: Math.round(windSpeed * 1.944), // Convert m/s to knots
        type: 'wind'
      }
    }
  } else {
    // Process wave data
    if (values.waves) {
      const waveData = windyAPI.utils.wave2obj(values.waves)
      processedData = {
        direction: Math.round(waveData.dir),
        strength: (waveData.size).toFixed(1),
        period: Math.round(waveData.period),
        type: 'waves'
      }
    }
  }
  
  if (processedData.direction !== undefined) {
    windData.value = processedData
    emit('windy-data-updated', {
      dataType: selectedDataType.value,
      data: processedData
    })
  }
}

const updateWindyData = async () => {
  if (!props.selectedSpot || !showWindyData.value) return
  
  isLoading.value = true
  apiError.value = null
  
  try {
    if (windyAPI) {
      // Check if the requested overlay type is available
      const availableOverlays = windyAPI.store.getAllowed('overlay')
      
      if (selectedDataType.value === 'wind') {
        windyAPI.store.set('overlay', 'wind')
      } else {
        // Try to find the best wave overlay
        const waveOptions = ['waves', 'ecmwfWaves', 'gfsWaves', 'iconEuWaves', 'iconWaves']
        const availableWaveOverlay = waveOptions.find(overlay => availableOverlays.includes(overlay))
        
        if (availableWaveOverlay) {
          windyAPI.store.set('overlay', availableWaveOverlay)
          console.log('Switching to wave overlay:', availableWaveOverlay)
          apiError.value = null // Clear any previous errors
        } else {
          apiError.value = {
            title: 'Wave Data Not Available',
            message: 'No wave overlays found in your API subscription. Showing wind data instead.'
          }
          selectedDataType.value = 'wind'
          windyAPI.store.set('overlay', 'wind')
        }
      }
      
      // Reopen picker to get new data
      windyAPI.picker.open({
        lat: props.selectedSpot.coordinates.lat,
        lon: props.selectedSpot.coordinates.lng
      })
    } else {
      // Initialize if not already done
      await initWindyMap()
    }
  } catch (error) {
    console.error('Error updating Windy data:', error)
    apiError.value = {
      title: 'Windy API Error',
      message: 'Failed to load weather data. Using simulated data instead.'
    }
    await simulateWindyData()
  } finally {
    isLoading.value = false
  }
}

const toggleWindyOverlay = async () => {
  showWindyData.value = !showWindyData.value
  if (showWindyData.value) {
    await nextTick()
    await updateWindyData()
  } else {
    windData.value = null
    // Clean up Windy map if needed
    if (windyAPI && windyAPI.picker) {
      windyAPI.picker.close()
    }
    // Hide the Windy div and move it back to body
    const windyDiv = document.getElementById('windy')
    if (windyDiv) {
      windyDiv.style.display = 'none'
      document.body.appendChild(windyDiv)
    }
  }
}

// Check if wind/swell direction matches any swell windows
const swellWindowMatch = computed(() => {
  if (!windData.value || !props.swellWindows || props.swellWindows.length === 0) {
    return null
  }
  
  const windDirection = windData.value.direction
  
  // Calculate angles for each swell window
  for (const window of props.swellWindows) {
    if (!props.selectedSpot) continue
    
    const spotLat = props.selectedSpot.coordinates.lat
    const spotLng = props.selectedSpot.coordinates.lng
    
    // Calculate angles from spot to window points
    const angle1 = Math.atan2(
      window.point1.lat - spotLat,
      window.point1.lng - spotLng
    ) * 180 / Math.PI
    
    const angle2 = Math.atan2(
      window.point2.lat - spotLat,
      window.point2.lng - spotLng
    ) * 180 / Math.PI
    
    // Normalize angles to 0-360
    const normalizedAngle1 = (angle1 + 360) % 360
    const normalizedAngle2 = (angle2 + 360) % 360
    
    // Check if wind direction falls within this window
    const minAngle = Math.min(normalizedAngle1, normalizedAngle2)
    const maxAngle = Math.max(normalizedAngle1, normalizedAngle2)
    
    // Handle wrapping around 360/0
    let isInWindow = false
    if (maxAngle - minAngle > 180) {
      // Window crosses 0 degrees
      isInWindow = windDirection >= maxAngle || windDirection <= minAngle
    } else {
      isInWindow = windDirection >= minAngle && windDirection <= maxAngle
    }
    
    if (isInWindow) {
      return {
        text: `${selectedDataType.value === 'wind' ? 'Wind' : 'Swell'} is coming through this window!`,
        class: 'match-good'
      }
    }
  }
  
  return {
    text: `${selectedDataType.value === 'wind' ? 'Wind' : 'Swell'} is not aligned with windows`,
    class: 'match-poor'
  }
})

// Watch for spot changes
watch(() => props.selectedSpot, () => {
  if (showWindyData.value) {
    updateWindyData()
  }
})

onMounted(() => {
  // Auto-enable for demonstration
  if (props.selectedSpot) {
    selectedDataType.value = 'waves'
  }
})
</script>

<style scoped>
.windy-integration {
  position: relative;
  width: 100%;
}

.windy-controls {
  background: rgba(255, 255, 255, 0.95);
  padding: 1rem;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
  margin-bottom: 1rem;
}

.windy-controls h4 {
  margin: 0 0 0.75rem 0;
  color: #222;
  font-size: 1.1rem;
}

.controls-row {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  flex-wrap: wrap;
}

.controls-row label {
  font-weight: 500;
  color: #444;
}

.controls-row select {
  padding: 0.375rem 0.5rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  background: white;
}

.windy-btn {
  padding: 0.375rem 0.75rem;
  background: #17a2b8;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: 500;
  transition: background 0.2s;
}

.windy-btn:hover {
  background: #138496;
}

.info-panel {
  margin-top: 1rem;
  padding: 0.75rem;
  background: #f8f9fa;
  border-radius: 6px;
  border-left: 4px solid #17a2b8;
}

.data-item {
  margin-bottom: 0.5rem;
  font-size: 0.9rem;
}

.data-item:last-child {
  margin-bottom: 0;
}

.match-good {
  color: #28a745;
  font-weight: bold;
}

.match-poor {
  color: #dc3545;
  font-weight: bold;
}

.loading {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  margin-top: 1rem;
  color: #17a2b8;
  font-weight: 500;
}

.spinner {
  width: 16px;
  height: 16px;
  border: 2px solid #f3f3f3;
  border-top: 2px solid #17a2b8;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.error-panel {
  margin-top: 1rem;
  padding: 0.75rem;
  background: #fff3cd;
  border-radius: 6px;
  border-left: 4px solid #ffc107;
}

.error-message {
  color: #856404;
  font-size: 0.9rem;
}

.windy-map-container {
  margin-top: 1rem;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
}

.windy-map {
  width: 100%;
  height: 300px;
  min-height: 300px;
}
</style>
