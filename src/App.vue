<script setup>
import { ref, onMounted } from 'vue'
import SwellMap from './components/SwellMap.vue'
import SwellWindowCalculator from './components/SwellWindowCalculator.vue'

const selectedSpot = ref(null)
const swellWindows = ref([])
const exposureAngles = ref(null)
const communitySpots = ref([])
const isLoadingCommunityData = ref(true)

// Shared JSONBin ID for the community database
const COMMUNITY_DB_ID = '688d73b2f7e7a370d1f1f48f' // This will be our shared database

onMounted(() => {
  loadCommunityDatabase()
})

const onSpotSelected = (spot) => {
  selectedSpot.value = spot
  // Reset windows when new spot is selected
  swellWindows.value = []
  exposureAngles.value = null
}

const onWindowsUpdated = (windows) => {
  swellWindows.value = windows
  calculateExposureAngles()
}

const onConfigurationLoaded = (config) => {
  // Load simplified community spot format
  selectedSpot.value = {
    name: config.name,
    coordinates: config.location
  }
  swellWindows.value = config.swellWindows
  calculateExposureAngles()
}

const onSpotCompleted = () => {
  // Handle when user clicks "Done" - spot is already reset by the map component
  // No additional action needed here since the map component handles the reset
}

const loadCommunityDatabase = async () => {
  try {
    const response = await fetch(`https://api.jsonbin.io/v3/b/${COMMUNITY_DB_ID}`)
    if (response.ok) {
      const result = await response.json()
      communitySpots.value = result.record.spots || []
      console.log(`Loaded ${communitySpots.value.length} community spots`)
    } else {
      // If the bin doesn't exist, we'll create it when the first spot is added
      communitySpots.value = []
    }
  } catch (error) {
    console.warn('Failed to load community database:', error)
    communitySpots.value = []
  }
  isLoadingCommunityData.value = false
}

const addSpotToCommunity = async (spotConfig) => {
  try {
    // Create simplified spot data with just the essential information
    const simplifiedSpot = {
      id: Date.now().toString(),
      name: spotConfig.spot.name,
      location: {
        lat: spotConfig.spot.coordinates.lat,
        lng: spotConfig.spot.coordinates.lng
      },
      swellWindows: spotConfig.swellWindows.map(window => ({
        point1: { lat: window.point1.lat, lng: window.point1.lng },
        point2: { lat: window.point2.lat, lng: window.point2.lng }
      })),
      addedAt: new Date().toISOString(),
      addedBy: 'Anonymous User'
    }
    
    // Add to local array
    communitySpots.value.push(simplifiedSpot)
    
    // Update the shared database
    const updatedDatabase = {
      spots: communitySpots.value,
      lastUpdated: new Date().toISOString(),
      version: '1.0'
    }
    
    const response = await fetch(`https://api.jsonbin.io/v3/b/${COMMUNITY_DB_ID}`, {
      method: 'PUT',
      headers: {
        'Content-Type': 'application/json',
        'X-Access-Key': decodeURIComponent(import.meta.env.VITE_JSONBIN_ACCESS_KEY) // Decode the URL-encoded key
      },
      body: JSON.stringify(updatedDatabase)
    })
    
    if (response.ok) {
      return { success: true, message: `"${simplifiedSpot.name}" added to community database!` }
    } else {
      throw new Error('Failed to update community database')
    }
  } catch (error) {
    console.error('Failed to add spot to community:', error)
    return { success: false, message: 'Failed to add spot to community database' }
  }
}

const calculateExposureAngles = () => {
  if (swellWindows.value.length === 0) {
    exposureAngles.value = null
    return
  }
  
  // Calculate exposure angles from swell windows
  const angles = []
  swellWindows.value.forEach(window => {
    const angle1 = calculateBearing(selectedSpot.value.coordinates, window.point1)
    const angle2 = calculateBearing(selectedSpot.value.coordinates, window.point2)
    
    // Ensure start is always the smaller angle for consistent calculations
    const startAngle = Math.min(angle1, angle2)
    const endAngle = Math.max(angle1, angle2)
    
    // Handle the case where the arc crosses 0° (e.g., 350° to 10°)
    const directDiff = endAngle - startAngle
    const wraparoundDiff = (startAngle + 360) - endAngle
    
    if (wraparoundDiff < directDiff) {
      // The shorter path crosses 0°, so swap start/end
      angles.push({ start: endAngle, end: startAngle + 360 })
    } else {
      angles.push({ start: startAngle, end: endAngle })
    }
  })
  
  exposureAngles.value = mergeAngles(angles)
}

