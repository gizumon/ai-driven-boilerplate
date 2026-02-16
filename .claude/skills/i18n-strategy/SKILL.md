---
name: i18n-strategy
description: i18nextを使ったReact Nativeアプリの多言語対応ベストプラクティス。設定、翻訳ファイル管理、コンポーネント使用法、RTL対応を提供します。
---

# 多言語対応（i18n）スキル

## 概要

i18next を使ったReact Native アプリの多言語対応ベストプラクティス。

---

## i18next 設定

```typescript
// src/i18n/index.ts
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import * as Localization from 'expo-localization';
import ja from './ja.json';
import en from './en.json';

i18n.use(initReactI18next).init({
  resources: { ja: { translation: ja }, en: { translation: en } },
  lng: Localization.locale.split('-')[0],
  fallbackLng: 'en',
  interpolation: { escapeValue: false },
});

export default i18n;
```

---

## 翻訳ファイル管理

### 構成

```
/src/i18n
  index.ts
  ja.json
  en.json
```

### 翻訳ファイルの規約

- フラットなキー構造（ネストは2階層まで）
- 画面名をプレフィックスに使用

```json
{
  "postList.title": "投稿一覧",
  "postList.empty": "投稿はありません",
  "postDetail.copy": "コピー",
  "postDetail.copied": "コピーしました",
  "common.loading": "読み込み中...",
  "common.error": "エラーが発生しました",
  "common.retry": "再試行"
}
```

---

## コンポーネントでの使用

```typescript
import { useTranslation } from 'react-i18next';

function PostListScreen() {
  const { t } = useTranslation();
  return <Text>{t('postList.title')}</Text>;
}
```

---

## 言語切替フロー

- デバイスのロケール設定に自動追従
- 将来的にアプリ内切替UIを追加可能な設計にする
- language_code はAPIレスポンスに含まれるため、コンテンツ言語とUI言語を分離

---

## RTL レイアウト対応

- `I18nManager` を使用したRTL制御
- Flex レイアウトで `start` / `end` を使用（`left` / `right` を避ける）
- MVP では RTL 非対応でも構造的に対応可能にしておく
