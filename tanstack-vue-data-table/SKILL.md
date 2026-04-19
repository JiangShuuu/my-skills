---
name: tanstack-vue-data-table
description: 使用 TanStack Vue Table（@tanstack/vue-table）時的規範，包含 useVueTable 的寫法、columns 定義、meta 型別、以及狀態處理慣例
---

## 檔案結構

```
src/views/{feature}/
├── {Feature}Page.vue                          # 頁面（管 filters、state、傳 props）
└── components/
    ├── {Feature}Table.vue                     # Table 元件（useVueTable + template）
    ├── {Feature}Actions.vue                   # Row actions（dropdown + modal）
    └── data-table/
        └── {feature}-columns.ts              # 欄位定義 + Meta 型別
```

---

## 第一步：定義 Meta 型別與欄位（columns.ts）

需要在 header/cell 存取頁面層的 state（如 checkbox、sort）時，透過 `table.options.meta` 傳遞，不要用 `provide/inject`。

```ts
// data-table/vehicle-management-columns.ts
import { h } from 'vue'
import type { ColumnDef } from '@tanstack/vue-table'
import { ArrowUp, ArrowDown, ArrowUpDown } from 'lucide-vue-next'
import { Checkbox } from '@/components/ui/checkbox'
import type { VehicleItem } from '@/apis/vehicle_specs'
import VehicleManagementActions from '../VehicleManagementActions.vue'

type SortableColumn =
  | 'licensePlate'
  | 'vehicleModel'
  | 'vehicleCc'
  | 'createdAt'

// 定義 meta 型別，讓欄位能安全取用頁面層的 state 與 callback
export type VehicleManagementMeta = {
  selectedIds: Set<number>
  isAllSelected: boolean
  isSomeSelected: boolean
  toggleRow: (id: number, checked: boolean | 'indeterminate') => void
  toggleAll: (checked: boolean | 'indeterminate') => void
  sortBy: string
  sortOrder: 'asc' | 'desc'
  handleSort: (col: SortableColumn) => void
}

// 抽出 helper，避免每個可排序欄重複相同的 icon 邏輯
const sortHeader = (
  col: SortableColumn,
  label: string,
  meta: VehicleManagementMeta,
) => {
  const icon =
    meta.sortBy === col
      ? meta.sortOrder === 'asc'
        ? h(ArrowUp, { class: 'h-3 w-3' })
        : h(ArrowDown, { class: 'h-3 w-3' })
      : h(ArrowUpDown, { class: 'h-3 w-3 text-gray-400' })

  return h(
    'button',
    {
      class: 'flex items-center gap-1 hover:text-foreground',
      onClick: () => meta.handleSort(col),
    },
    [label, icon],
  )
}

export const columns: ColumnDef<VehicleItem>[] = [
  // checkbox 欄：header 全選，cell 單列
  {
    id: 'select',
    size: 40,
    header: ({ table }) => {
      const meta = table.options.meta as VehicleManagementMeta
      return h('div', { class: 'px-4' }, [
        h(Checkbox, {
          checked: meta.isAllSelected
            ? true
            : meta.isSomeSelected
              ? 'indeterminate'
              : false,
          'onUpdate:checked': (v: boolean | 'indeterminate') =>
            meta.toggleAll(v),
        }),
      ])
    },
    cell: ({ row, table }) => {
      const meta = table.options.meta as VehicleManagementMeta
      return h('div', { class: 'px-4' }, [
        h(Checkbox, {
          checked: meta.selectedIds.has(row.original.vehicleId),
          'onUpdate:checked': (v: boolean | 'indeterminate') =>
            meta.toggleRow(row.original.vehicleId, v),
        }),
      ])
    },
  },
  // row actions 欄
  {
    id: 'actions',
    size: 48,
    header: () => h('div', ''),
    cell: ({ row }) => h(VehicleManagementActions, { row: row.original }),
  },
  // 可排序欄
  {
    accessorKey: 'licensePlate',
    size: 128,
    header: ({ table }) =>
      sortHeader(
        'licensePlate',
        '車牌',
        table.options.meta as VehicleManagementMeta,
      ),
    cell: ({ row }) => h('span', row.original.licensePlate ?? '—'),
  },
  // 純文字欄
  {
    accessorKey: 'vehicleSpec',
    size: 224,
    header: () => h('span', '已綁定車型'),
    cell: ({ row }) => {
      const spec = row.original.vehicleSpec
      return spec
        ? h('span', { class: 'text-green-600' }, spec.vehicleModel)
        : h('span', { class: 'text-gray-400' }, '未綁定')
    },
  },
]
```

**重點：**

- 欄位 `size` 用數字（px），在 Table 元件內透過 inline style 套用，不要在 `TableHead`/`TableCell` 直接寫 Tailwind `w-*`
- 元件事件在 `h()` 中用 `'onUpdate:checked'` 格式，不是 `@update:checked`
- Row actions 元件內可以自帶 modal，不用提升到頁面層

---

