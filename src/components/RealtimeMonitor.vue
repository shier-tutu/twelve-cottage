<template>
  <div class="realtime-container">
    <div class="time-filters">
      <button
        v-for="filter in timeFilters"
        :key="filter.value"
        :class="['filter-btn', { active: selectedFilter === filter.value }]"
        @click="selectTimeFilter(filter.value)"
      >
        {{ filter.label }}
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
import { onMounted, ref } from 'vue'
import * as XLSX from 'xlsx'
import VChart from 'vue-echarts'
import { use } from 'echarts/core'
import { CanvasRenderer } from 'echarts/renderers'
import { LineChart, ScatterChart } from 'echarts/charts'
import {
  TitleComponent,
  TooltipComponent,
  GridComponent,
  LegendComponent
} from 'echarts/components'

// 注册 ECharts 组件
use([
  CanvasRenderer,
  LineChart,
  ScatterChart,
  TitleComponent,
  TooltipComponent,
  GridComponent,
  LegendComponent
])

const chartOption = ref({})
const selectedFilter = ref(24)
const allData = ref<any[]>([])

// 统计数据
const statistics = ref({
  maxFluctuation: 0,
  maxValue: 0,
  maxTime: '',
  minValue: 0,
  minTime: ''
})

const timeFilters = [
  { label: '3小时', value: 3 },
  { label: '6小时', value: 6 },
  { label: '12小时', value: 12 },
  { label: '24小时', value: 24 }
]

const selectTimeFilter = (hours: number) => {
  selectedFilter.value = hours
  updateChart()
}

const updateChart = () => {
  if (allData.value.length === 0) return

  // 时间刻度间隔映射（毫秒）
  const intervalMap: Record<number, number> = {
    3: 30 * 60 * 1000,      // 3小时 → 半小时间隔
    6: 60 * 60 * 1000,      // 6小时 → 1小时间隔
    12: 2 * 60 * 60 * 1000, // 12小时 → 2小时间隔
    24: 4 * 60 * 60 * 1000  // 24小时 → 4小时间隔
  }

  // 1. 找到最后一条数据的时间作为结束时间
  const lastItem = allData.value[allData.value.length - 1]
  const endTime = new Date(lastItem['血糖时间'])

  // 2. 计算起始时间（往前推 N 小时）
  const startTime = new Date(endTime)
  startTime.setHours(startTime.getHours() - selectedFilter.value)

  // 3. 对齐起始时间到合适的刻度
  if (selectedFilter.value === 3) {
    // 3小时：对齐到半小时（00或30分钟）
    const minutes = startTime.getMinutes()
    if (minutes < 30) {
      startTime.setMinutes(0)
    } else {
      startTime.setMinutes(30)
    }
  } else if (selectedFilter.value === 6) {
    // 6小时：对齐到整点
    startTime.setMinutes(0)
  } else if (selectedFilter.value === 12) {
    // 12小时：对齐到2小时的倍数
    const hours = startTime.getHours()
    startTime.setHours(Math.floor(hours / 2) * 2)
    startTime.setMinutes(0)
  } else if (selectedFilter.value === 24) {
    // 24小时：对齐到4小时的倍数
    const hours = startTime.getHours()
    startTime.setHours(Math.floor(hours / 4) * 4)
    startTime.setMinutes(0)
  }
  startTime.setSeconds(0)
  startTime.setMilliseconds(0)

  // 4. 过滤时间范围内的数据，并转换为 [时间, 血糖值] 格式
  const chartData: [string, number][] = []
  allData.value.forEach((item: any) => {
    const itemTime = new Date(item['血糖时间'])
    if (itemTime >= startTime && itemTime <= endTime) {
      chartData.push([item['血糖时间'], parseFloat(item['血糖值'])])
    }
  })

  // 5. 计算统计数据
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
  }

  // 6. 计算 Y 轴最大值：向上取整到最接近的 5 的倍数，最小值为 30
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
      interval: intervalMap[selectedFilter.value],
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
        type: selectedFilter.value === 3 ? 'scatter' : 'line', // 只有3小时用散点图，其他用折线图
        data: chartData,
        smooth: selectedFilter.value >= 12, // 12-24小时使用平滑曲线
        showSymbol: selectedFilter.value === 3, // 只有3小时显示点
        symbol: 'circle',
        symbolSize: selectedFilter.value === 3 ? 8 : 0,
        itemStyle: {
          color: '#5470c6'
        },
        lineStyle: {
          width: 2
        }
      }
    ]
  }
}

onMounted(async () => {
  try {
    // 使用 BASE_URL 来适配不同环境的路径
    const filePath = `${import.meta.env.BASE_URL}data/001.xlsx`
    console.log('文件路径:', filePath)

    const response = await fetch(filePath)

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`)
    }

    const arrayBuffer = await response.arrayBuffer()

    // 解析 Excel 文件
    const workbook = XLSX.read(arrayBuffer, { type: 'array' })

    // 检查是否有工作表
    if (workbook.SheetNames.length === 0) {
      console.error('Excel 文件中没有工作表')
      return
    }

    const firstSheetName = workbook.SheetNames[0]!
    const worksheet = workbook.Sheets[firstSheetName]!
    const data = XLSX.utils.sheet_to_json(worksheet)

    // 在控制台打印 JSON 数据
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
.realtime-container {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
}

.time-filters {
  display: flex;
  justify-content: space-around;
  padding: 10px;
  background: white;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.filter-btn {
  flex: 1;
  margin: 0 5px;
  padding: 10px 15px;
  border: 2px solid #e0e0e0;
  background: white;
  color: #666;
  border-radius: 20px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s ease;
}

.filter-btn:hover {
  border-color: #667eea;
  color: #667eea;
}

.filter-btn.active {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border-color: #667eea;
  color: white;
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.3);
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
