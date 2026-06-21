<template>
  <div
    class="card w-full"
    v-if="hasProvidersWithTraffic"
  >
    <div class="card-title px-4 pt-4">
      {{ $t('providerTrafficOverview') }}
    </div>
    <div
      class="card-body grid max-h-128 gap-2 overflow-y-auto"
      :style="
        hasMultipleProvidersWithTraffic
          ? `grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));`
          : 'grid-template-columns: 1fr;'
      "
    >
      <!-- 总流量 -->
      <div
        class="bg-base-200/50 flex flex-col gap-2 rounded-lg p-2"
        v-if="hasMultipleProvidersWithTraffic"
      >
        <div class="flex items-center justify-between">
          <div class="text-lg font-medium">
            {{ $t('totalTraffic') }}
          </div>
          <div class="text-base-content/70 text-sm">{{ totalPercentage }}%</div>
        </div>
        <div class="w-full">
          <progress
            class="progress h-2 w-full"
            :class="getProgressColor(totalPercentage)"
            :value="totalPercentage"
            max="100"
          ></progress>
        </div>
        <div class="text-base-content/60 flex items-center justify-between text-sm">
          <div>{{ $t('remainingTraffic') }}: {{ totalRemainingStr }}</div>
          <div>{{ $t('usedTraffic') }}: {{ totalUsedStr }} / {{ totalTotalStr }}</div>
        </div>
        <!-- 今日流量 -->
        <div class="text-base-content/60 flex items-center justify-between text-sm">
          <div :class="{ 'text-error font-medium': totalTodayOverWarn }">
            {{ $t('todayTraffic') }}: {{ totalTodayUsedStr }}
            <span v-if="totalTodayOverWarn">⚠️</span>
          </div>
        </div>
      </div>
      <!-- 各提供商流量 -->
      <div
        v-for="provider in providersWithTraffic"
        :key="provider.name"
        class="bg-base-200/50 flex flex-col gap-2 rounded-lg p-2"
      >
        <div class="flex items-center justify-between">
          <div class="text-lg font-medium">
            {{ provider.name }}
          </div>
          <div class="text-base-content/70 text-sm">{{ provider.percentage }}%</div>
        </div>
        <div class="w-full">
          <progress
            class="progress h-2 w-full"
            :class="getProgressColor(provider.percentage)"
            :value="provider.percentage"
            max="100"
          ></progress>
        </div>
        <div class="text-base-content/60 flex items-center justify-between text-sm">
          <div>{{ $t('remainingTraffic') }}: {{ provider.remainingStr }}</div>
          <div>{{ $t('usedTraffic') }}: {{ provider.usedStr }} / {{ provider.totalStr }}</div>
        </div>
        <!-- 今日流量 + 到期 -->
        <div class="text-base-content/60 flex items-center justify-between text-sm">
          <div :class="{ 'text-error font-medium': isOverWarn(provider.name) }">
            {{ $t('todayTraffic') }}: {{ todayUsedStr(provider.name) }}
            <span v-if="isOverWarn(provider.name)">⚠️</span>
          </div>
          <div
            v-if="provider.expireDate"
            :class="{ 'text-warning': provider.expireDays >= 0 && provider.expireDays <= 7 }"
          >
            {{ $t('expire') }}: {{ provider.expireDate }}
            <span v-if="provider.expireDays >= 0"
              >({{ $t('daysRemaining', { days: provider.expireDays }) }})</span
            >
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { prettyBytesHelper } from '@/helper/utils'
import { proxyProviederList } from '@/store/proxies'
import { useStorage } from '@vueuse/core'
import { toFinite } from 'lodash'
import { computed, watch } from 'vue'

interface ProviderTrafficInfo {
  name: string
  used: number
  remaining: number
  total: number
  percentage: number
  usedStr: string
  remainingStr: string
  totalStr: string
  expireDate: string
  expireDays: number
}

// ---- 每日流量快照(本地持久化)：{ 机场名: { 'YYYY-MM-DD': { start, end } } } ----
type DayRecord = { start: number; end: number }
const trafficHistory = useStorage<Record<string, Record<string, DayRecord>>>(
  'overview/provider-traffic-history',
  {},
)
// 今日流量预警阈值(GB)，超过则高亮。0 = 不预警
const dailyWarnGB = useStorage<number>('overview/provider-traffic-daily-warn-gb', 50)

const localDateStr = (d = new Date()): string => {
  const y = d.getFullYear()
  const m = String(d.getMonth() + 1).padStart(2, '0')
  const day = String(d.getDate()).padStart(2, '0')
  return `${y}-${m}-${day}`
}

