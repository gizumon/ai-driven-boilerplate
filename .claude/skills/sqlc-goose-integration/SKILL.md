---
name: sqlc-goose-integration
description: sqlcによる型安全なSQLクエリ生成と、gooseによるマイグレーション管理のベストプラクティス。設定、クエリ記述規約、型マッピングを提供します。
---

# sqlc + goose 統合スキル

## 概要

sqlcによる型安全なSQLクエリ生成と、gooseによるマイグレーション管理のベストプラクティス。

---

## sqlc 設定

`backend/sqlc.yaml` に配置:

```yaml
version: "2"
sql:
  - engine: "postgresql"
    queries: "sql/query"
    schema: "sql/schema"
    gen:
      go:
        package: "sqlcgen"
        out: "internal/adapter/repository/sqlcgen"
        sql_package: "pgx/v5"
        emit_json_tags: true
        emit_empty_slices: true
```

---

## クエリファイル記述規約

`backend/sql/query/` に配置:

```sql
-- name: FindDraftPosts :many
SELECT * FROM posts
WHERE status = 'draft'
ORDER BY created_at DESC;

-- name: FindPostByID :one
SELECT * FROM posts
WHERE id = $1;

-- name: CreatePost :one
INSERT INTO posts (id, status, language_code, scheduled_at, created_at, updated_at)
VALUES ($1, $2, $3, $4, $5, $6)
RETURNING *;

-- name: AcquirePostLock :execrows
UPDATE posts
SET job_locked_at = NOW(), updated_at = NOW()
WHERE id = $1 AND job_locked_at IS NULL;
```

---

## goose マイグレーション

`backend/sql/schema/` に配置:

```sql
-- +goose Up
CREATE TABLE posts (
    id UUID PRIMARY KEY,
    status TEXT NOT NULL DEFAULT 'draft',
    language_code TEXT NOT NULL,
    scheduled_at TIMESTAMP NOT NULL,
    job_locked_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- +goose Down
DROP TABLE posts;
```

### マイグレーション実行

```bash
goose -dir sql/schema postgres "$DATABASE_URL" up
```

---

## PostgreSQL 型マッピング

| PostgreSQL | Go |
|---|---|
| UUID | pgtype.UUID |
| TEXT | string |
| TIMESTAMP | pgtype.Timestamp |
| TIMESTAMP (nullable) | pgtype.Timestamp |
| BOOLEAN | bool |
| TEXT[] | []string |

---

## 生成コードの利用パターン

```go
// Adapter で sqlc 生成コードを利用
func (r *postRepository) FindDraftPosts(ctx context.Context) ([]*entity.Post, error) {
    rows, err := r.queries.FindDraftPosts(ctx)
    if err != nil {
        return nil, fmt.Errorf("find draft posts: %w", err)
    }
    return toPostEntities(rows), nil
}
```

- 生成コード（sqlcgen）を直接 UseCase に渡さない
- Adapter 層で Entity への変換を行う
