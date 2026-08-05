#  DevStream — Time-Series Telemetry & Memory Profiler Engine (Vue 3)

DevStream is a time-series telemetry and observability laboratory engineered with Vue 3 (Composition API `<script setup>`). It simulates streaming infrastructure metrics (CPU load, memory heap, network latency), manages memory buffers via sliding window arrays (`slice()`), detects metric anomaly spikes in real-time, and renders animated stream visualizations.

##  Technical Architecture Overview
*  **Sliding Window Buffering:** Implements array memory windowing techniques to maintain a fixed time-series capacity (`.slice(-windowSize)`), preventing memory leaks during continuous streaming.
*  **Real-Time Anomaly Analysis:** Computes moving averages and evaluates dynamic percentage thresholds to instantly flag telemetry spikes down to logging terminals.
*  **Vue Lifecycle Streams:** Utilizes Vue 3 `onMounted` and `onUnmounted` lifecycle hooks to manage timer interval memory allocations safely.

##  Running Instructions
1. Install dependencies: `npm install`
2. Launch dev workspace: `npm run dev`
