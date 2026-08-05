<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'


const METRIC_TYPES = [
  { key: 'cpu', label: 'CPU Utilization', unit: '%', color: '#38bdf8', max: 100 },
  { key: 'memory', label: ' Memory Heap', unit: 'MB', color: '#a78bfa', max: 1024 },
  { key: 'latency', label: ' Network Latency', unit: 'ms', color: '#f43f5e', max: 500 }
]


const isStreaming = ref(true)
const activeMetric = ref('cpu')
const bufferWindowSize = ref(20) 
const anomalyThreshold = ref(85) 


const timeSeriesData = ref({
  cpu: [],
  memory: [],
  latency: []
})


const telemetryLogs = ref([
  ' DevStream Engine initialized. Time-series telemetry pipeline active.'
])

let streamInterval = null


const selectedMetricConfig = computed(() => {
  return METRIC_TYPES.find(m => m.key === activeMetric.value) || METRIC_TYPES[0]
})


const activeStreamBuffer = computed(() => {
  return timeSeriesData.value[activeMetric.value] || []
})


const averageMetricValue = computed(() => {
  const buffer = activeStreamBuffer.value
  if (buffer.length === 0) return 0
  const sum = buffer.reduce((acc, point) => acc + point.value, 0)
  return Math.round(sum / buffer.length)
})


const anomalyCount = computed(() => {
  return activeStreamBuffer.value.filter(point => point.isAnomaly).length
})


const generateTelemetryPacket = () => {
  const timestamp = new Date().toLocaleTimeString()

  METRIC_TYPES.forEach(metric => {
    
    const isSpike = Math.random() < 0.12 
    let baseVal = 0

    if (metric.key === 'cpu') baseVal = isSpike ? 92 : Math.floor(Math.random() * 40) + 20
    else if (metric.key === 'memory') baseVal = isSpike ? 950 : Math.floor(Math.random() * 300) + 400
    else if (metric.key === 'latency') baseVal = isSpike ? 460 : Math.floor(Math.random() * 60) + 15

    const normalizedPercentage = Math.min(100, Math.round((baseVal / metric.max) * 100))
    const isAnomaly = normalizedPercentage >= anomalyThreshold.value

    const packet = {
      id: `p-${Date.now()}-${Math.random().toString().slice(-3)}`,
      timestamp,
      value: baseVal,
      percentage: normalizedPercentage,
      isAnomaly
    }

    // Push packet to array and enforce sliding window memory constraint
    const currentBuffer = timeSeriesData.value[metric.key]
    timeSeriesData.value[metric.key] = [...currentBuffer, packet].slice(-bufferWindowSize.value)

    if (isAnomaly && metric.key === activeMetric.value) {
      telemetryLogs.value.unshift(
        ` ANOMALY ALERT: High ${metric.label} detected at ${timestamp}! Value: ${baseVal}${metric.unit} (${normalizedPercentage}%)`
      )
    }
  })
}

// Stream Control Operations
const toggleStreaming = () => {
  isStreaming.value = !isStreaming.value
  if (isStreaming.value) {
    startStream()
    telemetryLogs.value.unshift('▶ Telemetry stream resumed.')
  } else {
    stopStream()
    telemetryLogs.value.unshift('⏸ Telemetry stream paused.')
  }
}

const clearBuffers = () => {
  timeSeriesData.value = { cpu: [], memory: [], latency: [] }
  telemetryLogs.value.unshift(' Time-series memory buffers cleared.')
}

const startStream = () => {
  stopStream()
  streamInterval = setInterval(generateTelemetryPacket, 1000)
}

const stopStream = () => {
  if (streamInterval) clearInterval(streamInterval)
}

// Vue Lifecycle Hooks
onMounted(() => {
  startStream()
})

onUnmounted(() => {
  stopStream()
})
</script>

