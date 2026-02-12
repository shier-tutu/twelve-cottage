<template>
  <div class="daily-container">
    <div class="date-selector">
      <button class="nav-btn" @click="previousDay">
        <svg viewBox="0 0 24 24" width="20" height="20" fill="currentColor">
          <path d="M15.41 7.41L14 6l-6 6 6 6 1.41-1.41L10.83 12z"/>
        </svg>
      </button>
      <div class="date-display" @click="openDatePicker">
        {{ formatDate(selectedDate) }}
        <input
          ref="dateInput"
          type="date"
          v-model="dateInputValue"
          @change="onDateChange"
          class="date-input-overlay"
        />
      </div>
      <button class="nav-btn" @click="nextDay" :disabled="isToday">
        <svg viewBox="0 0 24 24" width="20" height="20" fill="currentColor">
          <path d="M10 6L8.59 7.41 13.17 12l-4.58 4.59L10 18l6-6z"/>
        </svg>
      </button>
    </div>
    <div class="statistics-container">
      <div class="stat-item">
        <div class="stat-label">最大血糖波动幅度</div>
        <div class="stat-value">{{ statistics.maxFluctuation }} mmol/L</div>
      </div>
      <div class="stat-item">
        <div class="stat-label">最高血糖</div>
        <div class="stat-value">{{ statistics.maxValue }} mmol/L</div>
        <div class="stat-time">{{ statistics.maxTime }}</div>
      </div>
      <div class="stat-item">
        <div class="stat-label">最低血糖</div>
        <div class="stat-value">{{ statistics.minValue }} mmol/L</div>
        <div class="stat-time">{{ statistics.minTime }}</div>
      </div>
    </div>
    <div class="chart-container">
      <v-chart :option="chartOption" style="width: 100%; height: 100%" />
    </div>
  </div>
</template>

<script setup lang="ts">
import { onMounted, ref, computed } from 'vue'
import * as XLSX from 'xlsx'
import VChart from 'vue-echarts'
import { use } from 'echarts/core'
import { CanvasRenderer } from 'echarts/renderers'
import { LineChart, ScatterChart } from 'echarts/charts'
import {
  TitleComponent,
  TooltipComponent,
  GridComponent,
  LegendComponent,
  MarkAreaComponent
} from 'echarts/components'

// 注册 ECharts 组件
use([
  CanvasRenderer,
  LineChart,
  ScatterChart,
  TitleComponent,
  TooltipComponent,
  GridComponent,
  LegendComponent,
  MarkAreaComponent
])

const chartOption = ref({})
const allData = ref<any[]>([])
const selectedDate = ref(new Date())
const dateInputValue = ref('')
const dateInput = ref<HTMLInputElement>()

// 统计数据
const statistics = ref({
  maxFluctuation: 0,
  maxValue: 0,
  maxTime: '',
  minValue: 0,
  minTime: ''
})

// 判断是否是今天
const isToday = computed(() => {
  const today = new Date()
  const selected = selectedDate.value
  return (
    today.getFullYear() === selected.getFullYear() &&
    today.getMonth() === selected.getMonth() &&
    today.getDate() === selected.getDate()
  )
})

// 格式化日期显示
const formatDate = (date: Date) => {
  const year = date.getFullYear()
  const month = (date.getMonth() + 1).toString().padStart(2, '0')
  const day = date.getDate().toString().padStart(2, '0')
  return `${year}-${month}-${day}`
}

// 打开日期选择器
const openDatePicker = () => {
  if (dateInput.value) {
    const input = dateInput.value
    // 尝试使用 showPicker API（更可靠）
    if (typeof (input as any).showPicker === 'function') {
      try {
        ;(input as any).showPicker()
      } catch (error) {
        // 如果 showPicker 失败，回退到 focus
        input.focus()
      }
    } else {
      // 不支持 showPicker，使用 focus
      input.focus()
    }
  }
}

