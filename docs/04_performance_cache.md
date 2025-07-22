# パフォーマンスとキャッシュディレクティブ

---

このカテゴリは、キャッシュ、非同期処理、レートリミットを通じてAPIのパフォーマンスを最適化し、リソース消費を管理し、スケーラビリティを向上させるために設計されたディレクティブに焦点を当てています。

## キャッシュディレクティブ

### @cache

リゾルバの結果をキャッシュします。一度取得したデータを一定期間再利用することで、パフォーマンスを向上させます。

**オプション**:
- `maxAge`: キャッシュの有効期間（秒単位）
- `private`: ユーザー固有のキャッシュにする場合はtrue（デフォルトはfalse）

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

キャッシュを作成する際にキーとして使用するフィールドを指定します。モデルのキャッシュにおいて、IDではなく別のフィールドをキーとして使用したい場合に役立ちます。

**オプション**:
- `field`: キャッシュキーとして使用するフィールド名

```graphql
type User @cacheKey(field: "email") {
  id: ID!
  email: String!
  name: String!
  profile: Profile! @cache
}
```

### @clearCache

タグによってリゾルバキャッシュをクリアします。データが変更されたときに関連するキャッシュを無効化するのに最適です。

**オプション**:
- `type`: キャッシュをクリアするモデルタイプ名
- `idSource`: IDを取得するソース（デフォルトは親モデルのIDフィールド）
- `field`: 特定のフィールドのキャッシュのみをクリアする場合に指定

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

レスポンスのHTTP Cache-Controlヘッダーに影響を与え、クライアント側のキャッシュ動作を制御します。

**オプション**:
- `maxAge`: キャッシュの最大寿命（秒単位）
- `scope`: キャッシュのスコープ（PUBLIC/PRIVATE）
- `inheritMaxAge`: 親フィールドからmaxAgeを継承するかどうか（boolean）

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

ミューテーションの実行をキューに入れられたジョブに遅延させます。長時間実行される操作をバックグラウンドで実行するために使用します。

**オプション**:
- `queue`: 使用するキュー名（Laravelのキューシステム）
- `connection`: 使用するキュー接続名
- `maxRealTime`: 同期実行の最大時間（ミリ秒）

```graphql
type Mutation {
  heavyOperation(input: HeavyOperationInput!): Boolean! @async
}
```

## レート制限ディレクティブ

### @throttle

LaravelのThrottleRequestsミドルウェアと同様に、フィールドへのアクセスにレートリミットを設定します。APIの乱用を防ぐのに役立ちます。

**オプション**:
- `maxAttempts`: 許容される最大リクエスト回数
- `decayMinutes`: レート制限がリセットされるまでの時間（分単位）
- `limitByAuthenticatedUser`: ユーザーごとに制限するかどうか（boolean）
- `limitByIp`: IPアドレスごとに制限するかどうか（boolean）

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
