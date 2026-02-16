---
name: structured-logging
description: Goアプリケーションにおける構造化ログ実装のベストプラクティス。slog活用、JSON形式出力、request_id付与、Worker実行ログ、メトリクス埋め込みを提供します。
---

# 構造化ログスキル

## 概要

Go アプリケーションにおける構造化ログ実装のベストプラクティス。

---

## slog パッケージ活用

```go
import "log/slog"

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    }))
    slog.SetDefault(logger)
}
```

---

## JSON 形式ログ出力

Cloud Logging 互換のJSON形式で出力:

```go
slog.Info("post generated",
    slog.String("post_id", post.ID.String()),
    slog.String("language_code", post.LanguageCode),
    slog.Int("content_count", len(contents)),
)
```

出力例:
```json
{
  "time": "2026-02-15T10:00:00Z",
  "level": "INFO",
  "msg": "post generated",
  "post_id": "uuid-value",
  "language_code": "ja",
  "content_count": 3
}
```

---

## request_id 付与パターン

### Middleware で注入

```go
func RequestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        requestID := r.Header.Get("X-Request-ID")
        if requestID == "" {
            requestID = uuid.New().String()
        }
        ctx := context.WithValue(r.Context(), requestIDKey, requestID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### ログ出力時に付与

```go
func LogWithRequestID(ctx context.Context, msg string, attrs ...slog.Attr) {
    requestID, _ := ctx.Value(requestIDKey).(string)
    attrs = append(attrs, slog.String("request_id", requestID))
    slog.LogAttrs(ctx, slog.LevelInfo, msg, attrs...)
}
```

---

## Worker 実行ログ

Worker バッチ処理では以下を明示的にログ出力:

- バッチ開始/終了
- 処理対象件数
- 各Post の処理結果（成功/失敗）
- Lock 取得結果
- FCM 送信結果

```go
slog.Info("worker batch started",
    slog.Time("execution_time", time.Now()),
)
// ... processing ...
slog.Info("worker batch completed",
    slog.Int("processed", processed),
    slog.Int("failed", failed),
    slog.Duration("duration", elapsed),
)
```

---

## メトリクス情報のログ埋め込み

KPI集計に使用するメトリクスは構造化ログに含める。
詳細は `observability` スキルを参照。

### 必須フィールド

- `operation`: 操作名（`post_generation`, `fcm_send`, `openai_call` 等）
- `success`: 成否（bool）
- `duration`: 所要時間
- 操作固有の属性（`post_id`, `token_usage`, `content_count` 等）

### エラーログ

即時対応が必要なエラーは `slog.Error` を使用し、アラート検知対象とする:

```go
slog.Error("worker batch failed",
    slog.String("operation", "worker_batch"),
    slog.Any("error", err),
    slog.Int("failed_count", failedCount),
)
```
