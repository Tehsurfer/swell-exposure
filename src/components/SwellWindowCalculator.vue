<template>
  <div class="swell-calculator">
    <div class="calculator-header">
      <h2>📐 Swell Window Calculator</h2>
      <p>Analyzing swell exposure for: <strong>{{ spot.name }}</strong></p>
    </div>

    <div class="calculator-summary" v-if="swellWindows.length > 0">
      <div class="compass-container">
        <h3>🧭 Swell Exposure</h3>
        
        <!-- Simple compass circle -->
        <div class="simple-compass">
          <svg class="compass-svg" viewBox="0 0 200 200">
            <!-- Background circle -->
            <circle cx="100" cy="100" r="80" fill="rgba(255, 255, 255, 0.1)" stroke="rgba(255, 255, 255, 0.3)" stroke-width="2"/>
            
            <!-- Cardinal directions -->
            <text x="100" y="25" text-anchor="middle" class="direction-label">N</text>
            <text x="175" y="105" text-anchor="middle" class="direction-label">E</text>
            <text x="100" y="185" text-anchor="middle" class="direction-label">S</text>
            <text x="25" y="105" text-anchor="middle" class="direction-label">W</text>
            
            <!-- Exposure arcs -->
            <g class="exposure-arcs">
              <path 
                v-for="(range, index) in exposureAngles" 
                :key="index"
                :d="createSimpleArc(range.start, range.end)"
                fill="rgba(255, 255, 255, 0.3)" 
                stroke="rgba(255, 255, 255, 0.9)"
                stroke-width="1"
                opacity="0.8"
              />
            </g>
            
            <!-- Center dot -->
            <circle cx="100" cy="100" r="4" fill="rgba(255, 255, 255, 0.8)"/>
          </svg>
        </div>
        
        <!-- Exposure ranges list -->
        <div class="exposure-list">
          <div 
            v-for="(range, index) in exposureAngles" 
            :key="index"
            class="exposure-item"
          >
            {{ getDirectionFromAngle(range.start) }} {{ Math.round(range.start) }}° → {{ getDirectionFromAngle(range.end) }} {{ Math.round(range.end) }}°
          </div>
        </div>
        
        <!-- Summary -->
        <div class="exposure-total">
          <strong>{{ Math.round(totalExposureRange) }}°</strong> total exposure • <span :style="{ color: exposureQuality.color }"><strong>{{ exposureQuality.text }}</strong></span>
        </div>
      </div>
    </div>

    
    <div class="status-message" v-if="saveStatus">
      {{ saveStatus }}
    </div>

    <div class="actions">
      <button 
        @click="saveConfiguration" 
        class="action-btn primary"
        :disabled="isLoading || swellWindows.length === 0"
      >
        {{ isLoading ? '⏳ Adding...' : '📂 Add to Community' }}
      </button>
      <button 
        @click="exportAsJson" 
        class="action-btn secondary"
      >
        📊 Export JSON
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

const props = defineProps({
  spot: {
    type: Object,
    required: true
  },
  swellWindows: {
    type: Array,
    default: () => []
  },
  exposureAngles: {
    type: Array,
    default: () => []
  }
})

const emit = defineEmits(['add-to-community'])
const saveStatus = ref('')
const isLoading = ref(false)

const formatCoordinates = (lat, lng) => {
  return `${lat.toFixed(4)}°, ${lng.toFixed(4)}°`
}

const formatAngle = (angle) => {
  return `${Math.round(angle)}°`
}

const getDirectionFromAngle = (angle) => {
  const directions = ['N', 'NNE', 'NE', 'ENE', 'E', 'ESE', 'SE', 'SSE', 'S', 'SSW', 'SW', 'WSW', 'W', 'WNW', 'NW', 'NNW']
  const index = Math.round(angle / 22.5) % 16
  return directions[index]
}

