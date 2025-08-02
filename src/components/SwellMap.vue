<script setup>
import { ref, onMounted, watch, nextTick } from 'vue'

const props = defineProps({
  selectedSpot: Object,
  swellWindows: Array,
  communitySpots: Array,
  isLoadingCommunity: Boolean
})

const emit = defineEmits(['spot-selected', 'windows-updated', 'load-community-spot', 'spot-completed', 'save-to-community'])
const mapContainer = ref(null)
let windyAPI = null
let map = null
let L = null // Store Leaflet instance from Windy
let spotMarker = null
let currentWindow = null
let windowLines = []
let isDrawingMode = false
let communityMarkers = []
let trianglePolygons = []

// Helper function to extend triangle points to create infinite ocean exposure
const createInfiniteTriangle = (spotLat, spotLng, point1Lat, point1Lng, point2Lat, point2Lng) => {
  // Fixed extension distance in degrees (approximately 2000km)
  const extensionDistance = 20 // degrees
  
  // Calculate normalized direction vectors from spot to each point
  const dir1 = {
    lat: point1Lat - spotLat,
    lng: point1Lng - spotLng
  }
  const dir2 = {
    lat: point2Lat - spotLat,
    lng: point2Lng - spotLng
  }
  
  // Calculate the magnitude (distance) of each direction vector
  const magnitude1 = Math.sqrt(dir1.lat * dir1.lat + dir1.lng * dir1.lng)
  const magnitude2 = Math.sqrt(dir2.lat * dir2.lat + dir2.lng * dir2.lng)
  
  // Normalize the direction vectors
  const normalizedDir1 = {
    lat: dir1.lat / magnitude1,
    lng: dir1.lng / magnitude1
  }
  const normalizedDir2 = {
    lat: dir2.lat / magnitude2,
    lng: dir2.lng / magnitude2
  }
  
  // Extend both points by the same fixed distance
  const extendedPoint1 = {
    lat: spotLat + normalizedDir1.lat * extensionDistance,
    lng: spotLng + normalizedDir1.lng * extensionDistance
  }
  
  const extendedPoint2 = {
    lat: spotLat + normalizedDir2.lat * extensionDistance,
    lng: spotLng + normalizedDir2.lng * extensionDistance
  }
  
  // Create the polygon points for an extended triangle (format for Windy: [lat, lng] pairs)
  return [
    [spotLat, spotLng],
    [point1Lat, point1Lng],
    [extendedPoint1.lat, extendedPoint1.lng],
    [extendedPoint2.lat, extendedPoint2.lng],
    [point2Lat, point2Lng]
  ]
}

