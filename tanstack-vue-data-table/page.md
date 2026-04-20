# Page.vue 模板

頁面負責：**filters state、selection state、sort state、組合 meta、傳 props**。

## Script

```ts
import { ref, computed } from 'vue'
import { useGetVehicles, type GetVehiclesFilters, type VehicleItem } from '@/apis/vehicle_specs'
import VehicleManagementTable from './components/VehicleManagementTable.vue'
import BindVehicleSpecModal from './components/BindVehicleSpecModal.vue'
import type { VehicleManagementMeta } from './components/data-table/vehicle-management-columns'

// filters 集中一個 ref，換頁 / 搜尋 / 排序都 spread 更新
const filters = ref<GetVehiclesFilters>({
  page: 1,
  pageSize: 10,
  sortBy: 'createdAt',
  sortOrder: 'desc',
})

const { data, isPending, isFetching } = useGetVehicles(filters)

const vehicleList = computed(() => data.value?.data ?? [])
const totalItems = computed(() => data.value?.total ?? 0)
const totalPages = computed(() => data.value?.totalPages ?? 1)

// --- Selection ---
const selectedIds = ref(new Set<number>())
const isAllSelected = computed(
  () => vehicleList.value.length > 0 && vehicleList.value.every((v) => selectedIds.value.has(v.vehicleId)),
)
const isSomeSelected = computed(
  () => vehicleList.value.some((v) => selectedIds.value.has(v.vehicleId)) && !isAllSelected.value,
)
const selectedVehicles = computed<VehicleItem[]>(
  () => vehicleList.value.filter((v) => selectedIds.value.has(v.vehicleId)),
)

const toggleRow = (id: number, checked: boolean | 'indeterminate') => {
  const next = new Set(selectedIds.value)
  checked ? next.add(id) : next.delete(id)
  selectedIds.value = next
}
const toggleAll = (checked: boolean | 'indeterminate') => {
  selectedIds.value = checked
    ? new Set(vehicleList.value.map((v) => v.vehicleId))
    : new Set()
}
const clearSelection = () => { selectedIds.value = new Set() }

// --- Sort ---
type SortableColumn = 'licensePlate' | 'vehicleModel' | 'vehicleCc' | 'createdAt'

const handleSort = (col: SortableColumn) => {
  clearSelection()
  filters.value =
    filters.value.sortBy === col
      ? { ...filters.value, sortOrder: filters.value.sortOrder === 'asc' ? 'desc' : 'asc', page: 1 }
      : { ...filters.value, sortBy: col, sortOrder: 'asc', page: 1 }
}

// --- Meta（必須用 computed）---
const tableMeta = computed<VehicleManagementMeta>(() => ({
  selectedIds: selectedIds.value,
  isAllSelected: isAllSelected.value,
  isSomeSelected: isSomeSelected.value,
  toggleRow,
  toggleAll,
  sortBy: filters.value.sortBy,
  sortOrder: filters.value.sortOrder,
  handleSort,
}))

// --- 換頁 ---
const handleNext = () => {
  if (filters.value.page < totalPages.value) {
    clearSelection()
    filters.value = { ...filters.value, page: filters.value.page + 1 }
  }
}
const handlePrev = () => {
  if (filters.value.page > 1) {
    clearSelection()
    filters.value = { ...filters.value, page: filters.value.page - 1 }
  }
}
const handleCurrentPage = (page: number) => {
  clearSelection()
  filters.value = { ...filters.value, page }
}

// --- 批次 modal ---
const bindModalRef = ref<InstanceType<typeof BindVehicleSpecModal>>()
```

## Template

```vue
<template>
  <!-- 批次工具列 -->
  <div v-if="selectedIds.size > 0" class="flex items-center gap-3">
    <span>已選 {{ selectedIds.size }} 筆</span>
    <Button @click="bindModalRef?.handleOpen(selectedVehicles)">批次綁定車型</Button>
    <Button variant="outline" @click="clearSelection">清除選取</Button>
  </div>

  <VehicleManagementTable
    :data="vehicleList"
    :total-items="totalItems"
    :total-pages="totalPages"
    :current-page="filters.page"
    :is-pending="isPending"
    :is-fetching="isFetching"
    :meta="tableMeta"
    @next="handleNext"
    @prev="handlePrev"
    @current="handleCurrentPage"
  />

  <BindVehicleSpecModal ref="bindModalRef" @success="clearSelection" />
</template>
```