const calculateBearing = (from, to) => {
  const lat1 = from.lat * Math.PI / 180
  const lat2 = to.lat * Math.PI / 180
  const deltaLng = (to.lng - from.lng) * Math.PI / 180
  
  const y = Math.sin(deltaLng) * Math.cos(lat2)
  const x = Math.cos(lat1) * Math.sin(lat2) - Math.sin(lat1) * Math.cos(lat2) * Math.cos(deltaLng)
  
  let bearing = Math.atan2(y, x) * 180 / Math.PI
  return (bearing + 360) % 360
}

const mergeAngles = (angles) => {
  // Sort and merge overlapping angle ranges
  const sorted = angles.sort((a, b) => a.start - b.start)
  const merged = []
  
  for (const angle of sorted) {
    if (merged.length === 0 || merged[merged.length - 1].end < angle.start) {
      merged.push(angle)
    } else {
      merged[merged.length - 1].end = Math.max(merged[merged.length - 1].end, angle.end)
    }
  }
  
  return merged
}
</script>

<template>
  <div id="app">
    <header class="header">
      <h1>Swell Window Calculator</h1>
      <p>Select a spot and draw swell windows to calculate exposure angles</p>
    </header>
    
    <main class="main-content">
      <div class="map-container">
        <SwellMap 
          @spot-selected="onSpotSelected" 
          :selected-spot="selectedSpot"
          :swell-windows="swellWindows"
          :community-spots="communitySpots"
          :is-loading-community="isLoadingCommunityData"
          @windows-updated="onWindowsUpdated"
          @load-community-spot="onConfigurationLoaded"
          @spot-completed="onSpotCompleted"
          @save-to-community="addSpotToCommunity"
        />
      </div>
      
      <div class="calculator-container" v-if="selectedSpot">
        <SwellWindowCalculator 
          :spot="selectedSpot" 
          :swell-windows="swellWindows"
          :exposure-angles="exposureAngles"
          @add-to-community="addSpotToCommunity"
        />
      </div>
      
      <div class="placeholder" v-else>
        <p>Click on the map to select a spot and start calculating swell windows</p>
      </div>
    </main>
  </div>
</template>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  background-color: #f0f8ff;
  margin: 0;
  padding: 0;
}

#app {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  margin: 0;
  padding: 0;
}

.header {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  padding: 1rem;
  text-align: center;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
}

.header h1 {
  font-size: 1.8rem;
  margin-bottom: 0.25rem;
  font-weight: 300;
}

.header p {
  font-size: 0.9rem;
  opacity: 0.9;
}

.main-content {
  flex: 1;
  display: grid;
  grid-template-columns: 2fr 1fr;
  gap: 1rem;
  padding: 0.5rem;
  width: 100vw;
  max-width: 100%;
}

.map-container {
  background: white;
  border-radius: 12px;
  box-shadow: 0 4px 20px rgba(0,0,0,0.1);
  overflow: hidden;
  min-height: 700px;
}

.analysis-container {
  background: white;
  border-radius: 12px;
  box-shadow: 0 4px 20px rgba(0,0,0,0.1);
  padding: 1.5rem;
}

.calculator-container {
  background: white;
  border-radius: 12px;
  box-shadow: 0 4px 20px rgba(0,0,0,0.1);
  padding: 1.5rem;
}

.placeholder {
  background: white;
  border-radius: 12px;
  box-shadow: 0 4px 20px rgba(0,0,0,0.1);
  padding: 1.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
  text-align: center;
  color: #666;
  font-style: italic;
}

@media (max-width: 768px) {
  .main-content {
    grid-template-columns: 1fr;
    padding: 0.25rem;
  }
  
  .header h1 {
    font-size: 1.5rem;
  }
  
  .header {
    padding: 0.75rem;
  }
}
</style>
