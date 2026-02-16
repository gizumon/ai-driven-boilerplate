# 命名規則

## Go (backend)

- パッケージ名: 小文字単語（`usecase`, `handler`, `entity`）
- 型名: PascalCase（`GeneratePostUseCase`, `PostRepository`）
- 変数/関数: camelCase（非公開）/ PascalCase（公開）
- ファイル名: snake_case（`generate_post.go`, `post_repository.go`）
- テストファイル: `*_test.go`

## TypeScript (frontend)

- コンポーネント: PascalCase（`PostListScreen.tsx`, `ContentCard.tsx`）
- hooks: camelCase + `use` prefix（`usePostStore.ts`, `useNotification.ts`）
- 型/インターフェース: "I" prefix + PascalCase（`IPost`, `IContent`, `IApiResponse`）
- ファイル名: コンポーネントはPascalCase、それ以外はcamelCase
- 定数: UPPER_SNAKE_CASE

## Terraform (infra)

- リソース名: snake_case（`cloud_run_service`, `cloud_sql_instance`）
- 変数名: snake_case
- モジュール名: snake_case
- ファイル名: snake_case（`main.tf`, `variables.tf`, `outputs.tf`）

## データベース

- テーブル名: snake_case 複数形（`posts`, `contents`, `agent_settings`）
- カラム名: snake_case（`post_id`, `created_at`, `language_code`）
