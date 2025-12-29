<script setup>
import CalHeatmap from 'cal-heatmap'
import Tooltip from 'cal-heatmap/plugins/Tooltip'
import LegendLite from 'cal-heatmap/plugins/LegendLite'
import axios from 'axios'
import { onMounted, ref, reactive, computed, watch } from 'vue'

const dataTypeConfig = {
  distance: { alia: 'd', scaleMax: 50, unit: 'km' },
  moving_time: { alia: 'mt', scaleMax: 120, unit: 'min' },
  elapsed_time: { alia: 'et', scaleMax: 240, unit: 'min' },
  elevation_gain: { alia: 'eg', scaleMax: 200, unit: 'm' },
  activity: { alia: 'a', scaleMax: 5, unit: 'times' },
}

const domainRangeConfig = {
  year: 1,
  month: 12,
}

const subDomainConfig = {
  day: 1,
  week: 3,
  month: 10,
}

const sportTypes = [
  { key: 'All', value: 'all' },
  { key: 'Ride', value: 'Ride' },
  { key: 'Run', value: 'Run' },
]

const domainModes = [
  { key: 'Year', value: 'year' },
  { key: 'Month', value: 'month' },
]

const subDomainModes = [
  { key: 'Day', value: 'day' },
  { key: 'Week', value: 'week' },
  { key: 'Month', value: 'month' },
]

const dataTypes = [
  { key: 'Activity', value: 'activity' },
  { key: 'Distance', value: 'distance' },
  { key: 'Moving Time', value: 'moving_time' },
  { key: 'Elapsed Time', value: 'elapsed_time' },
  { key: 'Elevation Gain', value: 'elevation_gain' },
]

const heatmapState = reactive({
  sportType: 'all',
  start: new Date(Date.UTC(new Date().getFullYear(), 0, 1)).getTime(),
  domainMode: 'year',
  subDomainMode: 'day',
  dataType: 'distance',
})

// cal-heatmap
const subDomain_w = 20
const subDomain_h = 20
const subDomain_gutter = 2

// ref
const rawHeatmapData = ref([])

// computed
const computedHeatmapData = computed(() => {
  let data = rawHeatmapData.value
  const end = new Date(new Date(heatmapState.start).getFullYear() + 1, 0, 1).getTime()
  data = data.filter((d) => d.t > heatmapState.start).filter((d) => d.t < end)

  if (heatmapState.sportType !== 'all') {
    data = data.filter((d) => d.type === heatmapState.sportType)
  }

  return data
})

const unit = computed(() => dataTypeConfig[heatmapState.dataType].unit)

const counter = computed(() => {
  let c = 0
  for (const item of computedHeatmapData.value) {
    c += item[dataTypeConfig[heatmapState.dataType].alia]
  }
  return c
})

const scaleDomain = computed(() => [
  0,
  dataTypeConfig[heatmapState.dataType].scaleMax * subDomainConfig[heatmapState.subDomainMode],
])

const getTooltipText = (timestamp, value, dayjsDate) => {
  return `${dayjsDate.format('YYYY/MM/DD')} - ${value}${unit.value}`
}

const baseTemplate = {
  itemSelector: '#cal-heatmap',
  subDomain: {
    width: subDomain_w,
    height: subDomain_h,
    gutter: subDomain_gutter,
    radius: 1,
  },
  date: {
    highlight: [new Date()],
    locale: { weekStart: 0 },
  },
  data: {
    type: 'json',
    x: 't', // 告訴他哪個欄位是日期
    groupY: 'sum', // 若一天有多筆資料，聚合方式
    defaultValue: 0, // 沒資料時顯示的值
  },
  scale: {
    color: {
      range: [
        '#ebedf0',
        '#d6f5d6',
        '#bff0b7',
        '#a8eb99',
        '#91e87b',
        '#7bd95f',
        '#64c963',
        '#4db753',
        '#36a144',
        '#216e39',
      ],
      type: 'quantile',
    },
  },
}

const computedTemplate = computed(() => ({
  itemSelector: baseTemplate.itemSelector,
  range: domainRangeConfig[heatmapState.domainMode],
  domain: { type: heatmapState.domainMode },
  subDomain: {
    ...baseTemplate.subDomain,
    type: heatmapState.subDomainMode,
  },
  date: {
    ...baseTemplate.date,
    start: new Date(heatmapState.start),
  },
  data: {
    ...baseTemplate.data,
    source: computedHeatmapData.value,
    y: dataTypeConfig[heatmapState.dataType].alia,
  },
  scale: {
    color: {
      ...baseTemplate.scale.color,
      domain: scaleDomain.value,
    },
  },
}))

const calheatmapPlugins = ref([
  [
    Tooltip,
    {
      text: getTooltipText,
    },
  ],
  [
    LegendLite,
    {
      itemSelector: '#legend',
      width: subDomain_w,
      height: subDomain_h,
      radius: 1,
    },
  ],
])

