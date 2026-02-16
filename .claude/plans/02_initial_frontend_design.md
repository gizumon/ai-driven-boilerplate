# 📱 フロントエンド開発エージェント向け実装指示書（MVP）

あなたはモバイルアプリ実装エージェントです。  
本プロジェクトは **モノレポ構成** です。ディレクトリ構成を厳守してください。

---

# 1. モノレポ構成

```
/frontend        ← 本指示の対象
/backend         ← Go (Hexagonal Architecture)
/infra           ← Terraform
/docs            ← 全体ドキュメント
```

---

# 2. フロントエンド技術スタック

## 基本方針

- クロスプラットフォーム（iOS / Android）
- Push通知必須
- 軽量MVP
- 将来の多言語対応を考慮
- BackendはCloud Run上のAPI

---

## 使用技術

- React Native（Expo Managed Workflow）
- TypeScript（必須）
- 状態管理: Zustand
- API通信: React Query
- Push通知: Firebase Cloud Messaging（FCM）
- ルーティング: Expo Router
- i18n: i18next

---

# 3. ディレクトリ構成（frontend/）

```
/frontend
  /app
  /src
    /screens
      DashboardScreen.tsx
      PostListScreen.tsx
      PostDetailScreen.tsx
    /components
    /features
      /posts
      /agentSettings
    /hooks
    /api
      client.ts
      posts.ts
      metrics.ts
    /store
      usePostStore.ts
    /types
      api.ts
    /i18n
      index.ts
      ja.json
      en.json
  /assets
  app.json
  package.json
  tsconfig.json
```

---

# 4. MVP機能要件

---

## 4.0 KPIダッシュボード画面

### API

```
GET /metrics/summary
```

### 表示内容

- Worker稼働状況（最終実行日時、直近24時間の成功/失敗数）
- Post生成状況（直近24時間の生成数、下書き総数）
- FCM通知状況（直近24時間の送信数、失敗数、アクティブデバイス数）
- AI使用状況（直近24時間のトークン使用量、平均レイテンシ）
- システムヘルス（API / DB 接続状態）

### UI要件

- カード形式でKPIを表示
- 異常値（失敗数 > 0、長時間Worker未実行等）は赤/オレンジで警告表示
- Pull to Refresh
- 最終更新日時表示
- ローディング / エラー状態表示

### 型定義（src/types/api.ts に追加）

```typescript
export interface MetricsSummary {
  worker: {
    lastExecutionAt: string
    last24hSuccessCount: number
    last24hFailureCount: number
    avgGenerationDurationMs: number
  }
  fcm: {
    last24hSentCount: number
    last24hFailureCount: number
    activeDeviceCount: number
  }
  posts: {
    totalDraftCount: number
    totalPublishedCount: number
    last24hCreatedCount: number
  }
  ai: {
    last24hTokenUsage: number
    avgOpenaiLatencyMs: number
  }
}

export interface HealthStatus {
  status: 'ok' | 'error'
  db: 'connected' | 'disconnected'
  version: string
}
```

---

## 4.1 Push通知受信

### 要件

- FCMトークン取得
- Backendへ `/device-tokens` APIで送信
- 通知タップ時に該当Post詳細画面へ遷移

### 通知ペイロード例

```
{
  "post_id": "uuid",
  "scheduled_at": "2026-02-20T10:00:00Z"
}
```

---

## 4.2 投稿一覧画面

### API

```
GET /posts
```

### 表示内容

- status = draft のみ
- scheduled_at
- language_code
- contents件数
- 作成日時

### UI要件

- 新着順表示
- Pull to Refresh
- ローディング表示
- 空状態表示
- エラー状態表示

---

## 4.3 投稿詳細画面

### API

```
GET /posts/:id
```

### 表示内容

- contents（複数表示）
- ai_intent（各コンテンツごとに表示）
- scheduled_at
- language_code

### 機能

- コンテンツ切り替えUI
- Clipboardコピー
- 「コピーしました」Toast表示

---

# 5. 型定義例（src/types/api.ts）

```
export interface Content {
  id: string
  body: string
  aiIntent: string
}

export interface Post {
  id: string
  status: 'draft' | 'published'
  scheduledAt: string
  languageCode: string
  contents: Content[]
  createdAt: string
}
```

---

# 6. 状態管理設計

Zustand:

- usePostStore
- useNotificationStore
- useMetricsStore

React Query:

- APIキャッシュ専用
- 永続化しない

---

# 7. 通知フロー

1. WorkerがPost生成
2. BackendがFCM送信
3. アプリ受信
4. post_id取得
5. PostDetailScreenへ遷移

---

# 8. エラーハンドリング方針

- APIエラー → Toast表示
- ネットワークエラー → 再試行ボタン表示
- 401 → 再取得処理（将来OAuth対応前提）

---

# 8.1 運用監視UI方針

- KPIダッシュボードをホーム画面（タブ最初の画面）とする
- 異常検知の視覚的フィードバック:
  - Worker失敗数 > 0 → 赤バッジ
  - Worker最終実行から60分超 → 警告カード
  - FCM失敗率 > 20% → オレンジ警告
  - DB接続断 → 全画面エラーバナー
- ダッシュボードデータは自動リフレッシュ（60秒間隔）

---

# 9. 多言語対応設計

- i18next導入
- languageCodeベースでUI言語切替可能設計
- RTL対応可能なレイアウト構造
- 将来追加言語を容易に追加可能

---

# 10. UI設計原則

- シンプル
- ダークモード対応
- テーマ切替可能設計
- Flexレイアウト中心
- アクセシビリティ考慮（フォントスケール対応）

---

# 11. セキュリティ

- API Base URLはenv管理
- 秘密情報を含めない
- FCMサーバーキーは保持しない

---

# 12. 実装順序

1. Expo初期化
2. TypeScript設定
3. Navigation設定
4. FCM連携
5. APIクライアント実装
6. KPIダッシュボード画面
7. 投稿一覧画面
8. 投稿詳細画面
9. Clipboard機能
10. i18n導入
11. テスト実施

---

# 13. docs更新義務

frontend実装に伴い下記の様な設計ドキュメントを更新すること：

```
/docs/docs/frontend/architecture.md
/docs/docs/frontend/api-integration.md
/docs/docs/frontend/details/push-flow.md
/docs/docs/frontend/details/i18n.md
```

---

# 14. MVP完了条件

- 通知が届く
- 投稿一覧取得できる
- バリエーション表示可能
- コピー可能
- iOS/Android両対応確認
- KPIダッシュボードが表示される
- 異常値の警告表示が機能する
- ダッシュボード自動リフレッシュが動作する

---

# 15. 将来拡張を考慮した設計

- OAuth連携拡張可能
- 自動投稿機能追加余地
- 投稿履歴画面追加可能
- 言語追加容易性
- デザインシステム導入余地確保
- KPIグラフ表示（時系列推移）追加余地
- アラート履歴画面追加余地
- 通知設定画面（アラート閾値カスタマイズ）追加余地

---

以上に従い、  
**frontend/ 配下のみ実装すること。**
