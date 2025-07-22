# 認証と認可ディレクティブ

---

これらのディレクティブは、Laravelの認証ガードと認可ポリシーと統合することでGraphQL APIを保護し、認可されたユーザーのみが特定のデータにアクセスしたり、特定のアクションを実行したりできるようにするために不可欠です。

## 認証ディレクティブ

### @auth

クエリの結果として現在認証されているユーザーを返します。使用するガードを指定できます。

```graphql
type Query {
  me: User! @auth
}
```

特定のガードを指定する場合：

```graphql
type Query {
  me: User! @auth(guard: "api")
}
```

### @guard

config/auth.phpからの1つ以上のガードを通じて認証を実行します。フィールドごと、またはオブジェクト内の全てのフィールドに適用できます。

フィールドレベルでの使用:

```graphql
type Query {
  adminUsers: [User!]! @guard(with: ["api:admin"])
}
```

タイプレベルでの使用:

```graphql
type Mutation @guard(with: ["api"]) {
  createPost(title: String!): Post!
  updatePost(id: ID!, title: String): Post!
}
```

## 認可ディレクティブ

### @can*ファミリーディレクティブ

これらのディレクティブは、Laravelポリシーをチェックして、現在のユーザーがフィールドにアクセスする権限があることを確認します。

#### @can

モデルポリシーに対して認可チェックを実行します。

```graphql
type Mutation {
  deletePost(id: ID!): Post! @can(ability: "delete")
}
```

#### @canModel

関連するモデルに対して認可チェックを実行します。

```graphql
type User {
  posts: [Post!]! @canModel(ability: "view")
}
```

#### @canQuery

クエリの実行前に認可チェックを行います。

```graphql
type Query {
  viewableUsers: [User!]! @canQuery(ability: "viewAny", model: "App\\Models\\User")
}
```

#### @canResolved

データが解決された後に認可チェックを実行します。

```graphql
type User {
  email: String! @canResolved(ability: "viewEmail")
}
```

#### @canRoot

ルートタイプレベルで認可チェックを実行します。

```graphql
type Mutation @canRoot(ability: "manage", model: "App\\Models\\User") {
  createUser(name: String!): User!
  updateUser(id: ID!, name: String): User!
}
```

### @whereAuth

現在のユーザーが所有するインスタンスのみを返すようにタイプをフィルタリングします。

```graphql
type Query {
  myPosts: [Post!]! @whereAuth(relation: "user")
}
```

---

認証と認可のための専用ディレクティブ、特にLaravelポリシーと統合された@can*ファミリーの存在は、スキーマレベルでのセキュリティとアクセス制御に対するLighthouseの強い重視を示しています。

これは、「セキュアバイデフォルト」のアプローチを促進し、開発者がデータ定義と並行して権限を宣言的に定義できるようにします。これにより、不正なデータアクセスリスクが大幅に低減され、セキュリティポリシーがスキーマ内で透過的になります。

認証ロジックを各PHPリゾルバに命令的に実装する代わりに、Lighthouseはこれらの重要なセキュリティルールをGraphQLスキーマ内で直接定義することを可能にします。これは、アクセス制御がデータ自体と並行して宣言されることを意味し、APIのセキュリティ体制がスキーマ定義からより透過的で監査可能になります。

このLaravelの堅牢な認可システムの宣言的な統合は、「セキュアバイデザイン」の方法論を奨励します。これにより、カスタムリゾルバ全体に散在する可能性のある認可ロジックに関連するセキュリティの見落としの可能性が減少し、GraphQL API全体のセキュリティと信頼性が向上します。

次の章では、パフォーマンスとキャッシュに関連するディレクティブについて詳しく解説します。
