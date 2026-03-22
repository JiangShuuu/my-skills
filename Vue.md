# Vue 開發規範

## 目錄結構

```
src/
├── apis/           # API endpoints and MSW mocks
├── components/     # Reusable Vue components
├── composables/    # Composition API utilities
├── stores/         # Pinia state management
├── types/          # TypeScript type definitions
├── utils/          # Utility functions
├── plugins/        # Plugin configurations
└── views/          # Page components
    └── home/
        ├── index.vue
        ├── components/   # 此 view 專用元件
        ├── composables/  # 此 view 專用 composables
        ├── types/        # 此 view 專用型別
        └── utils/        # 此 view 專用工具函式
```

## 基本規則

- 時間處理一律使用 `date-fns`
- 盡量避免使用 `watch`
- 單一檔案的 `<template>` 或 `<script>` 超過 300 行時，優先拆分子組件
- .vue 檔案建立時順序為，`<script>`、`<template>`、`<styles>`
- 使用 tanstack-query 寫法時，參考 ./claude/skills/tanstack-vue-query Skill