const providersWithTraffic = computed<ProviderTrafficInfo[]>(() => {
  return proxyProviederList.value
    .filter((provider) => {
      const info = provider.subscriptionInfo
      return info && info.Total && info.Total > 0
    })
    .map((provider) => {
      const { Download = 0, Upload = 0, Total = 0, Expire = 0 } = provider.subscriptionInfo!
      const used = Download + Upload
      const remaining = Math.max(0, Total - used)
      const percentage = Total > 0 ? toFinite(((used / Total) * 100).toFixed(2)) : 0

      let expireDate = ''
      let expireDays = -1
      if (Expire && Expire > 0) {
        const dt = new Date(Expire * 1000)
        expireDate = localDateStr(dt)
        expireDays = Math.floor((Expire * 1000 - Date.now()) / 86400000)
      }

      return {
        name: provider.name,
        used,
        remaining,
        total: Total,
        percentage,
        usedStr: prettyBytesHelper(used, { binary: true }),
        remainingStr: prettyBytesHelper(remaining, { binary: true }),
        totalStr: prettyBytesHelper(Total, { binary: true }),
        expireDate,
        expireDays,
      }
    })
})

// 记录每日快照(以 providersWithTraffic 为驱动；不读 trafficHistory，避免与下方计算成环)
watch(
  providersWithTraffic,
  (list) => {
    const today = localDateStr()
    for (const p of list) {
      const hist = trafficHistory.value[p.name] ?? (trafficHistory.value[p.name] = {})
      if (!hist[today]) {
        // 以"最近一个历史日的 end"为今日基线，保证跨天不丢量；无历史则从当前值起算
        const priorDates = Object.keys(hist)
          .filter((d) => d < today)
          .sort()
        const baseline = priorDates.length ? hist[priorDates[priorDates.length - 1]].end : p.used
        hist[today] = { start: Math.min(baseline, p.used), end: p.used }
      } else {
        // 检测到计数器重置(订阅续费/机场清零)→ 重新基线，避免出现负数
        if (p.used < hist[today].start) hist[today].start = p.used
        hist[today].end = p.used
      }
      // 仅保留最近 30 天，防止 localStorage 膨胀
      const dates = Object.keys(hist).sort()
      while (dates.length > 30) {
        delete hist[dates.shift() as string]
      }
    }
  },
  { immediate: true, deep: true },
)

// ---- 今日流量(读 trafficHistory，模板方法形式，随快照变化自动更新)----
const todayUsedBytes = (name: string): number => {
  const rec = trafficHistory.value[name]?.[localDateStr()]
  return rec ? Math.max(0, rec.end - rec.start) : 0
}
const todayUsedStr = (name: string): string =>
  prettyBytesHelper(todayUsedBytes(name), { binary: true })
const isOverWarn = (name: string): boolean =>
  dailyWarnGB.value > 0 && todayUsedBytes(name) > dailyWarnGB.value * 1024 ** 3

const hasProvidersWithTraffic = computed(() => providersWithTraffic.value.length > 0)
const hasMultipleProvidersWithTraffic = computed(() => providersWithTraffic.value.length > 1)

// 计算总流量
const totalTraffic = computed(() => {
  return providersWithTraffic.value.reduce(
    (acc, provider) => ({
      used: acc.used + provider.used,
      remaining: acc.remaining + provider.remaining,
      total: acc.total + provider.total,
    }),
    { used: 0, remaining: 0, total: 0 },
  )
})

const totalPercentage = computed(() => {
  const { used, total } = totalTraffic.value
  return total > 0 ? toFinite(((used / total) * 100).toFixed(2)) : 0
})

const totalUsedStr = computed(() => prettyBytesHelper(totalTraffic.value.used, { binary: true }))
const totalRemainingStr = computed(() =>
  prettyBytesHelper(totalTraffic.value.remaining, { binary: true }),
)
const totalTotalStr = computed(() => prettyBytesHelper(totalTraffic.value.total, { binary: true }))

// 今日总流量(各机场今日用量之和)
const totalTodayBytes = computed(() =>
  providersWithTraffic.value.reduce((acc, p) => acc + todayUsedBytes(p.name), 0),
)
const totalTodayUsedStr = computed(() => prettyBytesHelper(totalTodayBytes.value, { binary: true }))
const totalTodayOverWarn = computed(
  () => dailyWarnGB.value > 0 && totalTodayBytes.value > dailyWarnGB.value * 1024 ** 3,
)

const getProgressColor = (percentage: number) => {
  if (percentage >= 90) return 'progress-error'
  if (percentage >= 70) return 'progress-warning'
  return 'progress-success'
}
</script>
