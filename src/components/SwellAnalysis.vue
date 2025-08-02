<script setup>
import { ref, computed, watch } from 'vue'

const props = defineProps({
  spot: {
    type: Object,
    required: true
  },
  swellData: {
    type: Object,
    required: true
  }
})

const exposureRating = computed(() => {
  if (!props.swellData) return 'Unknown'
  
  const height = props.swellData.swellHeight
  if (height < 1) return 'Poor'
  if (height < 2) return 'Fair' 
  if (height < 3) return 'Good'
  if (height < 4) return 'Very Good'
  return 'Excellent'
})

const exposureColor = computed(() => {
  switch (exposureRating.value) {
    case 'Poor': return '#ff4444'
    case 'Fair': return '#ff8800'
    case 'Good': return '#ffcc00'
    case 'Very Good': return '#88cc00'
    case 'Excellent': return '#00cc44'
    default: return '#666666'
  }
})

const swellDirectionDegrees = computed(() => {
  const directions = {
    'N': 0, 'NNE': 22.5, 'NE': 45, 'ENE': 67.5,
    'E': 90, 'ESE': 112.5, 'SE': 135, 'SSE': 157.5,
    'S': 180, 'SSW': 202.5, 'SW': 225, 'WSW': 247.5,
    'W': 270, 'WNW': 292.5, 'NW': 315, 'NNW': 337.5
  }
  return directions[props.swellData?.swellDirection] || 0
})

const formatCoordinates = (lat, lng) => {
  return `${lat.toFixed(4)}°, ${lng.toFixed(4)}°`
}

const getSwellAdvice = () => {
  const height = props.swellData?.swellHeight || 0
  const period = props.swellData?.swellPeriod || 0
  
  if (height < 1) return "Small swell conditions. Consider other activities or wait for better conditions."
  if (height > 4) return "Large swell conditions. Exercise caution and consider experience level."
  if (period < 8) return "Short period swell. Conditions may be choppy and less organized."
  if (period > 15) return "Long period swell. Expect powerful, well-organized waves."
  return "Moderate conditions. Good for most skill levels with proper safety precautions."
}
</script>

<template>
  <div class="swell-analysis">
    <header class="analysis-header">
      <h2>🌊 Swell Analysis</h2>
      <div class="spot-info">
        <h3>{{ spot.name }}</h3>
        <p class="coordinates">
          {{ formatCoordinates(spot.coordinates.lat, spot.coordinates.lng) }}
        </p>
      </div>
    </header>

    <div class="metrics-grid">
      <div class="metric-card">
        <div class="metric-icon">📏</div>
        <div class="metric-content">
          <h4>Swell Height</h4>
          <div class="metric-value">{{ swellData.swellHeight }}m</div>
        </div>
      </div>

      <div class="metric-card">
        <div class="metric-icon">⏱️</div>
        <div class="metric-content">
          <h4>Swell Period</h4>
          <div class="metric-value">{{ swellData.swellPeriod }}s</div>
        </div>
      </div>

      <div class="metric-card">
        <div class="metric-icon">🧭</div>
        <div class="metric-content">
          <h4>Direction</h4>
          <div class="metric-value">{{ swellData.swellDirection }}</div>
          <div class="compass" :style="{ transform: `rotate(${swellDirectionDegrees}deg)` }">
            ↑
          </div>
        </div>
      </div>

      <div class="metric-card exposure-card">
        <div class="metric-icon">⭐</div>
        <div class="metric-content">
          <h4>Exposure Rating</h4>
          <div 
            class="metric-value exposure-rating" 
            :style="{ color: exposureColor }"
          >
            {{ exposureRating }}
          </div>
        </div>
      </div>
    </div>

    <div class="analysis-details">
      <h4>📊 Analysis Details</h4>
      <div class="details-content">
        <div class="detail-item">
          <strong>Spot Quality:</strong> 
          <span class="quality-indicator" :style="{ backgroundColor: exposureColor }"></span>
          {{ exposureRating }}
        </div>
        <div class="detail-item">
          <strong>Last Updated:</strong> {{ new Date(spot.timestamp).toLocaleString() }}
        </div>
      </div>
    </div>

    <div class="recommendations">
      <h4>💡 Recommendations</h4>
      <p class="advice">{{ getSwellAdvice() }}</p>
    </div>

    <div class="actions">
      <button class="action-btn primary">
        📱 Get Alerts
      </button>
      <button class="action-btn secondary">
        📈 View History
      </button>
    </div>
  </div>
</template>

<style scoped>
.swell-analysis {
  height: 100%;
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

.analysis-header {
  text-align: center;
  padding-bottom: 1rem;
  border-bottom: 2px solid #f0f0f0;
}

.analysis-header h2 {
  color: #333;
  margin-bottom: 1rem;
  font-size: 1.5rem;
}

.spot-info h3 {
  color: #555;
  margin-bottom: 0.5rem;
}

.coordinates {
  color: #888;
  font-size: 0.9rem;
  font-family: monospace;
}

.metrics-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
}

.metric-card {
  background: linear-gradient(135deg, #f8f9fa 0%, #e9ecef 100%);
  border-radius: 8px;
  padding: 1rem;
  text-align: center;
  border: 1px solid #dee2e6;
  transition: transform 0.2s ease;
}

.metric-card:hover {
  transform: translateY(-2px);
}

.exposure-card {
  background: linear-gradient(135deg, #fff3cd 0%, #ffeaa7 100%);
  border-color: #f39c12;
}

.metric-icon {
  font-size: 1.5rem;
  margin-bottom: 0.5rem;
}

.metric-content h4 {
  color: #555;
  font-size: 0.9rem;
  margin-bottom: 0.5rem;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.metric-value {
  font-size: 1.5rem;
  font-weight: bold;
  color: #333;
}

.exposure-rating {
  font-size: 1.2rem;
  text-transform: uppercase;
  letter-spacing: 1px;
}

.compass {
  font-size: 1.2rem;
  margin-top: 0.5rem;
  transition: transform 0.3s ease;
}

.analysis-details, .recommendations {
  background: #f8f9fa;
  padding: 1rem;
  border-radius: 8px;
  border-left: 4px solid #007bff;
}

.analysis-details h4, .recommendations h4 {
  color: #333;
  margin-bottom: 0.5rem;
  font-size: 1rem;
}

.details-content {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.detail-item {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 0.9rem;
}

.quality-indicator {
  width: 12px;
  height: 12px;
  border-radius: 50%;
  border: 1px solid rgba(255,255,255,0.3);
}

.advice {
  color: #555;
  line-height: 1.5;
  font-size: 0.9rem;
  margin: 0;
}

.actions {
  display: flex;
  gap: 0.5rem;
  margin-top: auto;
}

.action-btn {
  flex: 1;
  padding: 0.75rem;
  border: none;
  border-radius: 6px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;
  font-size: 0.9rem;
}

.action-btn.primary {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
}

.action-btn.secondary {
  background: #e9ecef;
  color: #495057;
  border: 1px solid #ced4da;
}

.action-btn:hover {
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(0,0,0,0.15);
}

.action-btn.primary:hover {
  background: linear-gradient(135deg, #5a6fd8 0%, #6a4190 100%);
}

.action-btn.secondary:hover {
  background: #dee2e6;
}
</style>
