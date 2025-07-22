# 認証と認可ディレクティブ

---

これらのディレクティブは、Laravelの認証ガードと認可ポリシーと統合することでGraphQL APIを保護し、認可されたユーザーのみが特定のデータにアクセスしたり、特定のアクションを実行したりできるようにするために不可欠です。

## 認証ディレクティブ

### @auth

現在認証されているユーザーを返します。認証されていない場合はNULLが返されます。このディレクティブは、認証が必要なエンドポイントの保護に役立ちます。

**オプション**:
- `guard`: 使用する認証ガード（デフォルトは設定ファイルで指定されたデフォルトガード）

**例1: 基本的な使用法**
```graphql
type Query {
  me: User @auth
}
```

**例2: 特定のガードを指定**
```graphql
type Query {
  adminDashboard: Dashboard @auth(guard: "admin")
}
```

**例3: 複数のフィールドでの使用**
```graphql
type Query {
  me: User @auth
  mySettings: UserSettings @auth
}
```

**注意事項**:
- Laravelの認証システムと統合されています
- 未認証ユーザーにはNULLが返されます
- 細かい権限制御には@canなどの他のディレクティブとの組み合わせを推奨します

### @guard

特定の認証ガードを使用してフィールドまたはタイプを保護します。認証されていないユーザーがアクセスすると認証エラーが発生します。

**オプション**:
- `with`: 使用する認証ガードの名前（文字列または配列）

**例1: フィールドレベルでの使用**
```graphql
type Query {
  adminUsers: [User!]! @guard(with: ["api:admin"])
}
```

**例2: 複数のガードの指定**
```graphql
type Query {
  adminData: AdminData! @guard(with: ["api", "web"])
}
```

**例3: タイプレベルでの適用（すべてのフィールドに適用）**
```graphql
type Mutation @guard(with: ["api"]) {
  createPost(input: PostInput!): Post!
  updatePost(id: ID!, input: PostInput!): Post!
  deletePost(id: ID!): ID!
}
```

**注意事項**:
- @authとは異なり、認証されていないユーザーはエラーを受け取ります（NULLは返されません）
- 複数のガードを指定した場合、いずれか1つのガードで認証されていれば通過できます
- タイプレベルで適用することで、そのタイプのすべてのフィールドに対してガードを適用できます
- ネストされたタイプやフィールドに対して階層的に適用することができます

## 認可ディレクティブ

### @can

Laravelポリシーを使用して、ユーザーがアクションを実行する権限があるかどうかを確認します。このディレクティブはモデルベースのアクセス制御に最適です。

**オプション**:
- `ability`: チェックするポリシーアビリティ（メソッド名）
- `find`: モデルインスタンスを検索する属性（デフォルトはid）
- `model`: ポリシーを使用するモデルクラス
- `query`: 直接検索するのではなく、クエリを変更する場合はtrue
- `injectArgs`: 追加のパラメータをポリシーに注入する場合はtrue

**例1: 基本的な認可チェック**
```graphql
type Query {
  post(id: ID! @eq): Post @can(ability: "view")
}
```

**例2: クエリを変更するバージョン**
```graphql
type Query {
  posts: [Post!]! @can(ability: "view", query: true)
}
```

**例3: カスタムモデルの指定**
```graphql
type Mutation {
  updateUser(id: ID!, name: String): User
    @can(ability: "update", model: "App\\Models\\User")
}
```

**例4: 削除操作での使用**
```graphql
type Mutation {
  deletePost(id: ID!): Post! @can(ability: "delete")
}
```

**注意事項**:
- アクセスが拒否された場合は`Access denied`例外が発生します
- モデルが見つからない場合は`Model not found`例外が発生します
- ポリシーはモデルクラスに基づいて自動的に検出されます
- `query: true`を使用するとクエリビルダーを変更して、アクセス可能な結果のみを返します
- パフォーマンスのため、リストクエリには`query: true`を使用することを推奨します

