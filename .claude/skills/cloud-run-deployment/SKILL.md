---
name: cloud-run-deployment
description: GoアプリケーションをCloud Runにデプロイするためのベストプラクティス。Dockerfile構成、Cloud Run設定、環境変数管理、Cloud Scheduler連携を提供します。
---

# Cloud Run デプロイスキル

## 概要

Go アプリケーションを Cloud Run にデプロイするためのベストプラクティス。

---

## Dockerfile 構成（マルチステージビルド）

```dockerfile
# Build stage
FROM golang:1.23-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /api ./cmd/api

# Runtime stage
FROM gcr.io/distroless/static-debian12
COPY --from=builder /api /api
ENTRYPOINT ["/api"]
```

- API用とWorker用で別Dockerfileを用意する
- distroless イメージで最小構成にする

---

## Cloud Run 設定

### API サービス

- メモリ: 256Mi（MVP）
- CPU: 1
- 最大インスタンス数: 2（MVP）
- タイムアウト: 60s
- 同時実行数: 80

### Worker サービス

- メモリ: 512Mi
- CPU: 1
- 最大インスタンス数: 1
- タイムアウト: 300s
- 同時実行数: 1

---

## 環境変数管理

必要な環境変数:

```
DATABASE_URL        ← Cloud SQL接続文字列
OPENAI_API_KEY      ← Secret Manager参照
FCM_CREDENTIALS     ← Secret Manager参照
PORT                ← Cloud Run自動設定
```

- Secret Manager と Cloud Run の統合を利用
- Terraform で環境変数を定義

---

## Cloud Scheduler 連携

- 30分間隔で Worker の HTTP エンドポイントを呼び出す
- OIDC 認証でセキュアに接続
- タイムアウト: 300s

```
スケジュール: */30 * * * *
ターゲット: Worker Cloud Run サービスの /run エンドポイント
認証: OIDC トークン
```

---

## Cloud SQL 接続

- Cloud SQL Auth Proxy を使用（Cloud Run 統合）
- Unix ソケット接続
- 接続プール設定を適切に管理