const yearSelectorList = (start, end) => {
  const end_year = new Date(start).getUTCFullYear()
  const start_year = new Date(end).getUTCFullYear()
  const yearList = []
  for (let i = end_year; i >= start_year; i--) {
    yearList.push({ key: i, value: new Date(Date.UTC(i, 0, 1)).getTime() })
  }

  return yearList
}

let cal = null

onMounted(async () => {
  const req = await axios.get('https://strava.watercatuwu.workers.dev/heatmap')
  rawHeatmapData.value = req.data
  console.log(computedTemplate.value)
  cal = new CalHeatmap()
  cal.paint(computedTemplate.value, calheatmapPlugins.value)
})

watch(
  () => heatmapState.subDomainMode,
  () => {
    if (cal) cal.destroy()
    cal = new CalHeatmap()
    cal.paint(computedTemplate.value, calheatmapPlugins.value)
  },
)

watch(
  () => [
    heatmapState.domainMode,
    heatmapState.sportType,
    heatmapState.start,
    heatmapState.dataType,
  ],
  () => {
    cal.paint(computedTemplate.value, calheatmapPlugins.value)
  },
)
</script>

<template>
  <div v-if="rawHeatmapData.length > 0" id="main">
    <div id="heatmap-box">
      <div id="nav">
        <div id="dropdown-box">
          <select v-model="heatmapState.sportType" name="sportTypeSelector" class="dropdown">
            <option disabled value="">Sport Type</option>
            <option v-for="s in sportTypes" :key="s.key" :value="s.value">
              {{ s.key }}
            </option>
          </select>
          <select v-model="heatmapState.start" name="yearSelector" class="dropdown">
            <option disabled value="">Year</option>
            <option
              v-for="y in yearSelectorList(
                rawHeatmapData[0].t,
                rawHeatmapData[rawHeatmapData.length - 1].t,
              )"
              :key="y.key"
              :value="y.value"
            >
              {{ y.key }}
            </option>
          </select>
          <select
            v-model="heatmapState.domainMode"
            name="domainModeSelector"
            class="dropdown"
            :disabled="heatmapState.subDomainMode === 'month'"
          >
            <option disabled value="">Domain</option>
            <option v-for="m in domainModes" :key="m.key" :value="m.value">
              {{ m.key }}
            </option>
          </select>
          <select
            v-model="heatmapState.subDomainMode"
            name="subDomainModeSelector"
            class="dropdown"
          >
            <option disabled value="">SubDomain</option>
            <option v-for="m in subDomainModes" :key="m.key" :value="m.value">
              {{ m.key }}
            </option>
          </select>
          <select v-model="heatmapState.dataType" name="dataTypeSelector" class="dropdown">
            <option disabled value="">Data Type</option>
            <option v-for="d in dataTypes" :key="d.key" :value="d.value">
              {{ d.key }}
            </option>
          </select>
        </div>
        <div class="title">
          <img id="avatar" src="/user.jpg" />
        </div>
      </div>
      <div style="display: flex; justify-content: center">
        <div id="cal-heatmap"></div>
      </div>
      <div id="bottom-box" class="flex-between">
        <div class="title">{{ counter }}{{ unit }}</div>
        <div>
          <div id="legend"></div>
          <div
            class="flex-between"
            :style="{ width: `${subDomain_w * baseTemplate.scale.color.range.length}` }"
          >
            <span>{{ scaleDomain[0] }}{{ unit }}</span>
            <span>{{ scaleDomain[1] }}{{ unit }}</span>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.flex-between {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.btn {
  padding: 1rem 2rem 1rem 2rem;
  font-size: large;
  border-width: 0;
  border-radius: 0.5rem;
  background-color: #ebedf0;
  cursor: pointer;

  transition:
    background-color 0.15s ease,
    box-shadow 0.15s ease,
    transform 0.1s ease;
}

.btn:hover {
  background-color: #e2e2e2;
}

.dropdown {
  padding: 0.5rem 1rem 0.5rem 1rem;
  height: 3rem;
  font-size: large;
  border-width: 0;
  border-radius: 0.5rem;
  background-color: rgba(0, 0, 0, 0);
  cursor: pointer;
  outline: none;

  transition:
    background-color 0.15s ease,
    box-shadow 0.15s ease,
    transform 0.1s ease;
}

.dropdown:hover {
  background-color: #e2e2e2;
}

#dropdown-box {
  display: flex;
}

#nav {
  display: flex;
  width: 100%;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}

#main {
  display: flex;
  min-height: 100%;
  align-items: center;
  justify-content: center;
  flex-direction: column;
}

#heatmap-box {
  background: #fff;
  max-width: 100%;
  padding: 1rem 1rem 1rem 1rem;
}

.title {
  display: flex;
  align-items: center;
  height: 4rem;
  font-size: 2rem;
  font-weight: 500;
}

#bottom-box {
  width: 100%;
  margin-top: 1rem;
  padding: 0 0.5rem 0 0.5rem;
  box-sizing: border-box;
}

#btn-box {
  display: flex;
  align-items: center;
  justify-content: center;
  margin-top: 1rem;
}

#avatar {
  border-radius: 50%;
  height: 4rem;
  padding-left: 1rem;
}
</style>
