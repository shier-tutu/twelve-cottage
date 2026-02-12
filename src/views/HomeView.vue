<template>
  <div class="container">
    <div class="header">
      <h2>监测报表</h2>
    </div>
    <div class="tabs">
      <button
        v-for="tab in tabs"
        :key="tab.value"
        :class="['tab-btn', { active: activeTab === tab.value }]"
        @click="activeTab = tab.value"
      >
        {{ tab.label }}
      </button>
    </div>
    <div class="content">
      <RealtimeMonitor v-if="activeTab === 'realtime'" />
      <DailyBloodSugar v-if="activeTab === 'daily'" />
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import RealtimeMonitor from '@/components/RealtimeMonitor.vue'
import DailyBloodSugar from '@/components/DailyBloodSugar.vue'

const activeTab = ref('realtime')

const tabs = [
  { label: '实时检测', value: 'realtime' },
  { label: '每日血糖', value: 'daily' }
]
</script>

<style scoped>
.container {
  width: 100%;
  height: 100%;
  margin: 0;
  padding: 0;
  display: flex;
  flex-direction: column;
  background: #f8f9fa;
}

.header {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  padding: 15px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

h2 {
  margin: 0;
  color: white;
  text-align: center;
  font-size: 18px;
  font-weight: 600;
}

.tabs {
  display: flex;
  background: white;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.tab-btn {
  flex: 1;
  padding: 15px;
  border: none;
  background: white;
  color: #666;
  font-size: 15px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s ease;
  border-bottom: 3px solid transparent;
}

.tab-btn:hover {
  color: #667eea;
}

.tab-btn.active {
  color: #667eea;
  border-bottom-color: #667eea;
  font-weight: 600;
}

.content {
  flex: 1;
  overflow: hidden;
}
</style>

