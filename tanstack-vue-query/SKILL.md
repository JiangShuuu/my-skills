---
name: tanstack-vue-query
description: 使用 TanStack Vue Query（@tanstack/vue-query）時的規範，包含 useQuery、useMutation、useInfiniteQuery 的寫法、queryKeys 工廠函式、多響應式參數處理、Optimistic Update、以及狀態處理慣例
---

### 檔案組織

Query / Mutation 統一放在 `src/apis/`，每個資源一個檔案。

```
src/
└── apis/
    └── posts.ts   # 包含 queryKeys、queryFn、usePosts、useCreatePost 等
```

---

### queryKeys

使用工廠函式管理 queryKey，方便批次 invalidate。

```ts
// src/apis/posts.ts
export const postKeys = {
  all: ["posts"] as const,
  lists: () => [...postKeys.all, "list"] as const,
  list: (filters: PostFilters) => [...postKeys.lists(), filters] as const,
  details: () => [...postKeys.all, "detail"] as const,
  detail: (id: number) => [...postKeys.details(), id] as const,
};
```

---

### useQuery

```ts
import { useQuery } from "@tanstack/vue-query";
import { postKeys } from "./posts";
import type { Post } from "@/types";

async function fetchPost(id: number): Promise<Post> {
  const res = await fetch(`/api/posts/${id}`);
  if (!res.ok) throw new Error("Failed to fetch post");
  return res.json();
}

export function usePost(id: MaybeRefOrGetter<number>) {
  return useQuery({
    queryKey: computed(() => postKeys.detail(toValue(id))),
    queryFn: () => fetchPost(toValue(id)),
  });
}
```

- `queryKey` 若含響應式變數，須包在 `computed()` 中
- `queryFn` 內用 `toValue()` 取值

#### 多個響應式參數

當 payload 有多個響應式值時，將所有值都放入 `queryKey: computed(...)` 中。任何一個值改變，query 就會自動重新觸發。

payload 為物件時，建議在 `queryKey` 中**展開每個欄位**，讓觸發條件一目瞭然：

```ts
export function usePosts(filters: MaybeRefOrGetter<PostFilters>) {
  return useQuery({
    queryKey: computed(() => [
      "posts",
      toValue(filters).page,
      toValue(filters).keyword,
    ]),
    queryFn: () => fetchPosts(toValue(filters)),
  });
}
```

- 避免直接傳整個物件到 queryKey（`postKeys.list(toValue(filters))`），否則不易看出哪些欄位會觸發重新查詢
- 參數型別用 `MaybeRefOrGetter<T>` 而非 `Ref<T>`，可同時接受 `ref`、`computed` 或純值

---

#### enabled（條件查詢）

```ts
export function usePost(id: MaybeRefOrGetter<number | undefined>) {
  return useQuery({
    queryKey: computed(() => postKeys.detail(toValue(id)!)),
    queryFn: () => fetchPost(toValue(id)!),
    enabled: computed(() => !!toValue(id)),
  });
}
```

#### select（轉換資料）

```ts
export function usePostTitles() {
  return useQuery({
    queryKey: postKeys.lists(),
    queryFn: fetchPosts,
    select: (data) => data.map((p) => p.title),
  });
}
```

---

### useMutation

```ts
import { useMutation, useQueryClient } from "@tanstack/vue-query";
import { postKeys } from "./posts";

async function createPost(payload: CreatePostPayload): Promise<Post> {
  const res = await fetch("/api/posts", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(payload),
  });
  if (!res.ok) throw new Error("Failed to create post");
  return res.json();
}

export function useCreatePost() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: createPost,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: postKeys.lists() });
    },
  });
}
```

#### 在元件中使用

```vue
<script setup lang="ts">
import { useCreatePost } from "@/apis/posts";

const { mutate: createPost, isPending, isError } = useCreatePost();

function handleSubmit(payload: CreatePostPayload) {
  createPost(payload);
}
</script>
```

#### mutation 後重新拉取頁面資料

執行 POST / PUT / DELETE 後，透過 `invalidateQueries` 讓該頁用到的 GET query 重新觸發，確保資料同步。

**apis 層**：在 `onSuccess` 裡 invalidate 相關 queryKey。

```ts
// src/apis/posts.ts
export function useDeletePost() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (id: number) => fetch(`/api/posts/${id}`, { method: "DELETE" }),
    onSuccess: () => {
      // 讓列表重新 fetch
      queryClient.invalidateQueries({ queryKey: postKeys.lists() });
    },
  });
}

export function useUpdatePost() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: updatePost,
    onSuccess: (_data, variables) => {
      // 讓對應的詳細頁重新 fetch
      queryClient.invalidateQueries({ queryKey: postKeys.detail(variables.id) });
    },
  });
}
```

**元件層**：不需要手動呼叫 refetch，invalidate 後 TanStack Query 會自動重新觸發所有使用中的相關 query。

```vue
<script setup lang="ts">
import { usePost, useUpdatePost } from "@/apis/posts";

const { data } = usePost(postId);         // 這個 query 會在 mutation 成功後自動重新 fetch
const { mutate: updatePost } = useUpdatePost();
</script>
```

若頁面同時用到多個相關 query，可一次 invalidate 整個 namespace：

```ts
// invalidate 所有 posts 相關 query（lists + details）
queryClient.invalidateQueries({ queryKey: postKeys.all });
```

#### Optimistic Update

```ts
export function useUpdatePost() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: updatePost,
    onMutate: async (newPost) => {
      await queryClient.cancelQueries({
        queryKey: postKeys.detail(newPost.id),
      });
      const previous = queryClient.getQueryData(postKeys.detail(newPost.id));
      queryClient.setQueryData(postKeys.detail(newPost.id), newPost);
      return { previous };
    },
    onError: (_err, newPost, context) => {
      queryClient.setQueryData(postKeys.detail(newPost.id), context?.previous);
    },
    onSettled: (_data, _err, newPost) => {
      queryClient.invalidateQueries({ queryKey: postKeys.detail(newPost.id) });
    },
  });
}
```

---

### useInfiniteQuery

```ts
import { useInfiniteQuery } from "@tanstack/vue-query";

export function useInfinitePosts() {
  return useInfiniteQuery({
    queryKey: postKeys.lists(),
    queryFn: ({ pageParam }) => fetchPosts({ page: pageParam }),
    initialPageParam: 1,
    getNextPageParam: (lastPage, _allPages, lastPageParam) =>
      lastPage.hasMore ? lastPageParam + 1 : undefined,
  });
}
```

```vue
<script setup lang="ts">
import { useInfinitePosts } from "@/apis/posts";

const { data, fetchNextPage, hasNextPage, isFetchingNextPage } =
  useInfinitePosts();
const posts = computed(() => data.value?.pages.flatMap((p) => p.items) ?? []);
</script>
```

---

### 狀態處理慣例

```vue
<script setup lang="ts">
const { data, isPending, isError, error } = usePost(postId);
</script>

<template>
  <div v-if="isPending">Loading...</div>
  <div v-else-if="isError">{{ error?.message }}</div>
  <PostDetail v-else :post="data" />
</template>
```

- 使用 `isPending` 而非 `isLoading`（`isLoading` 為舊版 API）
- 不在元件內直接寫 `queryFn`，統一抽到 `apis/`
