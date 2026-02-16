# 依存関係ルール

## Hexagonal Architecture 依存方向

```
Handler → UseCase → Port ← Adapter
```

## 制約

- Entity は外部パッケージに依存しない
- UseCase は Port（interface）経由でのみ Adapter にアクセスする
- Handler に Business Logic を書かない
- 循環依存は禁止
- domain パッケージは adapter パッケージを import しない
- usecase パッケージは adapter パッケージを import しない

## SubAgent間の依存方向

```
design-planner → backend-developer → document-maintainer
design-planner → frontend-developer → document-maintainer
design-planner → infra-developer → document-maintainer
design-planner → qa-engineer
```

- 依存方向は単方向
- 循環依存禁止
- 並列実行時はAPI契約を先に定義