<template>
  <div class="studio-container">
    
    <!-- HUD HEADER CONTROL STRIP -->
    <header class="hud-header">
      <div>
        <h1 class="brand-title"> DevStream Time-Series Telemetry Profiler</h1>
        <p class="brand-subtitle">
          Vue 3 Composition API real-time metrics buffering, memory windowing, and anomaly detection engine.
        </p>
      </div>

      <div class="header-actions">
        <button 
          class="btn" 
          :class="isStreaming ? 'btn-pause' : 'btn-resume'"
          @click="toggleStreaming"
        >
          {{ isStreaming ? '⏸ Pause Stream' : '▶ Resume Stream' }}
        </button>
        <button class="btn btn-clear" @click="clearBuffers">🧹 Clear Buffers</button>
      </div>
    </header>

   
    <div class="stats-overview-grid">
      <div 
        v-for="metric in METRIC_TYPES" 
        :key="metric.key"
        class="metric-tab-card"
        :class="{ 'is-active': activeMetric === metric.key }"
        @click="activeMetric = metric.key"
      >
        <div class="metric-tab-header">
          <span class="metric-label">{{ metric.label }}</span>
          <span class="metric-indicator" :style="{ backgroundColor: metric.color }"></span>
        </div>
        <div class="metric-tab-body">
          <span class="current-value">
            {{ timeSeriesData[metric.key].slice(-1)[0]?.value || 0 }} {{ metric.unit }}
          </span>
          <span class="buffer-count">
            Window: {{ timeSeriesData[metric.key].length }}/{{ bufferWindowSize }}
          </span>
        </div>
      </div>
    </div>

   
    <div class="workspace-grid">
      
     
      <main class="monitor-card">
        <div class="card-header">
          <h3>Real-Time Stream Monitor: {{ selectedMetricConfig.label }}</h3>
          <span class="moving-avg">Moving Avg: {{ averageMetricValue }} {{ selectedMetricConfig.unit }}</span>
        </div>

        
        <div class="graph-viewport">
          <div v-if="activeStreamBuffer.length > 0" class="bars-container">
            <div 
              v-for="point in activeStreamBuffer" 
              :key="point.id" 
              class="bar-wrapper"
            >
              <div class="bar-value-label">{{ point.value }}</div>
              <div class="bar-track">
                <div 
                  class="bar-fill" 
                  :class="{ 'is-anomaly': point.isAnomaly }"
                  :style="{ 
                    height: `${point.percentage}%`, 
                    backgroundColor: point.isAnomaly ? '#f43f5e' : selectedMetricConfig.color 
                  }"
                ></div>
              </div>
              <div class="bar-time-label">{{ point.timestamp.split(':')[2] }}s</div>
            </div>
          </div>

          <div v-else class="empty-graph">
           Buffer empty. Resuming telemetry ingestion pipeline...
          </div>
        </div>
      </main>

     
      <aside class="controls-column">
        
        
        <section class="control-card">
          <h3>Buffer Window Parameters</h3>
          
          <div class="control-group">
            <label>Sliding Window Capacity ({{ bufferWindowSize }} Points)</label>
            <input 
              v-model.number="bufferWindowSize" 
              type="range" 
              min="10" 
              max="40" 
              step="5" 
              class="range-slider"
            />
          </div>

          <div class="control-group">
            <label>Anomaly Threshold Trigger ({{ anomalyThreshold }}%)</label>
            <input 
              v-model.number="anomalyThreshold" 
              type="range" 
              min="50" 
              max="98" 
              step="2" 
              class="range-slider"
            />
          </div>
        </section>

       
        <section class="control-card">
          <h3>Anomaly Detection HUD</h3>
          <div class="anomaly-hud">
            <div class="hud-stat">
              <span class="stat-title">Flagged Anomalies</span>
              <span class="stat-value" :class="{ 'has-anomalies': anomalyCount > 0 }">
                {{ anomalyCount }}
              </span>
            </div>
            <div class="hud-stat">
              <span class="stat-title">Status</span>
              <span class="stat-status" :class="anomalyCount > 0 ? 'status-warn' : 'status-ok'">
                {{ anomalyCount > 0 ? ' ATTENTION NEEDED' : ' STREAM NOMINAL' }}
              </span>
            </div>
          </div>
        </section>

      </aside>

    </div>

    <!-- BOTTOM DECK: REAL-TIME SYSTEM LOGS -->
    <footer class="log-card">
      <h3>Telemetry Stream Ingestion Logs</h3>
      <div class="log-terminal">
        <div 
          v-for="(log, idx) in telemetryLogs" 
          :key="idx" 
          class="log-line"
          :class="{ 'is-alert': log.includes('🚨') }"
        >
          {{ log }}
        </div>
      </div>
    </footer>

  </div>
</template>

<style>
/* Clean, modern dark telemetry dashboard styling */
body {
  margin: 0;
  background-color: #070a13;
  color: #f8fafc;
  font-family: monospace;
}

.studio-container {
  max-width: 1300px;
  margin: 30px auto;
  padding: 0 24px;
}

.hud-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid #1e293b;
  padding-bottom: 20px;
  margin-bottom: 25px;
}

.brand-title {
  margin: 0;
  font-size: 24px;
  color: #a78bfa;
}

