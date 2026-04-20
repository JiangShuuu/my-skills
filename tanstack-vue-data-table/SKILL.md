---
name: tanstack-vue-data-table
description: "shadcn-vue DataTable 完整指南。包含元件生成（/data-table 指令）、useVueTable 寫法、columns 定義、meta 型別、selection state 及使用規範。"
---

# shadcn-vue DataTable Skill

## 指令語法

```
/data-table [TableName] [欄位定義]
/data-table Payment id:string amount:number status:enum(pending,processing,success,failed) email:string
```

## 欄位類型

| 類型 | 說明 |
|------|------|
| `string` | 文字 |
| `number` | 數字（格式化） |
| `boolean` | Badge 顯示 |
| `date` | 日期 |
| `enum(...)` | Badge 顯示 |
| `currency` | 貨幣格式 |

## 安裝

```bash
yarn ui table
yarn add @tanstack/vue-table
```

---

## 檔案結構

```
src/views/{feature}/
├── {Feature}Page.vue
└── components/
    ├── {Feature}Table.vue
    ├── {Feature}Actions.vue
    └── data-table/
        └── {feature}-columns.ts
```

---

## 核心規則

- **Meta 不用 provide/inject**，透過 `table.options.meta` 傳遞頁面層 state
- **Meta 必須用 `computed` 包**，純物件不響應式
- **欄寬用 `size: number`（px）**，在 Table 元件用 `:style` 套用，不在 columns.ts 寫 Tailwind `w-*`
- **h() 事件格式**：`'onUpdate:checked'`，不是 `@update:checked`
- **單筆 modal** 放 Actions.vue 內；**批次 modal** 放 Page.vue，透過 ref 呼叫
- **換頁 / 搜尋 / 排序** 時呼叫 `clearSelection()`

### isPending vs isFetching

| | isPending | isFetching |
|---|---|---|
| 觸發時機 | 初次載入（cache 無資料） | 任何 refetch |
| UI | Skeleton | Overlay spinner |
| 舊資料 | 無 | 保留（避免畫面跳動） |

---

## 子檔模板

生成對應檔案時讀取：

- [columns.md](columns.md) — `{feature}-columns.ts`：Meta 型別、sortHeader、ColumnDef 範例
- [actions.md](actions.md) — `{Feature}Actions.vue`：Row actions dropdown + modal
- [table.md](table.md) — `{Feature}Table.vue`：useVueTable + skeleton / overlay / table template
- [page.md](page.md) — `{Feature}Page.vue`：filters、selection、sort state、meta 組合

---

## 需要的 UI 元件

- **shadcn-vue**: Table, Button, Input, Checkbox, Badge, DropdownMenu, Skeleton
- **lucide-vue-next**: ArrowUpDown, ArrowUp, ArrowDown, MoreHorizontal, ChevronDown
