# 🧠 バックエンド開発エージェント向け実装指示書（MVP）

あなたはバックエンド実装エージェントです。  
本プロジェクトは **モノレポ構成** です。ディレクトリ構成を厳守してください。

---

# 1. モノレポ構成

```
/frontend        ← React Native (Expo)
/backend         ← 本指示の対象（Golang）
/infra           ← Terraform
/docs            ← 全体ドキュメント
```

---

# 2. 技術スタック

## 言語

- Golang（常に最新安定版を使用）

## アーキテクチャ

- Hexagonal Architecture（Ports & Adapters）
- DDD-lite（Entity / UseCase 分離）

## DB

- Cloud SQL (PostgreSQL)
- ORM: sqlc
- Migration: goose（Hexagonal内に統合）

## インフラ

- Cloud Run
- Cloud Scheduler（30分間隔）
- Firebase Cloud Messaging（通知送信）
- Terraform管理（/infra）

## CI/CD

- GitHub Actions → Cloud Runデプロイ

---

# 3. アーキテクチャ構成（backend/）

```
/backend
  /cmd
    /api
    /worker
    /migrate        ← goose実行アプリ
  /internal
    /domain
      /entity
      /repository
    /usecase
    /port
    /adapter
      /handler      ← HTTP
      /repository   ← sqlc実装
      /external     ← FCM, OpenAI
    /infra
      /db
      /config
  /sql
    /schema
    /query
  go.mod
```

---

# 4. DB設計

---

## posts

```
id (uuid, PK)
status (text) -- draft | published
language_code (text)
scheduled_at (timestamp)
job_locked_at (timestamp nullable)
created_at
updated_at
```

### 仕様

- status = draft がMVP対象
- job_locked_at により重複実行防止
- 30分以上lockされている場合は再取得可能

---

## contents

```
id (uuid, PK)
post_id (uuid, FK)
body (text)
ai_intent (text)
created_at
```

### 仕様

- 1 post に複数 content
- ai_intent にAI生成意図を保存

---

## agent_settings

```
id (uuid, PK)
language_code (text)
prompt_template (text)
scheduled_times (timestamp[])
active (boolean)
created_at
updated_at
```

### 仕様

- cronは使用しない
- scheduled_times に複数日時指定可能
- Cloud Schedulerは30分間隔実行

---

## device_tokens

```
id (uuid)
token (text)
created_at
```

---

# 5. バッチ実行仕様

## Cloud Scheduler

- 30分間隔でWorker起動

## Worker処理フロー

1. 現在時刻 ±30分のscheduled_times取得
2. 未実行post作成
3. job_locked_at更新（トランザクション）
4. AIコンテンツ生成（複数パターン）
5. contents保存
6. FCM通知送信

---

# 6. 重複実行防止仕様

- job_locked_at IS NULL のみ取得
- UPDATE ... WHERE job_locked_at IS NULL
- rows affected = 1 の場合のみ実行
- 30分超lockは再実行対象

---

# 7. API仕様（MVP）

---

## GET /posts

- status=draftのみ返却
- 新着順

---

## GET /posts/:id

- contents含めて返却

---

## POST /device-tokens

- FCM token登録

---

## GET /health

- ヘルスチェック（DB接続確認含む）

---

## GET /metrics/summary

- KPIサマリー返却（直近24時間）

---

# 8. UseCase一覧

- GeneratePostUseCase
- ListPostsUseCase
- GetPostDetailUseCase
- RegisterDeviceTokenUseCase
- GetMetricsSummaryUseCase

---

# 9. 外部Adapter

---

## OpenAI Adapter

- コンテンツ複数生成
- ai_intent生成

---

## FCM Adapter

- device_tokens全件へ通知
- post_id含める

---

# 10. Migrationアプリ

`/cmd/migrate`

- goose実行専用
- infraから呼び出し可能
- Hexagonalの外部Adapter扱い

---

# 11. 言語対応設計

- language_codeを全postに保持
- AI生成時に言語指定
- 将来多言語UI連携前提

---

# 12. トランザクションポリシー

- GeneratePostは単一Tx
- Lock取得→生成→保存→commit
- FCMはTx外

---

# 13. エラーハンドリング

- Domainエラー定義
- Adapterエラーはラップ
- HTTPは適切なステータス返却

---

# 14. ログ設計

- 構造化ログ（JSON）
- request_id付与
- Worker実行ログ明示

---

# 14.1 KPI / メトリクス設計

## 収集対象KPI

### Worker メトリクス
- Post生成成功率（成功数 / 試行数）
- Post生成所要時間（OpenAI API応答時間含む）
- Worker 1回あたりの処理件数
- Lock 競合回数
- Lock タイムアウト回数（30分超再取得）

### API メトリクス
- エンドポイント別リクエスト数
- エンドポイント別レイテンシ（p50, p95, p99）
- エラーレート（4xx / 5xx）

### FCM メトリクス
- 通知送信成功数 / 失敗数
- 登録デバイストークン数
- 無効トークン検出数