// 日期变化处理
const onDateChange = () => {
  if (dateInputValue.value) {
    const newDate = new Date(dateInputValue.value)
    selectedDate.value = newDate
    updateChart()
  }
}

// 前一天
const previousDay = () => {
  const newDate = new Date(selectedDate.value)
  newDate.setDate(newDate.getDate() - 1)
  selectedDate.value = newDate
  dateInputValue.value = formatDate(newDate)
  updateChart()
}

// 后一天
const nextDay = () => {
  if (!isToday.value) {
    const newDate = new Date(selectedDate.value)
    newDate.setDate(newDate.getDate() + 1)
    selectedDate.value = newDate
    dateInputValue.value = formatDate(newDate)
    updateChart()
  }
}

const updateChart = () => {
  if (allData.value.length === 0) return

  // 设置选中日期的开始和结束时间（00:00:00 到 23:59:59）
  const startTime = new Date(selectedDate.value)
  startTime.setHours(0, 0, 0, 0)

  const endTime = new Date(selectedDate.value)
  endTime.setHours(23, 59, 59, 999)

  // 过滤当天的数据
  const chartData: [string, number][] = []
  allData.value.forEach((item: any) => {
    const itemTime = new Date(item['血糖时间'])
    if (itemTime >= startTime && itemTime <= endTime) {
      chartData.push([item['血糖时间'], parseFloat(item['血糖值'])])
    }
  })

  // 计算统计数据
  const values = chartData.map(d => d[1])
  if (values.length > 0) {
    const maxVal = Math.max(...values)
    const minVal = Math.min(...values)
    const maxIndex = values.indexOf(maxVal)
    const minIndex = values.indexOf(minVal)

    // 格式化时间
    const formatTime = (timeStr: string) => {
      const time = new Date(timeStr)
      const month = (time.getMonth() + 1).toString().padStart(2, '0')
      const date = time.getDate().toString().padStart(2, '0')
      const hours = time.getHours().toString().padStart(2, '0')
      const minutes = time.getMinutes().toString().padStart(2, '0')
      return `${month}-${date} ${hours}:${minutes}`
    }

    const maxData = chartData[maxIndex]
    const minData = chartData[minIndex]

    statistics.value = {
      maxFluctuation: parseFloat((maxVal - minVal).toFixed(1)),
      maxValue: maxVal,
      maxTime: maxData ? formatTime(maxData[0]) : '',
      minValue: minVal,
      minTime: minData ? formatTime(minData[0]) : ''
    }
  } else {
    // 没有数据时重置统计
    statistics.value = {
      maxFluctuation: 0,
      maxValue: 0,
      maxTime: '',
      minValue: 0,
      minTime: ''
    }
  }

  // 计算 Y 轴最大值
  const maxValue = values.length > 0 ? Math.max(...values) : 30
  const yAxisMax = Math.max(30, Math.ceil(maxValue / 5) * 5)

  // 配置图表选项
  chartOption.value = {
    tooltip: {
      trigger: 'axis',
      formatter: (params: any) => {
        if (!params || params.length === 0) return ''
        const param = params[0]
        if (!param || !param.value) return ''
        const time = new Date(param.value[0])
        const month = (time.getMonth() + 1).toString()
        const date = time.getDate().toString()
        const hours = time.getHours().toString().padStart(2, '0')
        const minutes = time.getMinutes().toString().padStart(2, '0')
        return `${month}-${date} ${hours}:${minutes}<br/>血糖值: ${param.value[1]} mmol/L`
      }
    },
    grid: {
      left: '5%',
      right: '5%',
      bottom: '5%',
      top: '5%',
      containLabel: false
    },
    xAxis: {
      type: 'time',
      min: startTime.getTime(),
      max: endTime.getTime(),
      interval: 4 * 60 * 60 * 1000, // 4小时间隔
      axisLabel: {
        rotate: 45,
        fontSize: 10,
        formatter: '{HH}:{mm}'
      }
    },
    yAxis: {
      type: 'value',
      name: '血糖值 (mmol/L)',
      nameLocation: 'end',
      nameGap: 10,
      min: 0,
      max: yAxisMax
    },
    series: [
      {
        name: '血糖值',
        type: 'line',
        data: chartData,
        smooth: true,
        showSymbol: false,
        itemStyle: {
          color: '#5470c6'
        },
        lineStyle: {
          width: 2
        },
        markArea: {
          silent: true,
          itemStyle: {
            color: 'rgba(144, 238, 144, 0.2)' // 浅绿色背景
          },
          data: [
            [
              {
                yAxis: 3.7
              },
              {
                yAxis: 10
              }
            ]
          ]
        }
      }
    ]
  }
}

