---
name: api-contract-design
description: フロントエンド・バックエンド間のAPI契約を定義するためのベストプラクティス。RESTful設計、レスポンス型定義、エラーフォーマット、命名規約を提供します。
---

# API 契約設計スキル

## 概要

フロントエンド・バックエンド間のAPI契約を定義するためのベストプラクティス。

---

## RESTful API 設計原則

- リソース指向のURL設計
- 適切なHTTPメソッドの使用
- ステータスコードの正確な返却
- JSON レスポンス

---

## エンドポイント設計

### MVP エンドポイント

| メソッド | パス | 説明 |
|---|---|---|
| GET | /posts | 下書き投稿一覧取得 |
| GET | /posts/:id | 投稿詳細取得（contents含む） |
| POST | /device-tokens | FCMトークン登録 |

---

## レスポンス型定義

### 一覧レスポンス

```json
{
  "posts": [
    {
      "id": "uuid",
      "status": "draft",
      "scheduledAt": "2026-02-20T10:00:00Z",
      "languageCode": "ja",
      "contentCount": 3,
      "createdAt": "2026-02-15T10:00:00Z"
    }
  ]
}
```

### 詳細レスポンス

```json
{
  "id": "uuid",
  "status": "draft",
  "scheduledAt": "2026-02-20T10:00:00Z",
  "languageCode": "ja",
  "contents": [
    {
      "id": "uuid",
      "body": "generated text",
      "aiIntent": "intent description"
    }
  ],
  "createdAt": "2026-02-15T10:00:00Z"
}
```

---

## エラーレスポンス統一フォーマット

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Post not found"
  }
}
```

### エラーコード一覧

| HTTPステータス | エラーコード | 説明 |
|---|---|---|
| 400 | INVALID_REQUEST | リクエスト不正 |
| 404 | NOT_FOUND | リソース未発見 |
| 409 | CONFLICT | 競合（Lock済み等） |
| 500 | INTERNAL_ERROR | サーバー内部エラー |

---

## JSON フィールド命名規約

- camelCase を使用（Go側で json tag 指定）
- タイムスタンプは ISO 8601 (RFC 3339) 形式
- UUID は文字列として返却

---

## バージョニング方針

- MVP ではバージョニング不要
- 将来的に必要な場合は URL パスプレフィックス（`/v1/posts`）を採用
