# ドキュメントルール

## 基本原則

- 1ファイル1責務
- 他ドキュメントとの重複禁止
- 対象読者を明確にする
- 本文に変更履歴を書かない（Gitで管理）

## ドキュメント基盤

- `docs/` は **Docusaurus** で構築する
- Docusaurus の設定ファイル（`docusaurus.config.ts`, `sidebars.ts` 等）は `docs/` 直下に配置
- ドキュメントコンテンツは `docs/docs/` 配下に Markdown で記述する
- サイドバーの順序制御には Docusaurus の `sidebar_position` frontmatter を使用する

## 配置先

- 一時情報（作業計画、TODO）: `.claude/plans/`
- 永続情報（仕様、設計）: `docs/docs/`（Docusaurus コンテンツディレクトリ）

## docs/docs/ 構成例

```
/docs
  docusaurus.config.ts
  sidebars.ts
  package.json
  /docs
    /backend <--- Backend全体に関する設計
      _category_.json
      architecture.md
      db-schema.md
      batch-design.md
      api-spec.md
      error-handling.md
    /frontend <--- Frontend全体に関する設計
      _category_.json
      architecture.md
      api-integration.md
      /details <--- 特定機能の詳細設計
        _category_.json
        push-flow.md
        i18n.md
    /infra <--- インフラ全体に関する設計
      _category_.json
      architecture.md
      terraform-structure.md
      deployment.md
```

* ファイル名に連番プレフィックスは付けない。表示順序は `sidebar_position` frontmatter または `_category_.json` で制御する
* 各カテゴリディレクトリに `_category_.json` を配置し、ラベルと順序を定義する

## 禁止事項

- SubAgent に ノウハウを書かない
- Skills に一時仕様を書かない
- Rules にベストプラクティスを書かない
- 同一設計を複数ドキュメントに書かない