## 第二步：Row Actions 元件（Actions.vue）

```vue
<!-- VehicleManagementActions.vue -->
<script lang="ts" setup>
import { ref } from 'vue'
import { MoreHorizontal } from 'lucide-vue-next'
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from '@/components/ui/dropdown-menu'
import { Button } from '@/components/ui/button'
import type { VehicleItem } from '@/apis/vehicle_specs'
import BindVehicleSpecModal from './BindVehicleSpecModal.vue'

const props = defineProps<{ row: VehicleItem }>()

const bindModalRef = ref<InstanceType<typeof BindVehicleSpecModal>>()
</script>

<template>
  <DropdownMenu>
    <DropdownMenuTrigger as-child>
      <Button variant="ghost" class="h-8 w-8 p-0">
        <MoreHorizontal class="h-4 w-4" />
      </Button>
    </DropdownMenuTrigger>
    <DropdownMenuContent align="end">
      <DropdownMenuItem @click="bindModalRef?.handleOpen([props.row])">
        編輯車型綁定
      </DropdownMenuItem>
    </DropdownMenuContent>
  </DropdownMenu>

  <BindVehicleSpecModal ref="bindModalRef" />
</template>
```

---

## 第三步：Table 元件（Table.vue）

```vue
<!-- VehicleManagementTable.vue -->
<script lang="ts" setup>
import { computed } from 'vue'
import { FlexRender, getCoreRowModel, useVueTable } from '@tanstack/vue-table'
import { Skeleton } from '@/components/ui/skeleton'
import Pagination from '@/components/Pagination.vue'
import {
  columns,
  type VehicleManagementMeta,
} from './data-table/vehicle-management-columns'
import type { VehicleItem } from '@/apis/vehicle_specs'

const props = defineProps<{
  data: VehicleItem[]
  totalItems: number
  totalPages: number
  currentPage: number
  isPending: boolean // 初次載入（顯示 skeleton）
  isFetching: boolean // 任意 refetch（顯示 overlay spinner）
  meta: VehicleManagementMeta
}>()

const emit = defineEmits<{ next: []; prev: []; current: [page: number] }>()

const table = useVueTable({
  get data() {
    return props.data
  },
  columns,
  getCoreRowModel: getCoreRowModel(),
  get meta() {
    return props.meta
  },
})

const selectedIds = computed(() => props.meta.selectedIds)
</script>

<template>
  <!-- 初次載入：skeleton -->
  <div v-if="isPending && !data.length" class="px-6 mt-4 space-y-2">
    <Skeleton v-for="n in 10" :key="n" class="w-full h-10 rounded-xl" />
  </div>

  <template v-else>
    <div class="relative w-full overflow-x-auto custom-scrollbar pb-2.5">
      <!-- 換頁/搜尋：overlay spinner -->
      <div
        v-if="isFetching && !isPending"
        class="absolute inset-0 z-10 flex items-center justify-center bg-white/60 rounded-xl"
      >
        <svg
          class="animate-spin h-7 w-7 text-gray-400"
          xmlns="http://www.w3.org/2000/svg"
          fill="none"
          viewBox="0 0 24 24"
        >
          <circle
            class="opacity-25"
            cx="12"
            cy="12"
            r="10"
            stroke="currentColor"
            stroke-width="4"
          />
          <path
            class="opacity-75"
            fill="currentColor"
            d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z"
          />
        </svg>
      </div>

      <Table class="border-b border-gray-02 relative min-w-full">
        <TableHeader class="bg-system-02 border-t border-gray-02">
          <TableRow
            v-for="headerGroup in table.getHeaderGroups()"
            :key="headerGroup.id"
          >
            <TableHead
              v-for="header in headerGroup.headers"
              :key="header.id"
              :style="
                header.column.columnDef.size
                  ? `width: ${header.column.columnDef.size}px`
                  : ''
              "
            >
              <FlexRender
                v-if="!header.isPlaceholder"
                :render="header.column.columnDef.header"
                :props="header.getContext()"
              />
            </TableHead>
          </TableRow>
        </TableHeader>
        <TableBody>
          <template v-if="table.getRowModel().rows?.length">
            <TableRow
              v-for="row in table.getRowModel().rows"
              :key="row.id"
              class="border-gray-02"
              :class="{ 'bg-blue-50': selectedIds.has(row.original.vehicleId) }"
            >
              <TableCell
                v-for="cell in row.getVisibleCells()"
                :key="cell.id"
                :style="
                  cell.column.columnDef.size
                    ? `width: ${cell.column.columnDef.size}px`
                    : ''
                "
              >
                <FlexRender
                  :render="cell.column.columnDef.cell"
                  :props="cell.getContext()"
                />
              </TableCell>
            </TableRow>
          </template>
          <template v-else>
            <TableRow>
              <TableCell :colspan="columns.length" class="h-24 text-center"
                >目前無資料</TableCell
              >
            </TableRow>
          </template>
        </TableBody>
      </Table>
    </div>

    <div class="flex justify-end mt-6">
      <Pagination
        :total-page="totalItems"
        :current-page="currentPage"
        @next="emit('next')"
        @prev="emit('prev')"
        @current="emit('current', $event)"
      />
    </div>
  </template>
</template>
```