### AI生成メトリクス
- OpenAI API 呼び出し回数
- OpenAI API レイテンシ
- トークン使用量
- 生成コンテンツ数（バリエーション数）

## 実装方針

- Cloud Run 標準メトリクスを活用（CPU、メモリ、リクエスト数）
- カスタムメトリクスは Cloud Monitoring API で送信
- 構造化ログ内にメトリクス値を含め、ログベースメトリクスとしても活用

### メトリクス記録パターン

```go
slog.Info("post_generation_completed",
    slog.String("post_id", post.ID.String()),
    slog.Duration("generation_duration", elapsed),
    slog.Duration("openai_duration", openaiElapsed),
    slog.Int("content_count", len(contents)),
    slog.Int("token_usage", tokenUsage),
    slog.Bool("success", true),
)
```

---

# 14.2 ヘルスチェック / 運用エンドポイント

## API

```
GET /health          ← Cloud Run ヘルスチェック
GET /metrics/summary ← KPIサマリー取得（フロントエンド用）
```

## /health レスポンス

```json
{
  "status": "ok",
  "db": "connected",
  "version": "1.0.0"
}
```

## /metrics/summary レスポンス

```json
{
  "worker": {
    "last_execution_at": "2026-02-15T10:00:00Z",
    "last_24h_success_count": 48,
    "last_24h_failure_count": 0,
    "avg_generation_duration_ms": 3200
  },
  "fcm": {
    "last_24h_sent_count": 48,
    "last_24h_failure_count": 0,
    "active_device_count": 5
  },
  "posts": {
    "total_draft_count": 120,
    "total_published_count": 0,
    "last_24h_created_count": 48
  },
  "ai": {
    "last_24h_token_usage": 15000,
    "avg_openai_latency_ms": 2800
  }
}
```

## UseCase 追加

- GetMetricsSummaryUseCase

---

# 14.3 アラート設計

## アラート条件

| アラート名 | 条件 | 重要度 |
|---|---|---|
| Worker連続失敗 | 直近3回連続で全Post生成失敗 | Critical |
| Worker未実行 | 60分以上Worker実行ログなし | Critical |
| FCM配信失敗率 | 失敗率 > 20% | Warning |
| API高エラーレート | 5xx率 > 5%（5分間） | Critical |
| APIレイテンシ劣化 | p95 > 3秒（5分間） | Warning |
| OpenAI API障害 | OpenAI呼び出し連続3回失敗 | Critical |
| DB接続障害 | ヘルスチェック失敗 | Critical |
| Lock競合多発 | 30分間にLock競合5回以上 | Warning |

## 実装方針

- Cloud Monitoring アラートポリシーで定義（Terraform管理）
- アラート通知先: メール + Cloud Monitoring ダッシュボード
- ログベースアラート（構造化ログの severity=ERROR を条件に）
- Worker失敗時は `slog.Error` で出力し、ログベースアラートで検知

---

# 14.4 運用ログ分類

## ログレベル運用基準

| レベル | 用途 |
|---|---|
| ERROR | 即時対応が必要（Worker失敗、DB障害、FCM障害） |
| WARN | 注意が必要（Lock競合、無効トークン検出、リトライ発生） |
| INFO | 通常運用ログ（Worker実行、Post生成、FCM送信） |
| DEBUG | 開発時のみ（詳細なリクエスト/レスポンス内容） |

## 必須ログポイント

- Worker バッチ開始 / 終了（処理件数、所要時間）
- Post 生成成功 / 失敗（post_id、エラー理由）
- OpenAI API 呼び出し（レイテンシ、トークン数）
- FCM 送信結果（成功数、失敗数、無効トークン数）
- Lock 取得 / 解放 / 競合
- API リクエスト（エンドポイント、ステータスコード、レイテンシ）

---

# 15. docs更新義務

以下を必ず更新：

```
/docs/docs/backend/architecture.md
/docs/docs/backend/db-schema.md
/docs/docs/backend/batch-design.md
/docs/docs/backend/api-spec.md
/docs/docs/backend/error-handling.md
/docs/docs/backend/observability.md
```

---

# 16. MVP完了条件

- Cloud RunでAPI稼働
- Workerが30分毎に実行
- posts + contents保存
- FCM通知送信
- 重複実行防止動作確認
- sqlc生成コード動作確認
- gooseマイグレーション成功
- ヘルスチェックエンドポイント応答確認
- KPIサマリーエンドポイント応答確認
- 構造化ログにメトリクス情報が含まれることを確認
- Cloud Monitoring アラートポリシーが設定済み

---

# 17. 禁止事項

- ORMをsqlc以外使わない
- cron_expression使用禁止
- BusinessロジックをHandlerに書かない
- Entityを外部依存させない

---

# 18. 将来拡張考慮

- OAuth連携（X OAuth）
- 自動投稿
- 投稿履歴管理
- 管理画面
- AIモデル差替可能設計

---

以上に従い、  
**backend/ 配下のみ実装すること。**