onMounted(async () => {
  try {
    const filePath = `${import.meta.env.BASE_URL}data/001.xlsx`
    console.log('文件路径:', filePath)

    const response = await fetch(filePath)

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`)
    }

    const arrayBuffer = await response.arrayBuffer()

    // 解析 Excel 文件
    const workbook = XLSX.read(arrayBuffer, { type: 'array' })

    if (workbook.SheetNames.length === 0) {
      console.error('Excel 文件中没有工作表')
      return
    }

    const firstSheetName = workbook.SheetNames[0]!
    const worksheet = workbook.Sheets[firstSheetName]!
    const data = XLSX.utils.sheet_to_json(worksheet)

    console.log('Excel 数据:', data)

    // 存储所有数据
    allData.value = data

    // 初始化图表
    updateChart()
  } catch (error) {
    console.error('读取失败:', error)
  }
})
</script>

<style scoped>
.daily-container {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
}

.date-selector {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 15px 10px;
  background: white;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
  gap: 20px;
}

.nav-btn {
  width: 36px;
  height: 36px;
  border: none;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 2px 8px rgba(102, 126, 234, 0.2);
}

.nav-btn:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
}

.nav-btn:active:not(:disabled) {
  transform: translateY(0);
}

.nav-btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
  background: #e0e0e0;
  box-shadow: none;
}

.confirm-btn {
  padding: 8px 20px;
  border: none;
  background: linear-gradient(135deg, #11998e 0%, #38ef7d 100%);
  color: white;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s ease;
  font-size: 14px;
  font-weight: 600;
  box-shadow: 0 2px 8px rgba(17, 153, 142, 0.2);
  white-space: nowrap;
}

.confirm-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(17, 153, 142, 0.4);
}

.confirm-btn:active {
  transform: translateY(0);
}

.date-display {
  font-size: 16px;
  font-weight: 600;
  color: #333;
  min-width: 140px;
  text-align: center;
  padding: 8px 16px;
  background: #f8f9ff;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s ease;
  position: relative;
  border: 2px solid transparent;
}

.date-display:hover {
  background: #eef1ff;
  border-color: #667eea;
}

.date-input-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  opacity: 0;
  cursor: pointer;
  border: none;
  background: transparent;
}

/* 隐藏原生日历图标 */
.date-input-overlay::-webkit-calendar-picker-indicator {
  display: none;
  -webkit-appearance: none;
}

.date-input-overlay::-moz-calendar-picker-indicator {
  display: none;
}

.statistics-container {
  display: flex;
  justify-content: space-around;
  padding: 15px 10px;
  background: white;
  gap: 10px;
}

.stat-item {
  flex: 1;
  text-align: center;
  padding: 15px 10px;
  background: #f8f9fa;
  border-radius: 8px;
  border: 1px solid #e9ecef;
}

.stat-label {
  font-size: 12px;
  color: #6c757d;
  margin-bottom: 8px;
}

.stat-value {
  font-size: 18px;
  font-weight: bold;
  color: #212529;
  margin-bottom: 4px;
}

.stat-time {
  font-size: 11px;
  color: #868e96;
}

.chart-container {
  height: 300px;
  background: white;
  padding: 20px 10px;
  border-radius: 0;
  box-shadow: none;
  overflow: hidden;
}
</style>
