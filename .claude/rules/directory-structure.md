# ディレクトリ構成ルール

## モノレポ構成

```
/backend    ← Go (Hexagonal Architecture)
/frontend   ← React Native (Expo)
/infra      ← Terraform
/docs       ← 永続ドキュメント（Docusaurus）
/.claude
  /plans    ← 一時的作業計画
  /skills   ← Skills定義
  /agents   ← SubAgent定義
  /rules    ← Rules定義
```

## 制約

- 各SubAgentは自身の担当ディレクトリのみ変更可能
- ディレクトリ横断変更は design-planner が計画を作成してから実施
- backend/ 配下のアーキテクチャ構成は Hexagonal Architecture に準拠
- frontend/ 配下は Expo Managed Workflow の構成に準拠
- infra/ 配下は Terraform モジュール構成に準拠

## backend/ 内部構成

```
/backend
  /cmd
    /api          ← HTTPサーバー
    /worker       ← バッチWorker
    /migrate      ← goose実行アプリ
  /internal
    /domain
      /entity
      /repository
    /usecase
    /port
    /adapter
      /handler    ← HTTP
      /repository ← sqlc実装
      /external   ← FCM, OpenAI
    /infra
      /db
      /config
  /sql
    /schema
    /query
```

## frontend/ 内部構成

```
/frontend
  /app            ← Expo Router
  /src
    /screens
    /components
    /features
    /hooks
    /api
    /store
    /types
    /i18n
  /assets
```