const createSimpleArc = (startAngle, endAngle) => {
  const centerX = 100
  const centerY = 100
  const radius = 80
  
  // Convert angles to radians and adjust for SVG coordinate system (0° = top)
  const startRad = (startAngle - 90) * Math.PI / 180
  const endRad = (endAngle - 90) * Math.PI / 180
  
  // Calculate start and end points
  const startX = centerX + radius * Math.cos(startRad)
  const startY = centerY + radius * Math.sin(startRad)
  const endX = centerX + radius * Math.cos(endRad)
  const endY = centerY + radius * Math.sin(endRad)
  
  // Determine if arc should be large (> 180 degrees)
  const angleDiff = endAngle - startAngle
  const largeArcFlag = angleDiff > 180 ? 1 : 0
  
  // Create triangular sector path: move to center, line to start, arc to end, close back to center
  return `M ${centerX} ${centerY} L ${startX} ${startY} A ${radius} ${radius} 0 ${largeArcFlag} 1 ${endX} ${endY} Z`
}

const totalExposureRange = computed(() => {
  if (!props.exposureAngles || props.exposureAngles.length === 0) return 0
  
  return props.exposureAngles.reduce((total, range) => {
    let rangeSize = range.end - range.start
    if (rangeSize < 0) rangeSize += 360 // Handle wraparound
    return total + rangeSize
  }, 0)
})

const exposureQuality = computed(() => {
  const range = totalExposureRange.value
  if (range >= 180) return { text: 'Excellent', color: '#00cc44' }
  if (range >= 120) return { text: 'Very Good', color: '#88cc00' }
  if (range >= 90) return { text: 'Good', color: '#ffcc00' }
  if (range >= 60) return { text: 'Fair', color: '#ff8800' }
  return { text: 'Limited', color: '#ff4444' }
})

const saveConfiguration = async () => {
  if (props.swellWindows.length === 0) {
    saveStatus.value = '⚠️ Please add at least one swell window before saving.'
    setTimeout(() => saveStatus.value = '', 3000)
    return
  }

  isLoading.value = true
  saveStatus.value = 'Saving spot to community database...'
  
  try {
    const config = {
      spot: props.spot,
      swellWindows: props.swellWindows,
      exposureAngles: props.exposureAngles,
      timestamp: new Date().toISOString(),
      version: '1.0'
    }
    
    const result = await emit('add-to-community', config)
    
    if (result && result.success) {
      saveStatus.value = result.message
    } else {
      saveStatus.value = '✅ Spot saved to community database!'
    }
    
  } catch (error) {
    console.error('Save error:', error)
    saveStatus.value = '❌ Failed to save spot to community database.'
  }
  
  isLoading.value = false
  setTimeout(() => saveStatus.value = '', 5000)
}

</script>

