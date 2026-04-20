# Actions.vue 模板

單筆 row action。**modal 放在元件內，不提升到頁面層。**

```vue
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

> **批次操作** modal 改放 Page.vue，透過 `ref` 呼叫 `handleOpen(rows)`
