# データベースルール

## ORM

- sqlc のみ使用する（他のORMは禁止）

## マイグレーション

- goose で管理する
- マイグレーションファイルは `backend/sql/schema/` に配置

## クエリ

- sqlc クエリファイルは `backend/sql/query/` に配置
- 生成コードは `backend/internal/adapter/repository/` で使用

## スケジューリング

- cron_expression の使用は禁止
- scheduled_times（timestamp配列）で日時を管理
- Cloud Scheduler は30分間隔の固定実行

## トランザクション

- UseCase 単位で管理する
- Lock取得 → 処理 → 保存 → commit
- 外部API呼び出し（FCM等）はトランザクション外で実行
