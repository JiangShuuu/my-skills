# columns.ts 模板

## Meta 型別

```ts
type SortableColumn = 'licensePlate' | 'vehicleModel' | 'vehicleCc' | 'createdAt'

export type VehicleManagementMeta = {
  // selection
  selectedIds: Set<number>
  isAllSelected: boolean
  isSomeSelected: boolean
  toggleRow: (id: number, checked: boolean | 'indeterminate') => void
  toggleAll: (checked: boolean | 'indeterminate') => void
  // sort
  sortBy: string
  sortOrder: 'asc' | 'desc'
  handleSort: (col: SortableColumn) => void
}
```

## sortHeader helper

```ts
const sortHeader = (col: SortableColumn, label: string, meta: VehicleManagementMeta) => {
  const icon =
    meta.sortBy === col
      ? meta.sortOrder === 'asc'
        ? h(ArrowUp, { class: 'h-3 w-3' })
        : h(ArrowDown, { class: 'h-3 w-3' })
      : h(ArrowUpDown, { class: 'h-3 w-3 text-gray-400' })

  return h(
    'button',
    { class: 'flex items-center gap-1 hover:text-foreground', onClick: () => meta.handleSort(col) },
    [label, icon],
  )
}
```

## ColumnDef 範例

### Checkbox 欄

```ts
{
  id: 'select',
  size: 40,
  header: ({ table }) => {
    const meta = table.options.meta as VehicleManagementMeta
    return h('div', { class: 'px-4' }, [
      h(Checkbox, {
        checked: meta.isAllSelected ? true : meta.isSomeSelected ? 'indeterminate' : false,
        'onUpdate:checked': (v: boolean | 'indeterminate') => meta.toggleAll(v),
      }),
    ])
  },
  cell: ({ row, table }) => {
    const meta = table.options.meta as VehicleManagementMeta
    return h('div', { class: 'px-4' }, [
      h(Checkbox, {
        checked: meta.selectedIds.has(row.original.vehicleId),
        'onUpdate:checked': (v: boolean | 'indeterminate') => meta.toggleRow(row.original.vehicleId, v),
      }),
    ])
  },
},
```

### 可排序欄

```ts
{
  accessorKey: 'licensePlate',
  size: 128,
  header: ({ table }) => sortHeader('licensePlate', '車牌', table.options.meta as VehicleManagementMeta),
  cell: ({ row }) => h('span', row.original.licensePlate ?? '—'),
},
```

### 純文字欄（含條件樣式）

```ts
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
```

### Actions 欄

```ts
{
  id: 'actions',
  size: 48,
  header: () => h('div', ''),
  cell: ({ row }) => h(VehicleManagementActions, { row: row.original }),
},
```

### 其他常用 cell

```ts
// 貨幣
cell: ({ row }) => {
  const v = Number.parseFloat(row.getValue('amount'))
  return h('div', { class: 'text-right font-medium' },
    new Intl.NumberFormat('zh-TW', { style: 'currency', currency: 'TWD' }).format(v))
}

// 狀態 Badge
cell: ({ row }) => {
  const s = row.getValue('status') as string
  const map: Record<string, string> = { success: 'default', pending: 'secondary', failed: 'destructive' }
  return h(Badge, { variant: map[s] || 'default' }, () => s)
}

// 布林 Badge
cell: ({ row }) => {
  const v = row.getValue('isActive')
  return h(Badge, { variant: v ? 'default' : 'secondary' }, () => (v ? '是' : '否'))
}

// 日期
cell: ({ row }) => h('div', new Date(row.getValue('createdAt')).toLocaleDateString('zh-TW'))
```

## 完整 imports

```ts
import { h } from 'vue'
import type { ColumnDef } from '@tanstack/vue-table'
import { ArrowUp, ArrowDown, ArrowUpDown } from 'lucide-vue-next'
import { Checkbox } from '@/components/ui/checkbox'
import { Badge } from '@/components/ui/badge'
```
