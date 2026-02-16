---
name: expo-react-native-patterns
description: Expo Managed WorkflowでのReact Nativeアプリ実装ベストプラクティス。Expo Router、Zustand、React Query、TypeScript型定義、UIパターンを提供します。
---

# Expo React Native 実装パターンスキル

## 概要

Expo Managed Workflow での React Native アプリ実装ベストプラクティス。

---

## Expo Router ナビゲーション設計

### ファイルベースルーティング

```
/app
  _layout.tsx         ← Root Layout
  index.tsx           ← 投稿一覧（ホーム）
  /posts
    [id].tsx          ← 投稿詳細
```

### Layout 構成

```typescript
// app/_layout.tsx
export default function RootLayout() {
  return (
    <QueryClientProvider client={queryClient}>
      <Stack>
        <Stack.Screen name="index" options={{ title: 'Posts' }} />
        <Stack.Screen name="posts/[id]" options={{ title: 'Post Detail' }} />
      </Stack>
    </QueryClientProvider>
  );
}
```

---

## Zustand ストア設計

### 原則

- 1ストア = 1ドメイン
- APIキャッシュは React Query に任せる
- Zustand はUI状態・通知状態など非同期通信以外の状態に使用

```typescript
// src/store/useNotificationStore.ts
interface NotificationStore {
  lastPostId: string | null;
  setLastPostId: (id: string) => void;
}

export const useNotificationStore = create<NotificationStore>((set) => ({
  lastPostId: null,
  setLastPostId: (id) => set({ lastPostId: id }),
}));
```

---

## React Query API通信パターン

### APIクライアント

```typescript
// src/api/client.ts
import axios from 'axios';
import Constants from 'expo-constants';

const apiClient = axios.create({
  baseURL: Constants.expoConfig?.extra?.apiBaseUrl,
  timeout: 10000,
});

export default apiClient;
```

### Query Hook

```typescript
// src/api/posts.ts
export function usePostsQuery() {
  return useQuery({
    queryKey: ['posts'],
    queryFn: async () => {
      const { data } = await apiClient.get<Post[]>('/posts');
      return data;
    },
  });
}

export function usePostDetailQuery(id: string) {
  return useQuery({
    queryKey: ['posts', id],
    queryFn: async () => {
      const { data } = await apiClient.get<Post>(`/posts/${id}`);
      return data;
    },
    enabled: !!id,
  });
}
```

---

## TypeScript 型定義管理

- API レスポンス型は `src/types/api.ts` に集約
- コンポーネント Props 型はコンポーネントファイル内に定義
- 共通UI型は `src/types/` に配置

---

## UI パターン

### Pull to Refresh

```typescript
<FlatList
  data={posts}
  renderItem={renderItem}
  refreshControl={
    <RefreshControl refreshing={isRefetching} onRefresh={refetch} />
  }
  ListEmptyComponent={<EmptyState />}
/>
```

### ローディング・エラー・空状態

- `isLoading`: スケルトン or スピナー表示
- `isError`: エラーメッセージ + リトライボタン
- データが空: 空状態コンポーネント