---

## 第四步：頁面層（Page.vue）

頁面負責：filters state、selection state、sort state、組合 meta、傳 props。

```vue
<script lang="ts" setup>
import { ref, computed } from 'vue'
import { useGetVehicles, type GetVehiclesFilters, type VehicleItem } from '@/apis/vehicle_specs'
import VehicleManagementTable from './components/VehicleManagementTable.vue'
import BindVehicleSpecModal from './components/BindVehicleSpecModal.vue'
import type { VehicleManagementMeta } from './components/data-table/vehicle-management-columns'

// filters 集中管一個 ref，換頁/搜尋/排序都 spread 更新
const filters = ref<GetVehiclesFilters>({ page: 1, pageSize: 10, sortBy: 'createdAt', sortOrder: 'desc', ... })

const { data, isPending, isFetching } = useGetVehicles(filters)

const vehicleList = computed(() => data.value?.data ?? [])
const totalItems = computed(() => data.value?.total ?? 0)
const totalPages = computed(() => data.value?.totalPages ?? 1)

// selection：用 Set 管 id，每次換頁/搜尋時 clearSelection
const selectedIds = ref(new Set<number>())
const isAllSelected = computed(() => vehicleList.value.length > 0 && vehicleList.value.every((v) => selectedIds.value.has(v.vehicleId)))
const isSomeSelected = computed(() => vehicleList.value.some((v) => selectedIds.value.has(v.vehicleId)) && !isAllSelected.value)
const selectedVehicles = computed<VehicleItem[]>(() => vehicleList.value.filter((v) => selectedIds.value.has(v.vehicleId)))

const toggleRow = (id: number, checked: boolean | 'indeterminate') => {
  const next = new Set(selectedIds.value)
  checked ? next.add(id) : next.delete(id)
  selectedIds.value = next
}
const toggleAll = (checked: boolean | 'indeterminate') => {
  selectedIds.value = checked ? new Set(vehicleList.value.map((v) => v.vehicleId)) : new Set()
}
const clearSelection = () => { selectedIds.value = new Set() }

// sort
type SortableColumn = 'licensePlate' | 'vehicleModel' | 'vehicleCc' | 'createdAt'
const handleSort = (col: SortableColumn) => {
  clearSelection()
  filters.value = filters.value.sortBy === col
    ? { ...filters.value, sortOrder: filters.value.sortOrder === 'asc' ? 'desc' : 'asc', page: 1 }
    : { ...filters.value, sortBy: col, sortOrder: 'asc', page: 1 }
}

// 組合 meta：computed 確保響應式
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

// 換頁
const handleNext = () => { if (filters.value.page < totalPages.value) { clearSelection(); filters.value = { ...filters.value, page: filters.value.page + 1 } } }
const handlePrev = () => { if (filters.value.page > 1) { clearSelection(); filters.value = { ...filters.value, page: filters.value.page - 1 } } }
const handleCurrentPage = (page: number) => { clearSelection(); filters.value = { ...filters.value, page } }

// 批次 modal
const bindModalRef = ref<InstanceType<typeof BindVehicleSpecModal>>()
</script>

<template>
  <!-- 批次工具列 -->
  <div v-if="selectedIds.size > 0" class="flex items-center gap-3">
    <span>已選 {{ selectedIds.size }} 筆</span>
    <Button @click="bindModalRef?.handleOpen(selectedVehicles)"
      >批次綁定車型</Button
    >
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

---

## 注意事項

### isPending vs isFetching

- `isPending`：初次載入（cache 還沒資料），用來顯示 **skeleton**
- `isFetching`：任何 refetch（換頁、搜尋、invalidate），用來顯示 **overlay spinner**
- 兩者分開處理，換頁時保留舊資料避免畫面跳動

### 欄寬固定

- 在 `ColumnDef` 用 `size: number`（px）
- Table 元件用 `:style="... width: ${size}px"` 套用到 `TableHead` 和 `TableCell`
- 不要在 columns.ts 裡寫 Tailwind class，class 無法動態對應欄位

### Selection 跨頁不保留

換頁、搜尋、排序時呼叫 `clearSelection()`，避免使用者誤以為跨頁勾選有效。

### Meta 用 computed 包住

```ts
// ✅ 正確：任一響應式值改變，meta 自動更新
const tableMeta = computed<MyMeta>(() => ({ ... }))

// ❌ 錯誤：純物件不響應式
const tableMeta = { selectedIds: selectedIds.value, ... }
```

### Row Actions modal 自帶 vs 頁面層

- **單筆操作**（edit/delete）：modal 放在 Actions 元件內，不需要提升到頁面
- **批次操作**：modal 放在頁面層，透過 `ref` 呼叫 `handleOpen(rows)`
