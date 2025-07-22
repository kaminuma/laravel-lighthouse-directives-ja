# スキーマ制御と高度なディレクティブ

---

このカテゴリは、スキーマの動作に対する高度な制御、イベントや機能フラグなどの広範なアプリケーションの懸念事項との統合、および複雑なGraphQLパターンのサポートを提供するディレクティブをカバーしています。

## スカラーとタイプ定義ディレクティブ

### @scalar

スカラー定義を実装するクラスを参照します。

```graphql
scalar DateTime @scalar(class: "Nuwave\\Lighthouse\\Schema\\Types\\Scalars\\DateTime")
```

### @enum

列挙型キーに内部値を割り当て、文字列キーとは異なる内部表現を可能にします。

```graphql
enum UserRole {
  ADMIN @enum(value: 1)
  EDITOR @enum(value: 2)
  CUSTOMER @enum(value: 3)
}
```

### @union

カスタム関数を使用してユニオンの具象型を決定します。

```graphql
union SearchResult @union(resolveType: "App\\GraphQL\\Unions\\SearchResultType") = User | Post | Comment
```

### @interface

カスタムリゾルバを使用してインターフェースの具象型を決定します。

```graphql
interface Node @interface(resolveType: "App\\GraphQL\\Interfaces\\NodeType") {
  id: ID!
}
```

## フィールドリゾルバディレクティブ

### @field

フィールドにリゾルバ関数を割り当て、データフェッチのカスタムロジックを可能にします。

```graphql
type User {
  fullName: String! @field(resolver: "App\\GraphQL\\Resolvers\\UserResolver@getFullName")
}
```

### @method

親オブジェクトのメソッドを呼び出すことでフィールドを解決します。

```graphql
type User {
  fullName: String! @method(name: "getFullName")
}
```

## スキーマ可視性制御ディレクティブ

### @deprecated

GraphQLスキーマの要素をサポート対象外としてマークし、オプションでreasonを指定できます。

```graphql
type User {
  name: String!
  email: String!
  oldField: String @deprecated(reason: "Use `newField` instead.")
  newField: String!
}
```

### @hide

環境に基づいてアノテーションが付けられた要素をスキーマから条件付きで除外します。

```graphql
type User {
  email: String! @hide(env: ["production"])
  password: String! @hide
}
```

### @show

環境に基づいてアノテーションが付けられた要素をスキーマに条件付きで含めます。

```graphql
type User {
  debugInfo: String @show(env: ["local", "development"])
}
```

### @feature

Laravel Pennant機能の状態に応じて、アノテーションが付けられた要素をスキーマに含めます。

```graphql
type Query {
  betaFeature: String! @feature(name: "beta-testers")
}
```

## 名前空間とネスト制御ディレクティブ

### @namespace

他のディレクティブで使用されるデフォルトの名前空間を再定義します。フィールドまたはオブジェクトタイプに適用可能です。

```graphql
type Mutation @namespace(field: "App\\GraphQL\\Mutations") {
  createUser: User! @field(resolver: "CreateUserResolver")
}
```

### @namespaced

懸念事項の分離のためにクエリとミューテーションのネストを可能にするno-opフィールドリゾルバを提供します。

```graphql
type Mutation {
  user: UserMutations! @namespaced
}

type UserMutations {
  create(name: String!): User!
  update(id: ID!, name: String!): User!
  delete(id: ID!): Boolean!
}
```

### @nest

子ArgResolverディレクティブへの呼び出しを委譲するno-opネスト引数リゾルバです。

```graphql
type Mutation {
  createUser(
    name: String!
    contactInput: UserContactInput! @nest
  ): User!
}

input UserContactInput {
  email: String!
  phone: String
}
```

## リアルタイム機能ディレクティブ

### @subscription

クライアントへのサブスクリプションのブロードキャストを処理するクラスを参照します。

```graphql
type Subscription {
  userCreated: User! @subscription(class: "App\\GraphQL\\Subscriptions\\UserCreatedSubscription")
}
```

### @broadcast

ミューテーションの結果を購読しているクライアントにブロードキャストし、サブスクリプションフィールド名を参照します。

```graphql
type Mutation {
  createPost(input: PostInput!): Post!
    @broadcast(subscription: "postCreated")
}
```

### @event

フィールドの解決後にイベントをディスパッチし、解決された値をイベントコンストラクタに渡します。

```graphql
type Mutation {
  createUser(name: String!, email: String!): User!
    @event(dispatch: "App\\Events\\UserCreated")
}
```

## グローバルIDディレクティブ

### @node

Relayのグローバルオブジェクト識別のためにタイプを登録し、モデルまたはカスタムリゾルバを通じて解決できるようにします。

```graphql
type User @node(resolver: "App\\GraphQL\\NodeResolver@resolveUser") {
  id: ID! @globalId
  name: String!
}
```

## パフォーマンス最適化ディレクティブ

### @complexity

実行前にフィールドの複雑度スコアの計算をカスタマイズし、カスタムリゾルバ関数を参照します。

```graphql
type Query {
  posts(
    first: Int!
  ): [Post!]! @complexity(resolver: "App\\GraphQL\\Complexity\\PostsComplexity")
}
```

---

このカテゴリは、Lighthouseの動的なスキーマ操作、拡張性、およびイベント、機能フラグ、リアルタイム更新（サブスクリプション）などの広範なLaravelアプリケーションの懸念事項との深い統合のための高度な機能を示しています。

@deprecated、@hide、@show、@featureなどのディレクティブは、スキーマの進化を管理し、環境やビジネスロジックに基づいてAPIを動的に調整するための強力なメカニズムを提供します。これは、長期的なAPIの保守性、柔軟なデプロイメント、およびA/Bテストや段階的な機能展開などの高度なアーキテクチャパターンにとって不可欠です。

データフェッチングだけでなく、これらのディレクティブはLighthouseをより広範なLaravelアプリケーションアーキテクチャの中心的な部分として機能させます。イベントとの統合（@event）により、GraphQL操作がアプリケーション全体にわたる副作用をトリガーできます。サブスクリプション（@subscription、@broadcast）はリアルタイム機能を可能にします。Relayとの互換性（@node）は、最新のフロントエンドの要求に応えます。

この包括的な統合は、Lighthouseが堅牢でスケーラブルかつ適応性の高いLaravelアプリケーションをGraphQLで構築するための、完全で不可欠なコンポーネントとなることを意味します。

次の章では、Lighthouse PHPディレクティブの総括と結論について述べます。
