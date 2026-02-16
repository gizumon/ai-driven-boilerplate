---
name: testing-strategy
description: バックエンド（Go）とフロントエンド（React Native）のテスト実装ベストプラクティス。テーブル駆動テスト、モック戦略、テストデータ管理を提供します。
---

# テスト戦略スキル

## 概要

バックエンド（Go）とフロントエンド（React Native）のテスト実装ベストプラクティス。

---

## Go テスト戦略

### テーブル駆動テスト

```go
func TestGeneratePostUseCase(t *testing.T) {
    tests := []struct {
        name    string
        setup   func(*mockPostRepo, *mockAIService)
        wantErr bool
    }{
        {
            name: "successfully generates post",
            setup: func(repo *mockPostRepo, ai *mockAIService) {
                repo.On("AcquireLock", mock.Anything, mock.Anything).Return(true, nil)
                ai.On("GenerateContents", mock.Anything, mock.Anything).Return(contents, nil)
            },
            wantErr: false,
        },
        {
            name: "fails when lock already acquired",
            setup: func(repo *mockPostRepo, ai *mockAIService) {
                repo.On("AcquireLock", mock.Anything, mock.Anything).Return(false, nil)
            },
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // arrange, act, assert
        })
    }
}
```

### モック戦略

- Port（interface）に対してモックを作成
- testify/mock または手動モック
- Adapter のテストは統合テストで実施

### テスト対象優先度

1. UseCase 層（ビジネスロジック）
2. Entity（バリデーション、ドメインルール）
3. Handler（リクエスト/レスポンス変換）
4. Adapter（統合テスト）

---

## React Native テスト戦略

### Jest + Testing Library

```typescript
import { render, screen, fireEvent } from '@testing-library/react-native';

describe('PostListScreen', () => {
  it('renders post list', async () => {
    render(<PostListScreen />);
    expect(await screen.findByText('Post Title')).toBeTruthy();
  });

  it('shows empty state when no posts', () => {
    render(<PostListScreen />);
    expect(screen.getByText('No posts')).toBeTruthy();
  });
});
```

### テスト対象優先度

1. Screen コンポーネント（主要UIフロー）
2. カスタム Hooks
3. Store（Zustand）
4. API クライアント

---

## テストデータ管理

### Go

- テストデータは各テストファイル内に定義
- ファクトリ関数でテストデータ生成

```go
func newTestPost(opts ...func(*entity.Post)) *entity.Post {
    p := &entity.Post{
        ID:           uuid.New(),
        Status:       entity.StatusDraft,
        LanguageCode: "ja",
        ScheduledAt:  time.Now(),
    }
    for _, opt := range opts {
        opt(p)
    }
    return p
}
```

### TypeScript

- MSW（Mock Service Worker）で API モック
- テストデータはファクトリ関数で生成

---

## CI でのテスト実行

- `go test ./...` で全Goテスト実行
- `npx jest` で全フロントエンドテスト実行
- GitHub Actions で自動実行
