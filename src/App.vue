<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue'

// ── Reactive state ──────────────────────────────────────────────
const latitude = ref(null)
const longitude = ref(null)
const lastUpdated = ref(null)
const error = ref(null)
const isLoading = ref(true)
const history = ref([])           // keeps last few positions to show a trail

let refreshTimer = null

// ── Fetch the ISS's current position ────────────────────────────
async function fetchISSLocation() {
  try {
    const response = await fetch('http://api.open-notify.org/iss-now.json')
    if (!response.ok) throw new Error('Network response was not ok')

    const data = await response.json()

    latitude.value = parseFloat(data.iss_position.latitude)
    longitude.value = parseFloat(data.iss_position.longitude)
    lastUpdated.value = new Date(data.timestamp * 1000).toLocaleTimeString()
    error.value = null

    // Keep a short trail of recent positions (max 5)
    history.value.push({ lat: latitude.value, lon: longitude.value })
    if (history.value.length > 5) history.value.shift()
  } catch (err) {
    error.value = 'Could not reach the ISS tracking API. Check your connection and try again.'
  } finally {
    isLoading.value = false
  }
}

// ── Convert lat/long into x/y position on our flat map ─────────
const markerPosition = computed(() => {
  if (latitude.value === null || longitude.value === null) return { x: 0, y: 0 }
  const x = ((longitude.value + 180) / 360) * 100
  const y = ((90 - latitude.value) / 180) * 100
  return { x, y }
})

// ── Lifecycle: fetch on load, then every 5 seconds ──────────────
onMounted(() => {
  fetchISSLocation()
  refreshTimer = setInterval(fetchISSLocation, 5000)
})

onUnmounted(() => {
  clearInterval(refreshTimer)
})
</script>

<template>
  <div class="iss-tracker">
    <header class="tracker-header">
      <h1>Where is the ISS right now?</h1>
      <p class="subtitle">Live position of the International Space Station, updated every 5 seconds.</p>
    </header>

    <div class="map-container">
      <div class="map-bg">
        <div
          v-if="!isLoading && !error"
          class="iss-marker"
          :style="{ left: markerPosition.x + '%', top: markerPosition.y + '%' }"
        >
          <span class="marker-dot"></span>
          <span class="marker-label">ISS</span>
        </div>

        <div
          v-for="(point, index) in history.slice(0, -1)"
          :key="index"
          class="trail-dot"
          :style="{
            left: ((point.lon + 180) / 360 * 100) + '%',
            top: ((90 - point.lat) / 180 * 100) + '%',
            opacity: 0.15 + (index * 0.15)
          }"
        ></div>
      </div>
    </div>

    <div class="info-panel">
      <div v-if="isLoading" class="status loading">Fetching ISS location…</div>

      <div v-else-if="error" class="status error">
        {{ error }}
        <button @click="fetchISSLocation" class="retry-btn">Try again</button>
      </div>

      <div v-else class="status success">
        <div class="coord-grid">
          <div class="coord-item">
            <span class="coord-label">Latitude</span>
            <span class="coord-value">{{ latitude.toFixed(4) }}°</span>
          </div>
          <div class="coord-item">
            <span class="coord-label">Longitude</span>
            <span class="coord-value">{{ longitude.toFixed(4) }}°</span>
          </div>
        </div>
        <p class="updated-text">Last updated: {{ lastUpdated }}</p>
        <button @click="fetchISSLocation" class="refresh-btn">Refresh now</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.iss-tracker {
  max-width: 640px;
  margin: 0 auto;
  padding: 24px;
  font-family: 'Segoe UI', system-ui, sans-serif;
  color: #e8eef5;
}

.tracker-header {
  text-align: center;
  margin-bottom: 24px;
}

.tracker-header h1 {
  font-size: 1.6rem;
  margin: 0 0 6px 0;
  color: #ffffff;
}

.subtitle {
  font-size: 0.9rem;
  color: #9aa9bd;
  margin: 0;
}

.map-container {
  position: relative;
  width: 100%;
  padding-top: 50%;
  border-radius: 12px;
  overflow: hidden;
  margin-bottom: 20px;
  border: 1px solid #2a3950;
}

.map-bg {
  position: absolute;
  inset: 0;
  background: linear-gradient(180deg, #0b1622 0%, #102232 50%, #0b1622 100%);
}

.iss-marker {
  position: absolute;
  transform: translate(-50%, -50%);
  display: flex;
  flex-direction: column;
  align-items: center;
  transition: left 0.6s ease, top 0.6s ease;
}

.marker-dot {
  width: 14px;
  height: 14px;
  background: #ffcc66;
  border-radius: 50%;
  box-shadow: 0 0 0 4px rgba(255, 204, 102, 0.25), 0 0 12px rgba(255, 204, 102, 0.6);
  animation: pulse 2s infinite;
}

.marker-label {
  font-size: 0.7rem;
  margin-top: 4px;
  background: rgba(0,0,0,0.5);
  padding: 1px 6px;
  border-radius: 4px;
  white-space: nowrap;
}

.trail-dot {
  position: absolute;
  width: 6px;
  height: 6px;
  background: #ffcc66;
  border-radius: 50%;
  transform: translate(-50%, -50%);
}

@keyframes pulse {
  0%, 100% { box-shadow: 0 0 0 4px rgba(255, 204, 102, 0.25), 0 0 12px rgba(255, 204, 102, 0.6); }
  50% { box-shadow: 0 0 0 8px rgba(255, 204, 102, 0.15), 0 0 18px rgba(255, 204, 102, 0.8); }
}

.info-panel {
  background: #131f30;
  border-radius: 12px;
  padding: 20px;
  border: 1px solid #2a3950;
}

.status.loading {
  text-align: center;
  color: #9aa9bd;
}

.status.error {
  text-align: center;
  color: #ff8a8a;
}

.coord-grid {
  display: flex;
  justify-content: center;
  gap: 40px;
  margin-bottom: 12px;
}

.coord-item {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.coord-label {
  font-size: 0.75rem;
  color: #9aa9bd;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.coord-value {
  font-size: 1.4rem;
  font-weight: 600;
  color: #ffcc66;
}

.updated-text {
  text-align: center;
  font-size: 0.8rem;
  color: #6b7d94;
  margin: 8px 0 16px 0;
}

.refresh-btn, .retry-btn {
  display: block;
  margin: 0 auto;
  background: #ffcc66;
  color: #1a1308;
  border: none;
  padding: 8px 20px;
  border-radius: 8px;
  font-weight: 600;
  cursor: pointer;
}

.refresh-btn:hover, .retry-btn:hover {
  background: #ffd98a;
}
</style>