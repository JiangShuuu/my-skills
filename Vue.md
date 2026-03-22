# CLAUDE.md

## 需求需要建立檔案時參考以下路徑

```
src/
├── apis/          # API endpoints and MSW mocks
├── components/    # Reusable Vue components
├── composables/   # Composition API utilities
├── stores/        # Pinia state management
├── views/         # Page components
  ├─── home/
      ├─── index.vue # basic file
      ├─── components # this view use components
      ├─── utils # this view use utils
      ├─── types # this view use types
      ├─── composables # this view use composables
├── types/         # TypeScript type definitions
├── utils/         # Utility functions
└── plugins/       # Plugin configurations
```

## 基本規則

- 與時間相關的需求，一率使用 date-fns 套件
- 盡量避免使用 watch
- 當 template 或者 script 在一個檔案內超過 300 行時，優先考慮拆分子組件