<style scoped>
.swell-calculator {
  height: 100%;
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

.calculator-header {
  text-align: center;
  padding-bottom: 1rem;
  border-bottom: 2px solid #e1e5e9;
}

.calculator-header h2 {
  margin: 0 0 0.5rem 0;
  color: #2c3e50;
  font-size: 1.5rem;
}

.calculator-header p {
  margin: 0;
  color: #666;
  font-size: 0.95rem;
}

.calculator-summary {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  padding: 2rem;
  border-radius: 12px;
  margin-bottom: 1rem;
}

.compass-container {
  text-align: center;
}

.compass-container h3 {
  margin: 0 0 1.5rem 0;
  font-size: 1.4rem;
}

.simple-compass {
  display: flex;
  justify-content: center;
  margin-bottom: 1.5rem;
}

.compass-svg {
  width: 200px;
  height: 200px;
  max-width: 100%;
}

.direction-label {
  font-size: 16px;
  font-weight: bold;
  fill: rgba(255, 255, 255, 0.9);
}

.exposure-list {
  margin-bottom: 1rem;
}

.exposure-item {
  background: rgba(255, 255, 255, 0.15);
  padding: 0.75rem;
  margin-bottom: 0.5rem;
  border-radius: 6px;
  font-weight: 500;
  font-size: 1rem;
  border-left: 4px solid rgba(255, 255, 255, 0.6);
}

.exposure-item:last-child {
  margin-bottom: 0;
}

.exposure-total {
  background: rgba(255, 255, 255, 0.1);
  padding: 1rem;
  border-radius: 8px;
  font-size: 1rem;
}

.exposure-details {
  background: #f8f9fa;
  padding: 1.5rem;
  border-radius: 8px;
  border: 1px solid #e9ecef;
}

.exposure-details h4 {
  margin: 0 0 1rem 0;
  color: #2c3e50;
  font-size: 1.1rem;
}

.angles-grid {
  display: grid;
  gap: 1rem;
}

.angle-range {
  background: white;
  border: 1px solid #dee2e6;
  border-radius: 6px;
  padding: 1rem;
}

.range-header {
  margin-bottom: 0.75rem;
}

.window-badge {
  background: #007bff;
  color: white;
  padding: 0.25rem 0.5rem;
  border-radius: 12px;
  font-size: 0.75rem;
  font-weight: 500;
}

.range-details {
  display: flex;
  align-items: center;
  gap: 1rem;
  flex-wrap: wrap;
}

.angle-info {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.25rem;
}

.angle-value {
  font-weight: bold;
  color: #2c3e50;
  font-size: 1.1rem;
}

.direction {
  font-size: 0.85rem;
  color: #6c757d;
  background: #f8f9fa;
  padding: 0.125rem 0.375rem;
  border-radius: 4px;
}

.range-separator {
  color: #007bff;
  font-weight: bold;
  font-size: 1.2rem;
}

.range-size {
  color: #28a745;
  font-weight: 500;
  background: #e8f5e8;
  padding: 0.25rem 0.5rem;
  border-radius: 4px;
  font-size: 0.9rem;
  margin-left: auto;
}

.calculator-details {
  background: #f8f9fa;
  padding: 1.5rem;
  border-radius: 8px;
}

.calculator-details h4 {
  margin: 0 0 1rem 0;
  color: #2c3e50;
}

.windows-list {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.window-item {
  display: flex;
  align-items: center;
  gap: 1rem;
  background: white;
  padding: 1rem;
  border-radius: 6px;
  border: 1px solid #dee2e6;
}

.window-number {
  background: #007bff;
  color: white;
  width: 2rem;
  height: 2rem;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  font-size: 0.9rem;
}

.window-info {
  flex: 1;
}

.window-points {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  font-size: 0.9rem;
  color: #6c757d;
}

.recommendations {
  background: #e8f4f8;
  padding: 1.5rem;
  border-radius: 8px;
  border-left: 4px solid #17a2b8;
}

.recommendations h4 {
  margin: 0 0 1rem 0;
  color: #0c5460;
}

.advice {
  margin: 0;
  color: #0c5460;
  line-height: 1.5;
}

.sharing-section {
  background: #f8f9fa;
  padding: 1.5rem;
  border-radius: 8px;
  border: 1px solid #e9ecef;
}

.sharing-section h4 {
  margin: 0 0 1rem 0;
  color: #2c3e50;
}

.sharing-controls {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.status-message {
  padding: 0.75rem;
  border-radius: 4px;
  font-weight: 500;
  text-align: center;
  background: #d4edda;
  color: #155724;
  border: 1px solid #c3e6cb;
}

.actions {
  display: flex;
  gap: 1rem;
  padding-top: 1rem;
  border-top: 1px solid #e9ecef;
}

.action-btn {
  flex: 1;
  padding: 0.75rem 1.5rem;
  border: none;
  border-radius: 6px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
}

.action-btn.primary {
  background: #28a745;
  color: white;
}

.action-btn.primary:hover:not(:disabled) {
  background: #218838;
  transform: translateY(-1px);
}

.action-btn.secondary {
  background: #6c757d;
  color: white;
}

.action-btn.secondary:hover {
  background: #5a6268;
  transform: translateY(-1px);
}

.action-btn:disabled {
  background: #e9ecef;
  color: #6c757d;
  cursor: not-allowed;
  transform: none;
}

@media (max-width: 768px) {
  .calculator-summary {
    padding: 1.5rem;
  }
  
  .compass-container h3 {
    font-size: 1.2rem;
    margin-bottom: 1rem;
  }
  
  .compass-svg {
    width: 160px;
    height: 160px;
  }
  
  .exposure-item {
    font-size: 0.9rem;
    padding: 0.6rem;
  }
  
  .exposure-total {
    font-size: 0.9rem;
  }
  
  .range-details {
    flex-direction: column;
    align-items: flex-start;
    gap: 0.5rem;
  }
  
  .range-size {
    margin-left: 0;
    align-self: flex-start;
  }
  
  .actions {
    flex-direction: column;
  }
}
</style>
