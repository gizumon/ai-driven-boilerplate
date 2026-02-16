---
name: fcm-integration
description: Firebase Cloud Messagingを使ったプッシュ通知のサーバーサイド送信とクライアントサイド受信のベストプラクティス。Go/React Nativeの両方をカバーします。
---

# Firebase Cloud Messaging 統合スキル

## 概要

FCM を使ったプッシュ通知のサーバーサイド送信とクライアントサイド受信のベストプラクティス。

---

## サーバーサイド（Go）

### FCM Adapter 実装

```go
type fcmAdapter struct {
    client *messaging.Client
}

func NewFCMAdapter(ctx context.Context, credentialsJSON []byte) (port.Notifier, error) {
    app, err := firebase.NewApp(ctx, nil, option.WithCredentialsJSON(credentialsJSON))
    if err != nil {
        return nil, fmt.Errorf("init firebase app: %w", err)
    }
    client, err := app.Messaging(ctx)
    if err != nil {
        return nil, fmt.Errorf("init messaging client: %w", err)
    }
    return &fcmAdapter{client: client}, nil
}
```

### 送信処理

- device_tokens テーブルから全トークンを取得
- `SendEachForMulticast` で一括送信
- 送信失敗トークンのエラーハンドリング
- 無効トークンの削除処理

### ペイロード設計

```go
message := &messaging.MulticastMessage{
    Tokens: tokens,
    Data: map[string]string{
        "post_id":      postID.String(),
        "scheduled_at": scheduledAt.Format(time.RFC3339),
    },
    Notification: &messaging.Notification{
        Title: "New post generated",
        Body:  "A new AI-generated post is ready for review.",
    },
}
```

---

## クライアントサイド（React Native / Expo）

### トークン取得

```typescript
import * as Notifications from 'expo-notifications';

async function registerForPushNotifications() {
  const { status } = await Notifications.requestPermissionsAsync();
  if (status !== 'granted') return;

  const token = await Notifications.getExpoPushTokenAsync();
  // Backend に送信
  await apiClient.post('/device-tokens', { token: token.data });
}
```

### 通知受信ハンドリング

- フォアグラウンド: Toast表示
- バックグラウンド: システム通知
- タップ時: `post_id` を取得して PostDetailScreen に遷移

---

## エラーハンドリング

- `messaging.IsRegistrationTokenNotRegistered`: トークン削除
- `messaging.IsServerUnavailable`: リトライ
- 一部トークン送信失敗は全体をエラーにしない
