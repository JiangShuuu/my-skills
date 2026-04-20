# Table.vue 模板

## Script

```ts
import { computed } from 'vue'
import { FlexRender, getCoreRowModel, useVueTable } from '@tanstack/vue-table'
import { Skeleton } from '@/components/ui/skeleton'
import Pagination from '@/components/Pagination.vue'
import { columns, type VehicleManagementMeta } from './data-table/vehicle-management-columns'
import type { VehicleItem } from '@/apis/vehicle_specs'

const props = defineProps<{
  data: VehicleItem[]
  totalItems: number
  totalPages: number
  currentPage: number
  isPending: boolean   // 初次載入 → skeleton
  isFetching: boolean  // 任意 refetch → overlay spinner
  meta: VehicleManagementMeta
}>()

const emit = defineEmits<{ next: []; prev: []; current: [page: number] }>()

const table = useVueTable({
  get data() { return props.data },
  columns,
  getCoreRowModel: getCoreRowModel(),
  get meta() { return props.meta },  // getter 保持響應式
})

const selectedIds = computed(() => props.meta.selectedIds)
```

## Template

```vue
<template>
  <!-- 初次載入：skeleton -->
  <div v-if="isPending && !data.length" class="px-6 mt-4 space-y-2">
    <Skeleton v-for="n in 10" :key="n" class="w-full h-10 rounded-xl" />
  </div>

  <template v-else>
    <div class="relative w-full overflow-x-auto custom-scrollbar pb-2.5">
      <!-- 換頁 / 搜尋：overlay spinner -->
      <div
        v-if="isFetching && !isPending"
        class="absolute inset-0 z-10 flex items-center justify-center bg-white/60 rounded-xl"
      >
        <svg class="animate-spin h-7 w-7 text-gray-400" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
          <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4" />
          <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z" />
        </svg>
      </div>

      <Table class="border-b border-gray-02 relative min-w-full">
        <TableHeader class="bg-system-02 border-t border-gray-02">
          <TableRow v-for="headerGroup in table.getHeaderGroups()" :key="headerGroup.id">
            <TableHead
              v-for="header in headerGroup.headers"
              :key="header.id"
              :style="header.column.columnDef.size ? `width: ${header.column.columnDef.size}px` : ''"
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
                :style="cell.column.columnDef.size ? `width: ${cell.column.columnDef.size}px` : ''"
              >
                <FlexRender :render="cell.column.columnDef.cell" :props="cell.getContext()" />
              </TableCell>
            </TableRow>
          </template>
          <template v-else>
            <TableRow>
              <TableCell :colspan="columns.length" class="h-24 text-center">目前無資料</TableCell>
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
