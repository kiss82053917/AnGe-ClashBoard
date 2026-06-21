<template>
  <div class="card w-full overflow-hidden backdrop-blur-none!">
    <div class="card-title need-blur flex items-center justify-between px-4 pt-4">
      <div class="flex w-full items-center gap-4 max-sm:flex-col max-sm:items-start">
        <div class="flex flex-1 items-center gap-2">
          {{ $t('totalConnections') }}

          <button
            class="btn btn-circle btn-sm"
            @click="showClearDialog = true"
          >
            <TrashIcon class="h-4 w-4" />
          </button>
          <QuestionMarkCircleIcon
            class="h-4 w-4 cursor-pointer"
            @mouseenter="showTip($event, totalConnectionsTip)"
          />
        </div>

        <div class="flex items-center gap-2 font-normal max-sm:flex-col max-sm:items-start">
          <div class="flex items-center gap-2">
            <span class="text-sm">{{ $t('aggregateBy') }}</span>
            <select
              v-model="aggregationType"
              class="select select-bordered select-sm w-32"
            >
              <option :value="ConnectionHistoryType.SourceIP">
                {{ $t('aggregateBySourceIP') }}
              </option>
              <option :value="ConnectionHistoryType.Destination">
                {{ $t('aggregateByDestination') }}
              </option>
              <option :value="ConnectionHistoryType.Process">{{ $t('aggregateByProcess') }}</option>
              <option :value="ConnectionHistoryType.Outbound">
                {{ $t('aggregateByOutbound') }}
              </option>
              <option :value="ConnectionHistoryType.Airport">
                {{ $t('aggregateByAirport') }}
              </option>
            </select>
          </div>
          <div class="flex items-center gap-2">
            <span class="text-sm">{{ $t('autoCleanupInterval') }}</span>
            <select
              v-model="autoCleanupInterval"
              class="select select-bordered select-sm w-28"
            >
              <option :value="AutoCleanupInterval.Never">
                {{ $t('autoCleanupIntervalNever') }}
              </option>
              <option :value="AutoCleanupInterval.Week">{{ $t('autoCleanupIntervalWeek') }}</option>
              <option :value="AutoCleanupInterval.Month">
                {{ $t('autoCleanupIntervalMonth') }}
              </option>
              <option :value="AutoCleanupInterval.Quarter">
                {{ $t('autoCleanupIntervalQuarter') }}
              </option>
            </select>
          </div>
        </div>
      </div>
    </div>
    <div class="card-body need-blur gap-0! p-0!">
      <div class="px-4 py-4">
        <div
          class="stats stats-vertical sm:stats-horizontal bg-base-200 w-full gap-2 rounded-[inherit] shadow max-md:grid max-md:grid-cols-2"
        >
          <div class="stat">
            <div class="stat-title text-xs">{{ aggregateSourceLabel }}</div>
            <div class="stat-value text-lg">{{ aggregateSourceCount }}</div>
          </div>
          <div class="stat md:hidden"></div>
          <div class="stat">
            <div class="stat-title text-xs">{{ t('download') }}</div>
            <div class="stat-value text-lg">{{ prettyBytesHelper(totalStats.download) }}</div>
          </div>
          <div class="stat">
            <div class="stat-title text-xs">{{ t('upload') }}</div>
            <div class="stat-value text-lg">{{ prettyBytesHelper(totalStats.upload) }}</div>
          </div>
          <div class="stat">
            <div class="stat-title text-xs">{{ t('totalTraffic') }}</div>
            <div class="stat-value text-lg">
              {{ prettyBytesHelper(totalStats.download + totalStats.upload) }}
            </div>
          </div>
          <div class="stat">
            <div class="stat-title text-xs">{{ t('connectionCount') }}</div>
            <div class="stat-value text-lg">{{ totalStats.count.toString() }}</div>
          </div>
        </div>
      </div>
    </div>
    <div
      ref="parentRef"
      class="h-96 overflow-auto rounded-b-[inherit]"
      @touchstart.passive.stop
      @touchmove.passive.stop
      @touchend.passive.stop
    >
      <div :style="{ height: `${totalSize}px` }">
        <table class="table-sm table-zebra table w-full rounded-none">
          <thead class="bg-base-200 sticky top-0 z-10">
            <tr>
              <th
                v-for="header in tanstackTable.getHeaderGroups()[0]?.headers"
                :key="header.id"
                class="cursor-pointer select-none"
                @click="header.column.getToggleSortingHandler()?.($event)"
              >
                <div class="flex items-center gap-1">
                  <FlexRender
                    v-if="!header.isPlaceholder"
                    :render="header.column.columnDef.header"
                    :props="header.getContext()"
                  />
                  <ArrowUpCircleIcon
                    v-if="header.column.getIsSorted() === 'asc'"
                    class="h-4 w-4"
                  />
                  <ArrowDownCircleIcon
                    v-if="header.column.getIsSorted() === 'desc'"
                    class="h-4 w-4"
                  />
                </div>
              </th>
            </tr>
          </thead>
          <tbody>
            <tr
              v-for="(virtualRow, index) in virtualRows"
              :key="virtualRow.key.toString()"
              :style="{
                height: `${virtualRow.size}px`,
                transform: `translateY(${virtualRow.start - index * virtualRow.size}px)`,
              }"
              class="hover:bg-primary! hover:text-primary-content whitespace-nowrap"
              :class="{ 'cursor-pointer': aggregationType === ConnectionHistoryType.Airport }"
              @click="onRowClick(rows[virtualRow.index])"
            >
              <td
                v-for="cell in rows[virtualRow.index].getVisibleCells()"
                :key="cell.id"
                class="text-sm"
              >
                <FlexRender
                  :render="cell.column.columnDef.cell"
                  :props="cell.getContext()"
                />
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
    <DialogWrapper
      v-model="showClearDialog"
      :title="$t('clearConnectionHistory')"
    >
      <div class="flex flex-col gap-4 p-2">
        <p class="text-sm">
          {{ $t('clearConnectionHistoryConfirm') }}
        </p>
        <div class="flex justify-end gap-2">
          <button
            class="btn btn-sm"
            @click="showClearDialog = false"
          >
            {{ $t('cancel') }}
          </button>
          <button
            class="btn btn-error btn-sm"
            @click="handleClearHistory"
          >
            {{ $t('confirm') }}
          </button>
        </div>
      </div>
    </DialogWrapper>
    <DialogWrapper
      v-model="showDrill"
      :title="`${drillAirport} · ${$t('nodeDetail')}`"
    >
      <div class="flex flex-col gap-3 p-2">
        <div class="bg-base-200/50 flex flex-wrap items-center gap-2 rounded-lg p-3 text-sm">
          <span class="font-medium">{{ $t('airportWarnThreshold') }}</span>
          <input
            type="number"
            min="0"
            class="input input-bordered input-sm w-24"
            :value="drillThreshold || ''"
            placeholder="0"
            @input="setDrillThreshold(($event.target as HTMLInputElement).value)"
          />
          <span>GB</span>
          <span class="text-base-content/60 w-full text-xs">{{ $t('airportWarnHint') }}</span>
        </div>
        <div class="max-h-96 overflow-auto">
          <table class="table-sm table-zebra table w-full">
            <thead class="bg-base-200 sticky top-0 z-10">
              <tr>
                <th>{{ $t('outbound') }}</th>
                <th>{{ $t('download') }}</th>
                <th>{{ $t('upload') }}</th>
                <th>{{ $t('totalTraffic') }}</th>
                <th>{{ $t('connectionCount') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="n in drillNodes"
                :key="n.key"
                class="whitespace-nowrap"
              >
                <td><ProxyName :name="n.key" /></td>
                <td>{{ prettyBytesHelper(n.download) }}</td>
                <td>{{ prettyBytesHelper(n.upload) }}</td>
                <td>{{ prettyBytesHelper(n.download + n.upload) }}</td>
                <td>{{ n.count }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </DialogWrapper>
  </div>
</template>

<script setup lang="ts">
import { ConnectionHistoryType, clearConnectionHistoryFromIndexedDB } from '@/helper/indexeddb'
import { showNotification } from '@/helper/notification'
import { getIPLabelFromMap } from '@/helper/sourceip'
import { useTooltip } from '@/helper/tooltip'
import { prettyBytesHelper } from '@/helper/utils'
import {
  aggregateConnections,
  aggregatedDataMap,
  airportOfNode,
  initAggregatedDataMap,
  mergeAggregatedData,
} from '@/store/connHistory'
import { activeConnections } from '@/store/connections'
import {
  ArrowDownCircleIcon,
  ArrowUpCircleIcon,
  QuestionMarkCircleIcon,
  TrashIcon,
} from '@heroicons/vue/24/outline'
import {
  FlexRender,
  getCoreRowModel,
  getSortedRowModel,
  useVueTable,
  type ColumnDef,
  type SortingState,
} from '@tanstack/vue-table'
import { useVirtualizer } from '@tanstack/vue-virtual'
import { useStorage } from '@vueuse/core'
import dayjs from 'dayjs'
import { computed, h, onMounted, ref } from 'vue'
import { useI18n } from 'vue-i18n'
import DialogWrapper from '../common/DialogWrapper.vue'
import ProxyName from '../proxies/ProxyName.vue'

const { t } = useI18n()
const { showTip } = useTooltip()

enum AutoCleanupInterval {
  Never = 'never',
  Week = 'week',
  Month = 'month',
  Quarter = 'quarter',
}

interface ConnectionHistoryData {
  key: string
  download: number
  upload: number
  count: number
}

const aggregationType = useStorage<ConnectionHistoryType>(
  'cache/connection-history-aggregation-type',
  ConnectionHistoryType.SourceIP,
)
const historicalData = computed(() => aggregatedDataMap.value[aggregationType.value])

// 出站节点维度的完整数据(历史+当前活跃);机场维度由它归并而来,保证机场总量与下钻明细一致
const outboundData = computed<ConnectionHistoryData[]>(() => {
  const current = aggregateConnections(activeConnections.value, ConnectionHistoryType.Outbound)
  return mergeAggregatedData(aggregatedDataMap.value[ConnectionHistoryType.Outbound], current)
})

const groupByAirport = (nodes: ConnectionHistoryData[]): ConnectionHistoryData[] => {
  const map = new Map<string, ConnectionHistoryData>()
  for (const n of nodes) {
    const key = airportOfNode(n.key)
    const existing = map.get(key)
    if (existing) {
      existing.download += n.download
      existing.upload += n.upload
      existing.count += n.count
    } else {
      map.set(key, { key, download: n.download, upload: n.upload, count: n.count })
    }
  }
  return Array.from(map.values())
}

const aggregatedData = computed<ConnectionHistoryData[]>(() => {
  if (aggregationType.value === ConnectionHistoryType.Airport) {
    return groupByAirport(outboundData.value)
  }
  const currentData = aggregateConnections(activeConnections.value, aggregationType.value)
  return mergeAggregatedData(historicalData.value, currentData)
})

const totalStats = computed(() => {
  return aggregatedData.value.reduce(
    (acc, item) => {
      acc.download += item.download
      acc.upload += item.upload
      acc.count += item.count
      return acc
    },
    { download: 0, upload: 0, count: 0 },
  )
})

const aggregateSourceCount = computed(() => aggregatedData.value.length)

const aggregateSourceLabel = computed(() => {
  if (aggregationType.value === ConnectionHistoryType.SourceIP) {
    return t('sourceIP')
  } else if (aggregationType.value === ConnectionHistoryType.Destination) {
    return t('host')
  } else if (aggregationType.value === ConnectionHistoryType.Process) {
    return t('process')
  } else if (aggregationType.value === ConnectionHistoryType.Airport) {
    return t('airport')
  } else {
    return t('outbound')
  }
})

// 单机场累计流量预警阈值(GB){ 机场名: GB };0/留空 = 不预警
const airportWarnMap = useStorage<Record<string, number>>('overview/airport-traffic-warn-gb', {})
const isAirportOverWarn = (name: string, total: number): boolean => {
  const gb = airportWarnMap.value[name]
  return !!gb && gb > 0 && total > gb * 1024 ** 3
}

// 机场下钻:点击机场行 → 弹出该机场的节点明细 + 设置该机场的流量预警
const drillAirport = ref<string | null>(null)
const showDrill = computed({
  get: () => drillAirport.value !== null,
  set: (v: boolean) => {
    if (!v) drillAirport.value = null
  },
})
const drillNodes = computed<ConnectionHistoryData[]>(() => {
  if (!drillAirport.value) return []
  return outboundData.value
    .filter((n) => airportOfNode(n.key) === drillAirport.value)
    .sort((a, b) => b.download + b.upload - (a.download + a.upload))
})
const drillThreshold = computed(() => airportWarnMap.value[drillAirport.value ?? ''] ?? 0)
const setDrillThreshold = (val: string) => {
  const name = drillAirport.value
  if (!name) return
  const num = Number(val)
  const next = { ...airportWarnMap.value }
  if (!num || num <= 0) {
    delete next[name]
  } else {
    next[name] = num
  }
  airportWarnMap.value = next
}
const onRowClick = (row?: { original: ConnectionHistoryData }) => {
  if (row && aggregationType.value === ConnectionHistoryType.Airport) {
    drillAirport.value = row.original.key
  }
}

const columns = computed<ColumnDef<ConnectionHistoryData>[]>(() => {
  const keyColumn: ColumnDef<ConnectionHistoryData> = {
    header: () => aggregateSourceLabel.value,
    id: 'key',
    accessorFn: (row) => row.key,
    cell: ({ row }) => {
      if (aggregationType.value === ConnectionHistoryType.SourceIP) {
        return getIPLabelFromMap(row.original.key)
      } else if (aggregationType.value === ConnectionHistoryType.Destination) {
        return row.original.key
      } else if (aggregationType.value === ConnectionHistoryType.Process) {
        return row.original.key
      } else if (aggregationType.value === ConnectionHistoryType.Airport) {
        const over = isAirportOverWarn(
          row.original.key,
          row.original.download + row.original.upload,
        )
        return h(
          'div',
          { class: ['flex items-center gap-1', over ? 'text-error font-medium' : ''] },
          [row.original.key, over ? h('span', '⚠️') : null],
        )
      } else {
        return h(ProxyName, { name: row.original.key })
      }
    },
  }

  return [
    keyColumn,
    {
      header: () => t('download'),
      id: 'download',
      accessorFn: (row) => row.download,
      cell: ({ row }) => prettyBytesHelper(row.original.download),
      sortingFn: (prev, next) => prev.original.download - next.original.download,
      sortDescFirst: true,
    },
    {
      header: () => t('upload'),
      id: 'upload',
      accessorFn: (row) => row.upload,
      cell: ({ row }) => prettyBytesHelper(row.original.upload),
      sortingFn: (prev, next) => prev.original.upload - next.original.upload,
      sortDescFirst: true,
    },
    {
      header: () => t('totalTraffic'),
      id: 'total',
      accessorFn: (row) => row.download + row.upload,
      cell: ({ row }) => prettyBytesHelper(row.original.download + row.original.upload),
      sortingFn: (prev, next) =>
        prev.original.download +
        prev.original.upload -
        (next.original.download + next.original.upload),
      sortDescFirst: true,
    },
    {
      header: () => t('connectionCount'),
      id: 'count',
      accessorFn: (row) => row.count,
      cell: ({ row }) => row.original.count.toString(),
      sortingFn: (prev, next) => prev.original.count - next.original.count,
      sortDescFirst: true,
    },
  ]
})

const sorting = useStorage<SortingState>('cache/connection-history-sorting', [
  { id: 'download', desc: true },
])

const tanstackTable = useVueTable({
  get data() {
    return aggregatedData.value
  },
  get columns() {
    return columns.value
  },
  state: {
    get sorting() {
      return sorting.value
    },
  },
  onSortingChange: (updater) => {
    if (typeof updater === 'function') {
      sorting.value = updater(sorting.value)
    } else {
      sorting.value = updater
    }
  },
  getSortedRowModel: getSortedRowModel(),
  getCoreRowModel: getCoreRowModel(),
})

const rows = computed(() => {
  return tanstackTable.getRowModel().rows
})

const parentRef = ref<HTMLElement | null>(null)
const rowVirtualizerOptions = computed(() => {
  return {
    count: rows.value.length,
    getScrollElement: () => parentRef.value,
    estimateSize: () => 36,
    overscan: 10,
  }
})

const rowVirtualizer = useVirtualizer(rowVirtualizerOptions)
const virtualRows = computed(() => rowVirtualizer.value.getVirtualItems())
const totalSize = computed(() => rowVirtualizer.value.getTotalSize() + 24)

const showClearDialog = ref(false)
const autoCleanupInterval = useStorage<AutoCleanupInterval>(
  'config/connection-history-auto-cleanup-interval',
  AutoCleanupInterval.Month,
)
const startTime = useStorage<number>('cache/connection-history-stats-start-time', Date.now())
const totalConnectionsTip = computed(() => {
  const dayjsTime = dayjs(startTime.value)

  return t('totalConnectionsTip', {
    statsStartTime: `${dayjsTime.format('YYYY-MM-DD HH:mm')} (${dayjsTime.fromNow()})`,
  })
})
const getCleanupIntervalMs = (interval: AutoCleanupInterval): number => {
  switch (interval) {
    case AutoCleanupInterval.Week:
      return 7 * 24 * 60 * 60 * 1000
    case AutoCleanupInterval.Month:
      return 30 * 24 * 60 * 60 * 1000
    case AutoCleanupInterval.Quarter:
      return 90 * 24 * 60 * 60 * 1000
    case AutoCleanupInterval.Never:
    default:
      return 0
  }
}

const checkAndPerformAutoCleanup = async () => {
  if (autoCleanupInterval.value === AutoCleanupInterval.Never) {
    return
  }

  const now = Date.now()
  const intervalMs = getCleanupIntervalMs(autoCleanupInterval.value)
  const timeSinceLastCleanup = now - startTime.value

  if (timeSinceLastCleanup >= intervalMs) {
    try {
      await clearConnectionHistoryFromIndexedDB()
      await initAggregatedDataMap()
      startTime.value = now
    } catch (error) {
      console.error('Failed to perform auto cleanup:', error)
    }
  }
}

const handleClearHistory = async () => {
  try {
    await clearConnectionHistoryFromIndexedDB()
    await initAggregatedDataMap()
    startTime.value = Date.now()
    showClearDialog.value = false
    showNotification({
      content: t('clearConnectionHistorySuccess'),
      type: 'alert-success',
    })
  } catch (error) {
    console.error('Failed to clear connection history:', error)
    showNotification({
      content: `${t('saveFailed')}: ${error}`,
      type: 'alert-error',
    })
  }
}

onMounted(() => {
  checkAndPerformAutoCleanup()
})
</script>
