# パフォーマンスとキャッシュディレクティブ

---

このカテゴリは、キャッシュ、非同期処理、レートリミットを通じてAPIのパフォーマンスを最適化し、リソース消費を管理し、スケーラビリティを向上させるために設計されたディレクティブに焦点を当てています。

## キャッシュディレクティブ

### @cache

リゾルバの結果をキャッシュします。maxAge（期間）とprivate（ユーザー固有のキャッシュ）のオプションがあります。

```graphql
type Query {
  highTrafficPage: Page! @cache(maxAge: 300)
}
```

ユーザー固有のキャッシュ：

```graphql
type Query {
  personalizedData: [Item!]! @cache(maxAge: 60, private: true)
}
```

### @cacheKey

キャッシュを作成する際にキーとして使用するフィールドを指定します。

```graphql
type User @cacheKey(field: "email") {
  id: ID!
  email: String!
  name: String!
  profile: Profile! @cache
}
```

### @clearCache

親タイプ、IDソース、クリアするフィールドを指定して、タグによってリゾルバキャッシュをクリアします。

```graphql
type Mutation {
  updatePost(id: ID!, title: String): Post! @clearCache(type: "Post")
}
```

特定のフィールドに対する詳細な指定：

```graphql
type Mutation {
  updateUser(id: ID!, name: String): User
    @clearCache(type: "User", idSource: ["id"], field: "posts")
}
```

### @cacheControl

レスポンスのHTTP Cache-Controlヘッダーに影響を与え、maxAgeとscope（PUBLIC/PRIVATE）を制御できます。

```graphql
type Query @cacheControl(maxAge: 1800, scope: PUBLIC) {
  publicData: [PublicItem!]!
}
```

フィールドレベルでの制御：

```graphql
type Query {
  page: Page! @cacheControl(maxAge: 3600, scope: PUBLIC)
  user: User! @cacheControl(maxAge: 60, scope: PRIVATE)
}
```

## 非同期処理ディレクティブ

### @async

ミューテーションの実行をキューに入れられたジョブに遅延させます。このディレクティブはルートミューテーションタイプのフィールドで使用され、実際の実行が非同期で行われる間、すぐにtrueを返します。

```graphql
type Mutation {
  heavyOperation(input: HeavyOperationInput!): Boolean! @async
}
```

## レート制限ディレクティブ

### @throttle

LaravelのThrottleRequestsミドルウェアと同様に、フィールドへのアクセスにレートリミットを設定します。

```graphql
type Query {
  limitedOperation: String! @throttle(maxAttempts: 5, decayMinutes: 10)
}
```

ユーザー固有のレート制限：

```graphql
type Mutation {
  sendMessage(input: MessageInput!): Message!
    @throttle(maxAttempts: 3, decayMinutes: 1, limitByAuthenticatedUser: true)
}
```

---

非同期操作、キャッシュ、およびスロットリングのためのディレクティブの存在は、Lighthouseが本番環境でのパフォーマンスとスケーラビリティを念頭に置いて設計されていることを示しています。

これらの機能により、開発者は複雑なキャッシュやキューイングロジックを手動で記述することなく、APIの応答時間を最適化し、リソース消費を管理できます。これにより、高トラフィックアプリケーションの運用オーバーヘッドが大幅に削減されます。

これらのディレクティブは、キュー管理、キャッシュ無効化戦略、レートリミットアルゴリズムといった複雑なインフラストラクチャの懸念を抽象化します。開発者がこれらのパターンをPHPで手動で実装する必要がある代わりに、GraphQLスキーマでそれらを宣言するだけで済みます。

これにより、開発時間が短縮され、これらの重要なパフォーマンス機能を実装する際のエラーの可能性が減少します。これらの最適化をスキーマ定義に直接組み込むことで、Lighthouseは開発者が最小限の労力でよりスケーラブルで回復力のあるGraphQL APIを構築することを可能にし、要求の厳しいパフォーマンス要件と高いユーザー負荷を持つアプリケーションに適しています。

次の章では、データ変換と入力操作に関連するディレクティブについて詳しく見ていきます。
