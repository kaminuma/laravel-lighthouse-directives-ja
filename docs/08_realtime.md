# リアルタイム機能（Realtime Features）

---

### @subscription

**概要**  
GraphQLサブスクリプションのブロードキャスト処理方法を定義するディレクティブです。  
WebSocketやPusherを通じて、サーバーからクライアントへのリアルタイムデータ配信を実現します。

**主な用途**
- サブスクリプション型の定義
- クライアントへのリアルタイム通知

**使用例**
```graphql
type Subscription {
    userCreated: User! @subscription(class: "App\\GraphQL\\Subscriptions\\UserCreated")
}
```

**注意点**
- サブスクリプションクラスはNuwave\Lighthouse\Schema\Types\GraphQLSubscriptionのサブクラスである必要があります
- 認証・認可の実装に注意

---

### @broadcast

**概要**  
ミューテーションの結果をGraphQLのサブスクリプション経由でブロードキャスト（配信）するディレクティブです。

**主な用途**
- ミューテーション完了時にクライアントへリアルタイム通知

**使用例**
```graphql
type Mutation {
    createUser(input: UserInput!): User!
        @broadcast(subscription: "userCreated")
}
```

**注意点**
- `subscription`引数で対応するサブスクリプションフィールド名を指定します
- shouldQueueをtrueにするとブロードキャスト自体をキューで非同期送信できます

---

### @event

**概要**  
フィールド解決後にLaravelのイベントをディスパッチするディレクティブです。

**主な用途**
- ミューテーションやクエリの完了後にイベントを発火
- 他システムとの連携や非同期処理のトリガー

**使用例**
```graphql
type Mutation {
    createUser(input: UserInput!): User!
        @event(dispatch: "App\\Events\\UserCreated")
}
```

**注意点**
- `dispatch`引数でイベントクラス名を指定します
- イベントにはリゾルバから得られた値や引数を渡すことができます

---

## 実装のベストプラクティス
- サブスクリプションはパフォーマンスに影響を与える可能性があるため、必要な更新のみを購読
- ブロードキャストチャネルのセキュリティを適切に設定
- イベントの非同期処理を活用してレスポンス時間を改善

## 注意点
- Webソケット接続の適切な管理が必要
- スケーラビリティを考慮したブロードキャストドライバーの選択
- 認証・認可の適切な実装