```

**例2: モデルクラスの明示的な指定**
```graphql
type Query {
  posts: [Post!]! @can(ability: "viewAny", model: "App\\Models\\Post")
}
```

**例3: カスタムパラメータの使用**
```graphql
type Mutation {
  updatePost(id: ID!, title: String): Post
    @can(ability: "update", find: "id", injectArgs: true)
}
```

**例4: 削除操作での使用**
```graphql
type Mutation {
  deletePost(id: ID!): Post! @can(ability: "delete")
}
```

**注意事項**:
- アクセスが拒否された場合は`Access denied`例外が発生します
- モデルが見つからない場合は`Model not found`例外が発生します
- ポリシーはモデルクラスに基づいて自動的に検出されます

### @canModel

リクエストデータから構築したモデルインスタンス（まだデータベースに存在しない可能性がある）に対して認可チェックを実行します。
主に新規作成のミューテーションで使用され、入力データに基づいてモデルを作成し、ポリシーチェックを行います。

**オプション**:
- `ability`: チェックするポリシーアビリティ（メソッド名）
- `model`: ポリシーを使用するモデルクラス
- `flatten`: 入力引数をフラット化するかどうか（ネストされた入力用）
- `scopes`: 適用するモデルスコープの配列

**例1: モデル作成時の認可**
```graphql
type Mutation {
  createPost(title: String!, content: String!): Post
    @canModel(ability: "create")
}
```

**例2: カスタムモデルクラスの指定**
```graphql
type Mutation {
  publishArticle(title: String!, body: String!): Article
    @canModel(ability: "publish", model: "App\\Models\\Article")
}
```

**例3: ネストされた入力データを使用**
```graphql
type Mutation {
  createPost(input: PostInput!): Post
    @canModel(ability: "create", flatten: true)
}
```

**注意事項**:
- 入力引数からモデルのインスタンスが作成されます（実際にはまだデータベースに保存されません）
- ミューテーションの実行前に権限チェックが行われます
- @createと組み合わせて使用すると効果的です
- `flatten: true`オプションを使用すると、入力引数のネストされた構造をフラットにします

### @canQuery

クエリビルダーを変更してクエリの実行前に認可チェックを適用します。
ユーザーがアクセスできるレコードのみを返すようにデータベースクエリを修正します。

**オプション**:
- `ability`: チェックするポリシーアビリティ（メソッド名）
- `model`: ポリシーを使用するモデルクラス

**例1: クエリ結果のフィルタリング**
```graphql
type Query {
  posts: [Post!]! @all @canQuery(ability: "view")
}
```

**例2: ページネーションと組み合わせて使用**
```graphql
type Query {
  articles: [Article!]! @paginate @canQuery(ability: "viewAny", model: "App\\Models\\Article")
}
```

**例3: リレーションシップのフィルタリング**
```graphql
type User {
  viewablePosts: [Post!]! @hasMany @canQuery(ability: "view")
}
```

**注意事項**:
- データベースレベルでのフィルタリングなのでパフォーマンス面で効率的
- アクセス拒否の場合はエラーを発生させずに、アクセス可能な結果だけを返します
- `@all`や`@paginate`などのディレクティブと組み合わせて使用するのが一般的
- N+1問題を防ぐために、クエリの最適化が重要です

### @canResolved

フィールドの値が解決された後（データが取得された後）に認可チェックを実行します。
フィールド値が解決された後にポリシーチェックを行うため、解決済みの値を使用できます。

**オプション**:
- `ability`: チェックするポリシーアビリティ（メソッド名）
- `model`: ポリシーを使用するモデルクラス（デフォルトは親モデル）
- `args`: ポリシーに追加で渡す引数

**例1: 基本的なフィールドレベルの認可**
```graphql
type User {
  email: String! @canResolved(ability: "viewEmail")
}
```

**例2: 特定の条件付き認可**
```graphql
type User {
  phoneNumber: String @canResolved(ability: "viewPrivateData")
  address: String @canResolved(ability: "viewPrivateData")
}
```

**例3: カスタムモデルとの連携**
```graphql
type Post {
  secretNotes: String @canResolved(ability: "viewNotes", model: "App\\Models\\PostNote")
}
```

**注意事項**:
- フィールドの値が解決された後にポリシーチェックが実行されます
- アクセスが拒否された場合、そのフィールドは`null`に設定されます（エラーは発生しません）
- センシティブなフィールドの保護に特に有用です
- 同じポリシーチェックを複数のフィールドに適用する場合に便利です
- パフォーマンスに注意: すべてのデータがロードされた後にチェックが実行されます

### @canRoot

ルート型（Queryやmutation）全体に対して認可チェックを実行します。ルートレベルでアクセス制御を行い、特定の操作タイプ全体を保護します。

**オプション**:
- `ability`: チェックするポリシーアビリティ（メソッド名）
- `model`: ポリシーを使用するモデルクラス（必須）
- `args`: ポリシーに追加で渡す引数

**例1: 管理者権限のチェック**
```graphql
type Mutation @canRoot(ability: "manage", model: "App\\Models\\User") {
  createUser(name: String!): User!
  updateUser(id: ID!, name: String): User!
  deleteUser(id: ID!): ID!
}
```

**例2: 特定の操作グループの保護**
```graphql
type UserMutations @canRoot(ability: "manageUsers", model: "App\\Models\\User") {
  create(input: UserInput!): User!
  update(id: ID!, input: UserUpdateInput!): User!
  delete(id: ID!): ID!
}
```

**例3: 引数の追加**
```graphql
type AdminQueries @canRoot(
  ability: "viewAdmin", 
  model: "App\\Models\\Dashboard",
  args: ["admin"]
) {
  analytics: Analytics!
  users: [User!]!
  settings: Settings!
}
```

**注意事項**:
- スキーマ内の型全体に対する包括的な権限チェックに最適
- 認可チェックは各フィールドのリゾルバが呼び出される前に実行されます
- アクセスが拒否された場合、そのフィールドへのアクセスはすべてブロックされます

### @whereAuth

現在認証されているユーザーに関連するレコードだけを返すようにクエリをフィルタリングします。
ユーザー所有のリソースのみを表示する場合に便利です。

**オプション**:
- `relation`: ユーザーとの関連を示すリレーション名
- `guard`: 使用する認証ガード（デフォルトは設定ファイルのデフォルトガード）
- `model`: カスタムユーザーモデルクラス名

**例1: 基本的な使用法**
```graphql
type Query {
  myPosts: [Post!]! @whereAuth(relation: "user")
}
```

**例2: カスタムリレーション名の使用**
```graphql
type Query {
  myComments: [Comment!]! @whereAuth(relation: "author")
}
```

**例3: 特定のガードの指定**
```graphql
type Query {
  myOrders: [Order!]! @whereAuth(relation: "customer", guard: "api")
}
```

**注意事項**:
- 認証されていないユーザーの場合は空のリストが返されます
- モデルにはユーザーを参照するリレーションが必要です
- @allや@paginateと組み合わせて使用することが多いです
- パフォーマンスのためにデータベースレベルでフィルタリングが適用されます

---

認証と認可のための専用ディレクティブ、特にLaravelポリシーと統合された@can*ファミリーの存在は、スキーマレベルでのセキュリティとアクセス制御に対するLighthouseの強い重視を示しています。

これは、「セキュアバイデフォルト」のアプローチを促進し、開発者がデータ定義と並行して権限を宣言的に定義できるようにします。これにより、不正なデータアクセスリスクが大幅に低減され、セキュリティポリシーがスキーマ内で透過的になります。

認証ロジックを各PHPリゾルバに命令的に実装する代わりに、Lighthouseはこれらの重要なセキュリティルールをGraphQLスキーマ内で直接定義することを可能にします。これは、アクセス制御がデータ自体と並行して宣言されることを意味し、APIのセキュリティ体制がスキーマ定義からより透過的で監査可能になります。

このLaravelの堅牢な認可システムの宣言的な統合は、「セキュアバイデザイン」の方法論を奨励します。これにより、カスタムリゾルバ全体に散在する可能性のある認可ロジックに関連するセキュリティの見落としの可能性が減少し、GraphQL API全体のセキュリティと信頼性が向上します。

次の章では、パフォーマンスとキャッシュに関連するディレクティブについて詳しく解説します。
