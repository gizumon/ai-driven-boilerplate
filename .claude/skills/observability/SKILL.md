---
name: observability
description: KPI取得・メトリクス収集・アラート・ログ運用に関するベストプラクティス。Cloud Monitoring連携、ヘルスチェック、アラート設計、ダッシュボード構成を提供します。
---

# 可観測性スキル（Observability）

## 概要

KPI取得・メトリクス収集・アラート・ログ運用に関するベストプラクティス。

---

## メトリクス収集の原則

### ログベースメトリクス

構造化ログにメトリクス値を含め、Cloud Monitoring のログベースメトリクスとして活用する。

```go
slog.Info("operation_completed",
    slog.String("operation", "post_generation"),
    slog.Duration("duration", elapsed),
    slog.Bool("success", true),
    slog.Int("item_count", count),
)
```

### カスタムメトリクス

Cloud Monitoring API を使用してカスタムメトリクスを送信する場合のパターン:

```go
// メトリクス記録用のインターフェース（Port層）
type MetricsRecorder interface {
    RecordDuration(ctx context.Context, name string, duration time.Duration, labels map[string]string)
    RecordCount(ctx context.Context, name string, value int64, labels map[string]string)
}
```

---

## KPI サマリー集計パターン

### リポジトリ層でのクエリ集計

```sql
-- 直近24時間のWorker実行結果
SELECT
    COUNT(*) FILTER (WHERE status = 'completed') as success_count,
    COUNT(*) FILTER (WHERE status = 'failed') as failure_count,
    AVG(duration_ms) as avg_duration_ms
FROM worker_executions
WHERE executed_at > NOW() - INTERVAL '24 hours';
```

### UseCase 層での集計

```go
type MetricsSummary struct {
    Worker  WorkerMetrics
    FCM     FCMMetrics
    Posts   PostMetrics
    AI      AIMetrics
}

func (uc *GetMetricsSummaryUseCase) Execute(ctx context.Context) (*MetricsSummary, error) {
    // 各リポジトリから直近24時間のメトリクスを取得
    // 集計してサマリーを返却
}
```

---

## ヘルスチェック実装パターン

```go
func (h *HealthHandler) Handle(w http.ResponseWriter, r *http.Request) {
    status := "ok"
    dbStatus := "connected"

    if err := h.db.PingContext(r.Context()); err != nil {
        status = "error"
        dbStatus = "disconnected"
        slog.Error("health check failed", slog.String("component", "db"), slog.Any("error", err))
    }

    json.NewEncoder(w).Encode(map[string]string{
        "status":  status,
        "db":      dbStatus,
        "version": h.version,
    })
}
```

---

## アラート設計パターン

### ログベースアラート

Cloud Monitoring で構造化ログの条件を監視:

```
resource.type="cloud_run_revision"
severity="ERROR"
jsonPayload.operation="post_generation"
```

### Terraform でのアラートポリシー定義

```hcl
resource "google_monitoring_alert_policy" "worker_failure" {
  display_name = "Worker連続失敗アラート"
  combiner     = "OR"

  conditions {
    display_name = "Worker ERROR ログ検出"
    condition_matched_log {
      filter = "resource.type=\"cloud_run_revision\" AND severity=\"ERROR\" AND jsonPayload.operation=\"worker_batch\""
    }
  }

  notification_channels = [google_monitoring_notification_channel.email.name]

  alert_strategy {
    notification_rate_limit {
      period = "300s"
    }
  }
}
```

---

## フロントエンド KPI 表示パターン

### React Query によるポーリング

```typescript
const useMetricsSummary = () => {
  return useQuery({
    queryKey: ['metrics', 'summary'],
    queryFn: () => apiClient.get<MetricsSummary>('/metrics/summary'),
    refetchInterval: 60_000, // 60秒間隔で自動リフレッシュ
  })
}
```

### 異常値判定ロジック

```typescript
const isWorkerHealthy = (metrics: MetricsSummary): boolean => {
  const lastExec = new Date(metrics.worker.lastExecutionAt)
  const minutesSinceLastExec = (Date.now() - lastExec.getTime()) / 60_000
  return minutesSinceLastExec < 60 && metrics.worker.last24hFailureCount === 0
}

const isFCMHealthy = (metrics: MetricsSummary): boolean => {
  const total = metrics.fcm.last24hSentCount + metrics.fcm.last24hFailureCount
  if (total === 0) return true
  return metrics.fcm.last24hFailureCount / total < 0.2
}
```

---

## ログレベル運用基準

| レベル | 用途 | 例 |
|---|---|---|
| ERROR | 即時対応必要 | Worker失敗、DB障害、外部API障害 |
| WARN | 注意が必要 | Lock競合、無効トークン、リトライ発生 |
| INFO | 通常運用ログ | Worker実行、Post生成、FCM送信 |
| DEBUG | 開発時のみ | 詳細リクエスト/レスポンス内容 |

---

## 運用ダッシュボード構成

Cloud Monitoring ダッシュボードに以下を配置:

1. Worker 実行状況（成功/失敗の時系列グラフ）
2. API レイテンシ（p50/p95/p99 の時系列グラフ）
3. FCM 配信状況（送信数/失敗数）
4. OpenAI API 使用状況（トークン使用量、レイテンシ）
5. Cloud Run リソース使用率（CPU、メモリ）
6. エラーログ一覧（直近のERROR/WARNログ）