// Load Windy API scripts
const loadWindyAPI = () => {
  return new Promise((resolve, reject) => {
    // Check if already loaded
    if (window.windyInit) {
      resolve()
      return
    }
    
    // Load Leaflet first (required by Windy)
    const leafletScript = document.createElement('script')
    leafletScript.src = 'https://unpkg.com/leaflet@1.4.0/dist/leaflet.js'
    leafletScript.onload = () => {
      // Load Leaflet CSS
      const leafletCSS = document.createElement('link')
      leafletCSS.rel = 'stylesheet'
      leafletCSS.href = 'https://unpkg.com/leaflet@1.4.0/dist/leaflet.css'
      document.head.appendChild(leafletCSS)
      
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

onMounted(async () => {
  try {
    // Load Windy API
    await loadWindyAPI()
    
    const apiKey = import.meta.env.VITE_WINDY_API_KEY
    if (!apiKey || apiKey === 'YOUR_WINDY_API_KEY') {
      console.warn('Windy API key not configured. Map functionality will be limited.')
    }
    
    // Get user's location first to center the map
    const userLocation = await getUserLocation()
    
    const options = {
      key: apiKey || 'demo',
      lat: userLocation.lat,
      lon: userLocation.lng,
      zoom: 10,
      overlay: 'wind' // Start with wind overlay
    }
    
    // Initialize windy map API
    window.windyInit(options, (windyAPIInstance) => {
      const { map: windyMap, store, picker } = windyAPIInstance
      
      // Store references
      windyAPI = windyAPIInstance
      map = windyMap
      L = window.L // Get Leaflet instance that Windy loaded
      
      // Set up map click handler
      map.on('click', handleMapClick)
      
      // Add community spots now that L is available
      addCommunitySpots()
      
      console.log('Windy map initialized successfully')
    })
  } catch (error) {
    console.error('Failed to initialize Windy map:', error)
  }
})

const getUserLocation = () => {
  return new Promise((resolve) => {
    if (navigator.geolocation) {
      navigator.geolocation.getCurrentPosition(
        (position) => {
          const { latitude, longitude } = position.coords
          console.log('Location found:', latitude, longitude)
          resolve({ lat: latitude, lng: longitude })
        },
        async (error) => {
          console.warn('Geolocation error:', error.message)
          // Fallback to IP-based location detection
          const ipLocation = await getLocationFromIP()
          resolve(ipLocation)
        },
        {
          enableHighAccuracy: true,
          timeout: 10000,
          maximumAge: 300000 // 5 minutes
        }
      )
    } else {
      console.warn('Geolocation is not supported by this browser')
      getLocationFromIP().then(resolve)
    }
  })
}

const getLocationFromIP = async () => {
  try {
    // Use a free IP geolocation service
    const response = await fetch('https://ipapi.co/json/')
    const data = await response.json()
    
    if (data.latitude && data.longitude) {
      const { latitude, longitude, city, country } = data
      console.log('IP location found:', city, country, latitude, longitude)
      return { lat: latitude, lng: longitude, city, country }
    }
  } catch (error) {
    console.warn('IP geolocation failed:', error.message)
  }
  
  // Default to Los Angeles
  return { lat: 34.0522, lng: -118.2437, city: 'Los Angeles', country: 'US' }
}

const addCommunitySpots = () => {
  if (!props.communitySpots || props.communitySpots.length === 0 || !map || !L) return
  
  // Clear existing community markers
  communityMarkers.forEach(marker => map.removeLayer(marker))
  communityMarkers = []
  
  props.communitySpots.forEach(spotConfig => {
    // Add the community spot marker using Leaflet (Windy uses Leaflet underneath)
    const marker = L.circleMarker([spotConfig.location.lat, spotConfig.location.lng], {
      radius: 8,
      fillColor: '#28a745',
      color: '#fff',
      weight: 2,
      opacity: 1,
      fillOpacity: 0.9
    }).addTo(map)
    
    marker.bindPopup(`
      <div style="text-align: center;">
        <strong>🌍 ${spotConfig.name}</strong><br>
        <small>Community Spot</small><br>
        Windows: ${spotConfig.swellWindows.length}<br>
        <button onclick="window.loadCommunitySpot('${spotConfig.id}')" style="margin-top: 8px; padding: 4px 8px; background: #28a745; color: white; border: none; border-radius: 4px; cursor: pointer;">
          Load This Spot
        </button>
      </div>
    `)
    
    marker.bindTooltip(`${spotConfig.name} (Community)`, { permanent: false, direction: 'top' })
    communityMarkers.push(marker)
    
    // Draw swell window triangles immediately for all community spots
    if (spotConfig.swellWindows && spotConfig.swellWindows.length > 0) {
      spotConfig.swellWindows.forEach(window => {
        // Create infinite triangle points
        const trianglePoints = createInfiniteTriangle(
          spotConfig.location.lat, spotConfig.location.lng,
          window.point1.lat, window.point1.lng,
          window.point2.lat, window.point2.lng
        )
        
        // Draw triangle with community spot styling (dark grey for visibility)
        const triangle = L.polygon(trianglePoints, {
          color: '#2c3e50',
          weight: 2,
          opacity: 0.8,
          fillColor: '#34495e',
          fillOpacity: 0.3
        }).addTo(map)
        
        trianglePolygons.push(triangle)
        
        // Add small markers at triangle endpoints
        const marker1 = L.circleMarker([window.point1.lat, window.point1.lng], {
          radius: 4,
          fillColor: '#2c3e50',
          color: '#fff',
          weight: 1,
          opacity: 0.8,
          fillOpacity: 0.8
        }).addTo(map)
        
        const marker2 = L.circleMarker([window.point2.lat, window.point2.lng], {
          radius: 4,
          fillColor: '#2c3e50',
          color: '#fff',
          weight: 1,
          opacity: 0.8,
          fillOpacity: 0.8
        }).addTo(map)
        
        communityMarkers.push(marker1, marker2)
      })
    }
  })
}

// Make function available globally for popup buttons
window.loadCommunitySpot = (spotId) => {
  const spotConfig = props.communitySpots.find(s => s.id === spotId)
  if (spotConfig && L) {
    // First remove any existing loaded spot visualizations
    map.eachLayer((layer) => {
      // Remove loaded spot triangles (purple polygons)
      if (layer instanceof L.Polygon && layer.options.fillColor === '#8e44ad') {
        map.removeLayer(layer)
      }
      // Remove loaded spot markers (purple circles)
      if (layer instanceof L.CircleMarker && layer.options.fillColor === '#8e44ad') {
        map.removeLayer(layer)
      }
    })
    
    // Draw the loaded spot's windows in a distinctive purple color
    if (spotConfig.swellWindows && spotConfig.swellWindows.length > 0) {
      spotConfig.swellWindows.forEach(window => {
        // Create infinite triangle points
        const trianglePoints = createInfiniteTriangle(
          spotConfig.location.lat, spotConfig.location.lng,
          window.point1.lat, window.point1.lng,
          window.point2.lat, window.point2.lng
        )
        
        // Draw triangle with loaded spot styling (purple)
        L.polygon(trianglePoints, {
          color: '#8e44ad',
          weight: 3,
          opacity: 0.9,
          fillColor: '#8e44ad',
          fillOpacity: 0.3
        }).addTo(map)
        
        // Add distinctive markers at triangle endpoints
        L.circleMarker([window.point1.lat, window.point1.lng], {
          radius: 6,
          fillColor: '#8e44ad',
          color: '#fff',
          weight: 2,
          opacity: 1,
          fillOpacity: 1
        }).addTo(map)
        
        L.circleMarker([window.point2.lat, window.point2.lng], {
          radius: 6,
          fillColor: '#8e44ad',
          color: '#fff',
          weight: 2,
          opacity: 1,
          fillOpacity: 1
        }).addTo(map)
      })
    }
    
    // Only emit the configuration to populate the analysis panel
    emit('load-community-spot', spotConfig)
  }
}

// Watch for changes in community spots and update markers
watch(() => props.communitySpots, () => {
  if (map && L) {
    addCommunitySpots()
  }
}, { deep: true })

const handleMapClick = (e) => {
  const { lat, lng } = e.latlng
  
  if (!props.selectedSpot) {
    // Select a spot
    selectSpot(lat, lng)
  } else if (isDrawingMode) {
    // Add point to current window
    addWindowPoint(lat, lng)
  }
}

const selectSpot = (lat, lng) => {
  if (!L) return // Make sure Leaflet is available
  
  // Clear existing spot marker
  if (spotMarker) {
    map.removeLayer(spotMarker)
  }
  
  // Add new spot marker using Leaflet
  spotMarker = L.marker([lat, lng], {
    icon: L.divIcon({
      className: 'spot-marker',
      html: '📍',
      iconSize: [30, 30],
      iconAnchor: [15, 30]
    })
  }).addTo(map)
  
  // Get spot name from user
  const spotName = prompt('Enter a name for this spot:', `Spot at ${lat.toFixed(4)}, ${lng.toFixed(4)}`) || `Spot at ${lat.toFixed(4)}, ${lng.toFixed(4)}`
  
  // Create spot data
  const spotData = {
    name: spotName,
    coordinates: { lat, lng },
    timestamp: new Date().toISOString()
  }
  
  // Emit the selected spot
  emit('spot-selected', spotData)
  
  // Add popup to marker
  spotMarker.bindPopup(`
    <div style="text-align: center;">
      <strong>${spotName}</strong><br>
      Lat: ${lat.toFixed(4)}<br>
      Lng: ${lng.toFixed(4)}<br>
      <button onclick="window.startDrawing()" style="margin-top: 8px; padding: 4px 8px; background: #007bff; color: white; border: none; border-radius: 4px; cursor: pointer;">
        Draw Swell Window
      </button>
    </div>
  `).openPopup()
}

const addWindowPoint = (lat, lng) => {
  if (!L) return // Make sure Leaflet is available
  
  if (!currentWindow) {
    // Start new window
    currentWindow = {
      point1: { lat, lng },
      point2: null,
      marker1: L.circleMarker([lat, lng], {
        radius: 8,
        fillColor: '#2c3e50',
        color: '#fff',
        weight: 2,
        opacity: 1,
        fillOpacity: 1
      }).addTo(map)
    }
  } else if (!currentWindow.point2) {
    // Complete the window
    currentWindow.point2 = { lat, lng }
    currentWindow.marker2 = L.circleMarker([lat, lng], {
      radius: 8,
      fillColor: '#2c3e50',
      color: '#fff',
      weight: 2,
      opacity: 1,
      fillOpacity: 1
    }).addTo(map)
    
    // Draw the swell window triangle with infinite ocean exposure
    const trianglePoints = createInfiniteTriangle(
      props.selectedSpot.coordinates.lat, props.selectedSpot.coordinates.lng,
      currentWindow.point1.lat, currentWindow.point1.lng,
      currentWindow.point2.lat, currentWindow.point2.lng
    )
    
    const triangle = L.polygon(trianglePoints, {
      color: '#2c3e50',
      weight: 3,
      opacity: 0.9,
      fillColor: '#34495e',
      fillOpacity: 0.4
    }).addTo(map)
    
    currentWindow.triangle = triangle
    windowLines.push(currentWindow)
    
    // Update swell windows
    emit('windows-updated', [...windowLines])
    
    // Reset for next window
    currentWindow = null
    isDrawingMode = false
    
    // Update UI
    document.querySelector('.drawing-indicator').style.display = 'none'
  }
}

const startDrawing = () => {
  isDrawingMode = true
  document.querySelector('.drawing-indicator').style.display = 'block'
}

const clearWindows = () => {
  windowLines.forEach(window => {
    if (window.marker1) map.removeLayer(window.marker1)
    if (window.marker2) map.removeLayer(window.marker2)
    if (window.triangle) map.removeLayer(window.triangle)
  })
  windowLines = []
  currentWindow = null
  isDrawingMode = false
  document.querySelector('.drawing-indicator').style.display = 'none'
  emit('windows-updated', [])
}

const resetSpot = () => {
  if (spotMarker) {
    map.removeLayer(spotMarker)
    spotMarker = null
  }
  clearWindows()
  emit('spot-selected', null)
}

const markAsDone = async () => {
  // Exit drawing mode if active
  isDrawingMode = false
  document.querySelector('.drawing-indicator').style.display = 'none'
  
  // If user has windows, save to community before resetting
  if (windowLines.length > 0 && props.selectedSpot) {
    try {
      // Create the spot configuration for saving
      const spotConfig = {
        spot: props.selectedSpot,
        swellWindows: windowLines.map(window => ({
          point1: window.point1,
          point2: window.point2
        })),
        exposureAngles: [], // Will be calculated by parent
        timestamp: new Date().toISOString(),
        version: '1.0'
      }
      
      // Emit to parent to save to community
      emit('save-to-community', spotConfig)
    } catch (error) {
      console.error('Error saving spot:', error)
    }
  }
  
  // Reset to opening state - clear spot and windows
  if (spotMarker) {
    map.removeLayer(spotMarker)
    spotMarker = null
  }
  
  // Clear all windows
  windowLines.forEach(window => {
    if (window.marker1) map.removeLayer(window.marker1)
    if (window.marker2) map.removeLayer(window.marker2)
    if (window.triangle) map.removeLayer(window.triangle)
  })
  windowLines = []
  currentWindow = null
  
  // Emit events to reset parent component state
  emit('windows-updated', [])
  emit('spot-selected', null)
  emit('spot-completed')
}

// Make functions available globally for popup buttons
window.startDrawing = startDrawing
window.clearWindows = clearWindows
window.resetSpot = resetSpot
</script>

<template>
  <div class="map-wrapper">
    <!-- Windy will initialize its map in this div -->
    <div id="windy" class="map-container"></div>
    <div class="map-overlay">
      <div class="instructions">
        <h3>🌊 Swell Window Calculator</h3>
        <div class="location-status">
          <small>📍 Map with real weather data</small>
        </div>
        <div class="step" v-if="!selectedSpot">
          <strong>Step 1:</strong> Click on the map to select and name a spot
        </div>
        <div class="step" v-else-if="swellWindows.length === 0">
          <strong>Step 2:</strong> Click "Draw Swell Window" in the popup, then click two points to create a swell opening
        </div>
        <div class="step" v-else>
          <strong>Step 3:</strong> Add more windows or view your exposure angles
        </div>
        <div class="controls" v-if="selectedSpot">
          <button @click="startDrawing" class="control-btn" :disabled="isDrawingMode">
            ➕ Add Window
          </button>
          <button @click="clearWindows" class="control-btn" v-if="swellWindows.length > 0">
            🗑️ Clear Windows
          </button>
          <button @click="resetSpot" class="control-btn">
            🔄 Reset Spot
          </button>
          <button @click="markAsDone" class="control-btn done-btn" v-if="swellWindows.length > 0">
            ✅ Done
          </button>
        </div>
      </div>
    </div>
    <div class="drawing-indicator" style="display: none;">
      <p>🎯 Click two points to define the swell window opening</p>
    </div>
  </div>
</template>

<style scoped>
.map-wrapper {
  position: relative;
  height: 100%;
  min-height: 600px;
}

.map-container {
  height: 100%;
  width: 100%;
  border-radius: 12px;
}

.map-overlay {
  position: absolute;
  top: 1rem;
  left: 1rem;
  z-index: 1000;
  background: rgba(255, 255, 255, 0.95);
  padding: 1rem;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
  max-width: 250px;
}

.instructions h3 {
  margin-bottom: 0.5rem;
  color: #222;
  font-size: 1.2rem;
  font-weight: bold;
}

.location-status {
  margin-bottom: 0.75rem;
  padding: 0.25rem 0.5rem;
  background: rgba(40, 167, 69, 0.1);
  border-radius: 4px;
  border-left: 3px solid #28a745;
}

.location-status small {
  color: #1b5e20;
  font-weight: bold;
}

.instructions p {
  font-size: 0.9rem;
  color: #666;
  margin-bottom: 1rem;
  line-height: 1.4;
}

.step {
  background: #e3e3e3;
  padding: 0.5rem;
  border-radius: 4px;
  margin-bottom: 1rem;
  font-size: 1rem;
  border-left: 3px solid #007bff;
  color: #222;
  font-weight: 500;
}

.controls {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.control-btn {
  padding: 0.5rem;
  border: none;
  border-radius: 4px;
  background: #007bff;
  color: #fff;
  font-size: 1rem;
  cursor: pointer;
  transition: background 0.2s;
  font-weight: bold;
}

.control-btn:hover:not(:disabled) {
  background: #0056b3;
}

.control-btn:disabled {
  background: #6c757d;
  cursor: not-allowed;
}

.control-btn.done-btn {
  background: #28a745;
}

.control-btn.done-btn:hover {
  background: #218838;
}

.drawing-indicator {
  position: absolute;
  top: 1rem;
  left: 50%;
  transform: translateX(-50%);
  background: #222;
  color: #fff;
  padding: 1rem 2rem;
  border-radius: 8px;
  box-shadow: 0 4px 20px rgba(0,0,0,0.3);
  z-index: 1001;
  text-align: center;
  font-weight: bold;
  font-size: 1.1rem;
}

.legend {
  border-top: 1px solid #eee;
  padding-top: 0.5rem;
}

.legend-item {
  display: flex;
  align-items: center;
  font-size: 0.8rem;
  color: #666;
}

.marker-sample {
  width: 12px;
  height: 12px;
  border-radius: 50%;
  background-color: #3388ff;
  border: 2px solid white;
  margin-right: 0.5rem;
  box-shadow: 0 1px 3px rgba(0,0,0,0.3);
}

/* Fix for Leaflet markers */
:global(.leaflet-marker-icon) {
  margin-left: -12px !important;
  margin-top: -41px !important;
}

:global(.spot-marker) {
  background: none !important;
  border: none !important;
  text-align: center;
  font-size: 24px;
}

:global(.user-location-marker) {
  background: none !important;
  border: none !important;
  text-align: center;
  font-size: 20px;
  filter: drop-shadow(0 2px 4px rgba(0,0,0,0.3));
}

:global(.ip-location-marker) {
  background: none !important;
  border: none !important;
  text-align: center;
  font-size: 20px;
  opacity: 0.8;
  filter: drop-shadow(0 2px 4px rgba(0,0,0,0.3));
}
</style>