.brand-subtitle {
  margin: 4px 0 0 0;
  color: #475569;
  font-size: 12px;
}

.header-actions {
  display: flex;
  gap: 12px;
}

.btn {
  padding: 8px 16px;
  border: none;
  border-radius: 6px;
  font-size: 12px;
  font-weight: bold;
  cursor: pointer;
  font-family: monospace;
}

.btn-pause { background-color: #f43f5e; color: #fff; }
.btn-resume { background-color: #10b981; color: #070a13; }
.btn-clear { background-color: #1e293b; color: #cbd5e1; border: 1px solid #334155; }

.stats-overview-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 20px;
  margin-bottom: 25px;
}

.metric-tab-card {
  background-color: #0f172a;
  border: 1px solid #1e293b;
  border-radius: 12px;
  padding: 16px;
  cursor: pointer;
  transition: all 0.2s;
}

.metric-tab-card.is-active {
  border-color: #a78bfa;
  background-color: #1e1b4b;
}

.metric-tab-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}

.metric-label {
  font-size: 12px;
  color: #cbd5e1;
  font-weight: bold;
}

.metric-indicator {
  width: 8px;
  height: 8px;
  border-radius: 50%;
}

.metric-tab-body {
  display: flex;
  justify-content: space-between;
  align-items: flex-end;
}

.current-value {
  font-size: 20px;
  font-weight: bold;
  color: #fff;
}

.buffer-count {
  font-size: 10px;
  color: #64748b;
}

.workspace-grid {
  display: grid;
  grid-template-columns: 1fr 320px;
  gap: 25px;
  margin-bottom: 25px;
}

.monitor-card, .control-card, .log-card {
  background-color: #0f172a;
  border: 1px solid #1e293b;
  border-radius: 14px;
  padding: 20px;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15px;
}

.card-header h3, .control-card h3, .log-card h3 {
  margin: 0;
  font-size: 12px;
  color: #64748b;
  text-transform: uppercase;
}

.moving-avg {
  font-size: 11px;
  color: #a78bfa;
  font-weight: bold;
}

.graph-viewport {
  height: 280px;
  background-color: #020617;
  border: 1px dashed #334155;
  border-radius: 10px;
  padding: 15px;
  display: flex;
  align-items: flex-end;
}

.bars-container {
  display: flex;
  width: 100%;
  height: 100%;
  align-items: flex-end;
  gap: 8px;
}

.bar-wrapper {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  height: 100%;
}

.bar-value-label {
  font-size: 8px;
  color: #64748b;
  margin-bottom: 4px;
}

.bar-track {
  flex-grow: 1;
  width: 100%;
  background-color: #070a13;
  border-radius: 4px;
  display: flex;
  align-items: flex-end;
  overflow: hidden;
}

.bar-fill {
  width: 100%;
  border-radius: 4px 4px 0 0;
  transition: height 0.3s ease, background-color 0.2s;
}

.bar-fill.is-anomaly {
  box-shadow: 0 0 8px #f43f5e;
}

.bar-time-label {
  font-size: 8px;
  color: #475569;
  margin-top: 4px;
}

.empty-graph {
  width: 100%;
  text-align: center;
  color: #475569;
  font-size: 12px;
  align-self: center;
}

.controls-column {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.control-group {
  margin-top: 15px;
}

.control-group label {
  display: block;
  font-size: 11px;
  color: #cbd5e1;
  margin-bottom: 8px;
}

.range-slider {
  width: 100%;
  accent-color: #a78bfa;
  cursor: pointer;
}

.anomaly-hud {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-top: 12px;
}

.hud-stat {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background-color: #070a13;
  padding: 10px 14px;
  border-radius: 8px;
  border: 1px solid #1e293b;
}

.stat-title {
  font-size: 11px;
  color: #64748b;
}

.stat-value {
  font-size: 16px;
  font-weight: bold;
  color: #10b981;
}

.stat-value.has-anomalies {
  color: #f43f5e;
}

.stat-status {
  font-size: 10px;
  font-weight: bold;
}

.status-ok { color: #10b981; }
.status-warn { color: #f43f5e; }

.log-terminal {
  background-color: #070a13;
  border-radius: 8px;
  padding: 12px;
  height: 110px;
  overflow-y: auto;
  margin-top: 10px;
}

.log-line {
  font-size: 11px;
  color: #64748b;
  margin-bottom: 4px;
}

.log-line.is-alert {
  color: #f43f5e;
  font-weight: bold;
}
</style>