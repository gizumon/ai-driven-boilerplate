---
name: golang-hexagonal-architecture
description: Go言語でHexagonal Architecture（Ports & Adapters）を実装するためのベストプラクティス。Entity設計、Port/Adapter分離、DI、トランザクション管理のパターンを提供します。
---

# Go + Hexagonal Architecture 実装スキル

## 概要

Go言語でHexagonal Architecture（Ports & Adapters）を実装するためのベストプラクティス集。

---

## Entity 設計原則

- Entity は純粋なドメインオブジェクトとする
- 外部パッケージ（DB、HTTP等）への依存を禁止
- 値オブジェクトを活用してドメイン知識を表現する
- バリデーションロジックは Entity 内に実装する
- Entity のコンストラクタで不変条件を保証する

```go
// Good: Entity は外部依存なし
type Post struct {
    ID           uuid.UUID
    Status       PostStatus
    LanguageCode string
    ScheduledAt  time.Time
    JobLockedAt  *time.Time
    CreatedAt    time.Time
    UpdatedAt    time.Time
}

func NewPost(languageCode string, scheduledAt time.Time) (*Post, error) {
    if languageCode == "" {
        return nil, ErrEmptyLanguageCode
    }
    return &Post{
        ID:           uuid.New(),
        Status:       StatusDraft,
        LanguageCode: languageCode,
        ScheduledAt:  scheduledAt,
        CreatedAt:    time.Now(),
        UpdatedAt:    time.Now(),
    }, nil
}
```

---

## Port（Interface）設計

- Port は `internal/port/` に定義
- Repository Port と External Service Port を分離
- インターフェースは利用側（UseCase）の視点で設計する

```go
// internal/port/post_repository.go
type PostRepository interface {
    FindDraftPosts(ctx context.Context) ([]*entity.Post, error)
    FindByID(ctx context.Context, id uuid.UUID) (*entity.Post, error)
    Create(ctx context.Context, post *entity.Post) error
    AcquireLock(ctx context.Context, id uuid.UUID) (bool, error)
}
```

---

## UseCase 実装パターン

- 1 UseCase = 1 ビジネスユースケース
- Port 経由で Adapter にアクセス
- トランザクション管理は UseCase 層の責務

```go
type GeneratePostUseCase struct {
    postRepo    port.PostRepository
    contentRepo port.ContentRepository
    aiService   port.AIService
    notifier    port.Notifier
    txManager   port.TransactionManager
}

func NewGeneratePostUseCase(
    postRepo port.PostRepository,
    contentRepo port.ContentRepository,
    aiService port.AIService,
    notifier port.Notifier,
    txManager port.TransactionManager,
) *GeneratePostUseCase {
    return &GeneratePostUseCase{...}
}
```

---

## Adapter 実装パターン

- Adapter は Port の実装
- 外部依存（DB、API）をカプセル化
- エラーは Domain エラーにラップして返す

```go
// internal/adapter/repository/post_repository.go
type postRepository struct {
    queries *sqlcgen.Queries
}

func NewPostRepository(queries *sqlcgen.Queries) port.PostRepository {
    return &postRepository{queries: queries}
}
```

---

## DI（依存性注入）パターン

- main関数（cmd/）で全依存を組み立てる
- コンストラクタインジェクションを使用
- DIコンテナは使わずシンプルに手動配線

```go
func main() {
    db := connectDB()
    queries := sqlcgen.New(db)

    postRepo := repository.NewPostRepository(queries)
    aiService := external.NewOpenAIAdapter(cfg.OpenAIKey)
    notifier := external.NewFCMAdapter(cfg.FCMCredentials)

    generateUC := usecase.NewGeneratePostUseCase(postRepo, aiService, notifier)

    handler := handler.NewPostHandler(generateUC)
    // ...
}
```

---

## トランザクション管理

- トランザクションは UseCase で制御
- Port として TransactionManager を定義
- FCM等の外部呼び出しはトランザクション外で実行

```go
type TransactionManager interface {
    WithTransaction(ctx context.Context, fn func(ctx context.Context) error) error
}
```
