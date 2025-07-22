# Eloquentとデータベース操作ディレクティブ

---

このカテゴリには、LaravelのEloquent ORMと基盤となるデータベースと直接対話し、一般的なデータフェッチ、作成、更新、削除操作を容易にするディレクティブが含まれます。

## 検索と取得ディレクティブ

### @all

指定されたモデルのすべてのインスタンスを取得します。デフォルトではモデル名はフィールド名と同じ型名から推測されますが、明示的に指定することも可能です。

**オプション**:
- `model`: 使用するEloquentモデルのクラス名 (例: `"App\\Models\\User"`)
- `builder`: クエリビルダを変更するメソッドを含むカスタムビルダークラス

**例1: デフォルトモデル名の使用**
```graphql
type Query {
  users: [User!]! @all
}
```

**例2: 明示的なモデル指定**
```graphql
type Query {
  people: [User!]! @all(model: "App\\Models\\User")
}
```

**例3: カスタムビルダーの使用**
```graphql
type Query {
  activeUsers: [User!]! @all(builder: "App\\GraphQL\\Builders\\UserBuilder@active")
}
```

**注意事項**:
- 返される結果の量に注意してください。データセットが大きい場合は`@paginate`の使用を検討してください。
- 認可チェックを適用する場合は`@can`ディレクティブと組み合わせることができます。

### @find

提供された引数に基づいて単一のEloquentモデルを検索します。デフォルトでは主キー（通常はID）に基づいて検索しますが、カスタム条件での検索も可能です。

**オプション**:
- `model`: 使用するEloquentモデルのクラス名 (例: `"App\\Models\\User"`)
- `by`: 検索に使用するカラム名 (デフォルト: モデルの主キー)
- `scopes`: モデルスコープのリスト、検索前にモデルに適用されます

**例1: 主キーによる基本的な検索**
```graphql
type Query {
  user(id: ID! @eq): User @find
}
```

**例2: カスタムカラムによる検索**
```graphql
type Query {
  userByEmail(email: String! @eq): User @find(by: "email")
}
```

**例3: 複数パラメータとモデル指定**
```graphql
type Query {
  person(id: ID! @eq): User! @find(model: "App\\Models\\User")
}
```

**例4: モデルスコープの適用**
```graphql
type Query {
  activeUser(id: ID! @eq): User @find(scopes: ["active"])
}
```

**注意事項**:
- 結果が見つからない場合はnullを返します（非Nullフィールドの場合はエラーになります）
- 複数の結果が見つかる場合はエラーになります（一意の結果を返すクエリにのみ使用してください）

### @first

提供された引数に基づいてモデルの最初のインスタンスを取得します。複数の検索条件を組み合わせることができ、結果の最初の項目を返します。

**オプション**:
- `model`: 使用するEloquentモデルのクラス名 (例: `"App\\Models\\User"`)
- `scopes`: 検索前に適用するモデルスコープの配列

**例1: 基本的な使用法**
```graphql
type Query {
  userByEmail(email: String! @eq): User @first
}
```

**例2: 複数の条件による検索**
```graphql
type Query {
  oldestActiveUser(is_active: Boolean! @eq): User @first
}
```

**例3: モデル指定とスコープ適用**
```graphql
type Query {
  featuredPost(category: String @eq): Post @first(model: "App\\Models\\Post", scopes: ["published", "featured"])
}
```

**注意事項**:
- `@first`は`orderBy`ディレクティブと組み合わせると特に効果的です
- 結果が見つからない場合はnullを返します
- `@find`とは異なり、複数の結果が一致する場合でもエラーにならず、最初の結果だけを返します

## 集計ディレクティブ

### @aggregate

モデルのコレクションに対して集計関数を実行し、その結果を返します。データベースレベルで計算を行うため、大量のレコードを効率的に処理できます。

**オプション**:
- `model`: 使用するEloquentモデルのクラス名 (例: `"App\\Models\\User"`)
- `field`: 集計を行うモデルのカラム
- `function`: 使用する集計関数 (AVG、SUM、MIN、MAX、COUNT)
- `relation`: 集計を行うリレーション名（モデルフィールドの代わりに使用）
- `builder`: カスタムクエリビルダーのクラス
- `scopes`: 適用するモデルスコープの配列

**例1: 単純な平均値の計算**
```graphql
type Query {
  averageUserAge: Float! @aggregate(model: "App\\Models\\User", field: "age", function: AVG)
}
```

**例2: リレーションを使用した集計**
```graphql
type Query {
  totalOrderAmount: Float! @aggregate(relation: "orders", field: "amount", function: SUM)
}
```

**例3: 集計関数の組み合わせ**
```graphql
type Query {
  productAnalytics: ProductStats @aggregate(model: "App\\Models\\Product") {
    min_price: Float! @aggregate(function: MIN, field: "price")
    max_price: Float! @aggregate(function: MAX, field: "price")
    avg_price: Float! @aggregate(function: AVG, field: "price")
    total_inventory: Int! @aggregate(function: SUM, field: "stock")
  }
}
```

**注意事項**:
- 対応するPHPの集計関数と同様に動作します
- 条件フィルタリングと組み合わせることができます
- リレーション集計は効率的なサブクエリを使用します

### @count

モデルまたはリレーションのレコード数を効率的に取得します。

**オプション**:
- `model`: カウント対象のEloquentモデルのクラス名 (例: `"App\\Models\\User"`)
- `relation`: カウント対象のリレーション名
- `column`: 特定のカラムをカウント対象とする場合に指定
- `distinct`: 重複を除外するかどうか (デフォルト: false)
- `builder`: カスタムクエリビルダーのクラス
- `scopes`: 適用するモデルスコープの配列

**例1: モデルインスタンスの数を取得**
```graphql
type Query {
  userCount: Int! @count(model: "App\\Models\\User")
}
```

**例2: リレーションの数を取得**
```graphql
type User {
  id: ID!
  name: String!
  postCount: Int! @count(relation: "posts")
}
```

**例3: 特定条件に基づくカウント**
```graphql
type Query {
  activeUserCount: Int! 
    @count(model: "App\\Models\\User", scopes: ["active"])
}
```

**例4: ユニークな値のカウント**
```graphql
type Query {
  uniqueRolesCount: Int! 
    @count(model: "App\\Models\\User", column: "role", distinct: true)
}
```

**注意事項**:
- 大規模なデータセットでも効率的に動作します
- SQL の `COUNT()` 関数に直接マッピングされます
- クエリ条件と組み合わせて使用できます

## リレーションシップディレクティブ

### @belongsTo

EloquentのBelongsToリレーションシップを通じてフィールドを解決します。親モデルの参照を取得するために使用します。

**オプション**:
- `relation`: 使用するリレーションシップメソッドの名前（デフォルトはフィールド名と同じ）
- `model`: リレーション先のモデルクラス名（自動推論できない場合に指定）
- `softDeletes`: 論理削除されたレコードを含めるかどうか（true/false）

**例1: 基本的な使用法**
```graphql
type Post {
  author: User! @belongsTo
}
```

**例2: カスタムリレーション名の指定**
```graphql
type Comment {
  writtenBy: User! @belongsTo(relation: "author")
}
```

**例3: 論理削除されたレコードを含める**
```graphql
type Post {
  category: Category @belongsTo(softDeletes: true)
}
```

**注意事項**:
- フィールド名とリレーションシップメソッド名が異なる場合は`relation`オプションを使用する
- `BelongsTo`リレーションシップは親モデルへの参照を表す (例: 投稿から著者へ、コメントから投稿へ)
- 外部キーは通常「親モデル名の単数形_id」の規則に従う

### @belongsToMany

多対多のリレーションシップを解決します。中間（ピボット）テーブルを介して複数の関連モデルを取得します。

**オプション**:
- `relation`: 使用するリレーションシップメソッドの名前（デフォルトはフィールド名と同じ）
- `type`: ページネーションタイプ（`paginator`、`simple`、`connection`）
- `defaultCount`: ページネーション時のデフォルト項目数（デフォルト: 15）
- `maxCount`: ページネーション時の最大項目数
- `model`: リレーション先のモデルクラス名（自動推論できない場合に指定）
- `scopes`: 適用するモデルスコープの配列
- `softDeletes`: 論理削除されたレコードを含めるかどうか（true/false）
- `pivot`: 中間テーブルから追加のカラムを取得するかどうか（true/false）
- `edgeType`: リレーションのエッジタイプ名を明示的に指定
- `withTimestamps`: ピボットテーブルのタイムスタンプを含めるかどうか

**例1: 基本的な使用法**
```graphql
type User {
  roles: [Role!]! @belongsToMany
}
```

**例2: ピボットテーブルのデータを含める**
```graphql
type User {
  roles: [Role!]! @belongsToMany(pivot: true)
}

# 自動生成されるピボットタイプ
type RoleUserPivot {
  created_at: DateTime
  role_id: ID!
  user_id: ID!
}
```

**例3: ページネーションの使用**
```graphql
type User {
  roles(first: Int!, page: Int): RolePaginator! @belongsToMany(type: "paginator")
}
```

**例4: カスタムピボットアクセサーの使用**
```graphql
type User {
  permissions: [Permission!]! @belongsToMany(relation: "permissionsViaRoles")
}
```

**注意事項**:
- 中間テーブル（ピボット）からデータを取得する場合は`pivot: true`を指定
- 大量のデータを扱う場合はページネーションの使用を検討
- 多対多リレーションではピボットテーブルが自動的に使用される

### @hasMany

1対多のリレーションシップを解決します。親モデルに関連付けられた子モデルのコレクションを取得します。

**オプション**:
- `relation`: 使用するリレーションシップメソッドの名前（デフォルトはフィールド名と同じ）
- `type`: ページネーションタイプ（`paginator`、`simple`、`connection`）
- `defaultCount`: ページネーション時のデフォルト項目数（デフォルト: 15）
- `maxCount`: ページネーション時の最大項目数
- `model`: リレーション先のモデルクラス名（自動推論できない場合に指定）
- `scopes`: 適用するモデルスコープの配列
- `softDeletes`: 論理削除されたレコードを含めるかどうか（true/false）

**例1: 基本的な使用法**
```graphql
type User {
  posts: [Post!]! @hasMany
}
```

**例2: ページネーションの使用**
```graphql
type User {
  posts(first: Int!, page: Int): PostPaginator! @hasMany(type: "paginator")
}
```

**例3: カスタムリレーション名の指定**
```graphql
type User {
  writtenArticles: [Post!]! @hasMany(relation: "posts")
}
```

**例4: スコープの適用**
```graphql
type User {
  publishedPosts: [Post!]! @hasMany(scopes: ["published"])
}
```

**注意事項**:
- 外部キーは通常「親モデル名の単数形_id」の規則に従う
- 大量のデータを扱う場合はページネーションの使用を検討
- フィルタリングディレクティブと組み合わせて使用可能

### @hasOne

1対1のリレーションシップを解決します。親モデルに関連付けられた単一の子モデルを取得します。

**オプション**:
- `relation`: 使用するリレーションシップメソッドの名前（デフォルトはフィールド名と同じ）
- `model`: リレーション先のモデルクラス名（自動推論できない場合に指定）
- `softDeletes`: 論理削除されたレコードを含めるかどうか（true/false）
- `scopes`: 適用するモデルスコープの配列

**例1: 基本的な使用法**
```graphql
type User {
  profile: Profile! @hasOne
}
```

**例2: カスタムリレーション名の指定**
```graphql
type User {
  userSettings: Settings! @hasOne(relation: "settings")
}
```

**例3: スコープの適用**
```graphql
type User {
  activeProfile: Profile @hasOne(scopes: ["active"])
}
```

**注意事項**:
- `hasOne`は親モデルから子モデルへの1対1のリレーション
- 外部キーは通常「親モデル名の単数形_id」の規則に従う
- 結果がnullの場合にエラーを避けるためには非Null宣言を避ける

### @hasManyThrough

中間モデルを介して関連する複数のモデルを取得する「多段リレーション」を解決します。
（例：Country→User→Postで国から投稿へのリレーション）

**オプション**:
- `relation`: 使用するリレーションシップメソッドの名前（デフォルトはフィールド名と同じ）
- `type`: ページネーションタイプ（`paginator`、`simple`、`connection`）
- `defaultCount`: ページネーション時のデフォルト項目数
- `maxCount`: ページネーション時の最大項目数
- `model`: リレーション先のモデルクラス名（自動推論できない場合に指定）
- `first`: 仲介モデルへのリレーション名
- `second`: 最終モデルへのリレーション名
- `scopes`: 適用するモデルスコープの配列
- `softDeletes`: 論理削除されたレコードを含めるかどうか（true/false）

**例1: 基本的な使用法**
```graphql
type Country {
  posts: [Post!]! @hasManyThrough
}
```

**例2: 明示的なリレーションパスの指定**
```graphql
type Country {
  userPosts: [Post!]! @hasManyThrough(first: "users", second: "posts")
}
```

**例3: ページネーションの使用**
```graphql
type Country {
  posts(first: Int!, page: Int): PostPaginator! @hasManyThrough(type: "paginator")
}
```

**注意事項**:
- このリレーションは2つの段階のリレーションを「飛び越えて」アクセスするために使用
- 実際には中間テーブルを経由してクエリが実行される
- 複雑なリレーションのためSQLの効率に注意

### @hasOneThrough

中間モデルを介して関連する単一のモデルを取得する「多段リレーション」を解決します。
@hasManyThroughの1対1バージョンです。

**オプション**:
- `relation`: 使用するリレーションシップメソッドの名前（デフォルトはフィールド名と同じ）
- `model`: リレーション先のモデルクラス名（自動推論できない場合に指定）
- `first`: 仲介モデルへのリレーション名
- `second`: 最終モデルへのリレーション名
- `scopes`: 適用するモデルスコープの配列
- `softDeletes`: 論理削除されたレコードを含めるかどうか（true/false）

**例1: 基本的な使用法**
```graphql
type User {
  latestPost: Post @hasOneThrough
}
```

**例2: 明示的なリレーションパスの指定**
```graphql
type Mechanic {
  carOwner: Owner @hasOneThrough(first: "car", second: "owner")
}
```

**例3: スコープの適用**
```graphql
type User {
  latestPublishedPost: Post @hasOneThrough(scopes: ["published"])
}
```

**注意事項**:
- 通常、2つの段階のリレーションを「飛び越えて」単一の関連レコードにアクセスするために使用
- 複雑なリレーションではモデル内で明示的に定義することを推奨
- 多対一、一対一のリレーションを組み合わせた場合に有効

### @morphMany

ポリモーフィック（多相）な1対多のリレーションシップを解決します。
単一のモデルから複数の異なるモデルタイプへの関連を定義できます。

**オプション**:
- `relation`: 使用するリレーションシップメソッドの名前（デフォルトはフィールド名と同じ）
- `type`: ページネーションタイプ（`paginator`、`simple`、`connection`）
- `model`: リレーション先のモデルクラス名（自動推論できない場合に指定）
- `scopes`: 適用するモデルスコープの配列
- `softDeletes`: 論理削除されたレコードを含めるかどうか（true/false）

**例1: 基本的な使用法**
```graphql
type Post {
  comments: [Comment!]! @morphMany
}
```

**例2: ページネーションの使用**
```graphql
type Post {
  comments(first: Int!, page: Int): CommentPaginator! @morphMany(type: "paginator")
}
```

**例3: スコープの適用**
```graphql
type Post {
  approvedComments: [Comment!]! @morphMany(scopes: ["approved"])
}
```

**注意事項**:
- ポリモーフィックリレーションではデータベースに*_type（クラス名）と*_id（主キー）のカラムが必要
- 複数の異なるモデル（例：Post、Video）から同じモデル（例：Comment）への関連付けが可能
- 関連するモデルは `MorphTo` リレーションを持つ必要がある

### @morphOne

ポリモーフィック（多相）な1対1のリレーションシップを解決します。
単一のモデルから1つの関連モデルへのポリモーフィックな関連を定義できます。

**オプション**:
- `relation`: 使用するリレーションシップメソッドの名前（デフォルトはフィールド名と同じ）
- `model`: リレーション先のモデルクラス名（自動推論できない場合に指定）
- `scopes`: 適用するモデルスコープの配列
- `softDeletes`: 論理削除されたレコードを含めるかどうか（true/false）

**例1: 基本的な使用法**
```graphql
type Post {
  image: Image @morphOne
}
```

**例2: カスタムリレーション名の指定**
```graphql
type Post {
  featuredImage: Image @morphOne(relation: "mainImage")
}
```

**例3: スコープの適用**
```graphql
type Post {
  publishedImage: Image @morphOne(scopes: ["published"])
}
```

**注意事項**:
- ポリモーフィックリレーションではデータベースに*_type（クラス名）と*_id（主キー）のカラムが必要
- 複数の異なるモデルから同じモデルへの1対1の関連付けが可能
- @morphManyの1対1バージョン

### @morphTo

ポリモーフィックなリレーションシップの「親側」を解決します。
複数の異なるモデルタイプの中から、このモデルが関連付けられているモデルを取得します。

**オプション**:
- `relation`: 使用するリレーションシップメソッドの名前（デフォルトはフィールド名と同じ）
- `type`: 型名の解決方法（型名のマッピング）
- `scopes`: 適用するモデルスコープの配列

**例1: 基本的な使用法**
```graphql
type Comment {
  commentable: Commentable! @morphTo
}
```

**例2: カスタムタイプマッピング**
```graphql
type Image {
  imageable: Imageable! @morphTo(type: [
    { type: "Post", model: "App\\Models\\BlogPost" },
    { type: "User", model: "App\\Models\\User" }
  ])
}
```

**例3: インターフェースの使用**
```graphql
# GraphQLインターフェースの定義
interface Commentable {
  id: ID!
}

type Post implements Commentable {
  id: ID!
  title: String!
}

type Video implements Commentable {
  id: ID!
  url: String!
}

type Comment {
  commentable: Commentable! @morphTo
}
```

**注意事項**:
- GraphQLのインターフェースを定義して、複数のモデル型を返す必要がある
- データベース内の*_type、*_idフィールドを使用して親モデルを特定する
- @morphMany、@morphOneの「逆側」のリレーション

### @morphToMany

ポリモーフィックな多対多のリレーションシップを解決します。
中間テーブル（ピボット）を介して異なるモデルタイプ間の多対多の関連を定義できます。

**オプション**:
- `relation`: 使用するリレーションシップメソッドの名前（デフォルトはフィールド名と同じ）
- `type`: ページネーションタイプ（`paginator`、`simple`、`connection`）
- `defaultCount`: ページネーション時のデフォルト項目数
- `maxCount`: ページネーション時の最大項目数
- `model`: リレーション先のモデルクラス名（自動推論できない場合に指定）
- `scopes`: 適用するモデルスコープの配列
- `pivot`: 中間テーブルから追加のカラムを取得するかどうか（true/false）

**例1: 基本的な使用法**
```graphql
type Post {
  tags: [Tag!]! @morphToMany
}
```

**例2: ピボットデータの取得**
```graphql
type Post {
  tags: [Tag!]! @morphToMany(pivot: true)
}
```

**例3: ページネーションの使用**
```graphql
type Post {
  tags(first: Int!, page: Int): TagPaginator! @morphToMany(type: "paginator")
}
```

**注意事項**:
- 中間テーブルには`taggable_id`や`taggable_type`などのポリモーフィックフィールドが必要
- 異なるモデルタイプ（Post, Video, など）が同じモデル（Tag）と関連付け可能
- 通常のManyToManyよりも柔軟だが、データベースの設計が複雑になる

## データ操作ディレクティブ

### @create

与えられた引数で新しいEloquentモデルを作成します。ミューテーション用のディレクティブです。

**オプション**:
- `model`: 作成するEloquentモデルのクラス名（デフォルトはフィールドの戻り値型から推論）
- `subscription`: 作成イベントを購読するためのサブスクリプション名

**例1: 基本的な使用法**
```graphql
type Mutation {
  createUser(name: String!, email: String!, password: String!): User @create
}
```

**例2: モデルの明示的な指定**
```graphql
type Mutation {
  addProfile(title: String!, user_id: ID!): Profile @create(model: "App\\Models\\UserProfile")
}
```

**例3: リレーションデータを含む作成**
```graphql
type Mutation {
  createUser(
    name: String!
    email: String!
    posts: CreatePostsRelation @spread
  ): User @create
}

input CreatePostsRelation {
  create: [CreatePostInput!]!
}

input CreatePostInput {
  title: String!
  content: String!
}
```

**注意事項**:
- バリデーションには@rulesディレクティブを組み合わせて使用
- 作成したモデルは自動的に返される
- データベースのデフォルト値や自動的に割り当てられる値（例：id, created_at）は自動的に処理される

### @createMany

複数の新しいEloquentモデルを一度に作成します。配列入力を受け取り、複数のモデルを作成します。

**オプション**:
- `model`: 作成するEloquentモデルのクラス名（デフォルトはフィールドの戻り値型から推論）
- `relation`: リレーション経由で作成する場合のリレーション名

**例1: 基本的な使用法**
```graphql
type Mutation {
  createUsers(input: [UserInput!]!): [User!]! @createMany
}

input UserInput {
  name: String!
  email: String!
}
```

**例2: モデルの明示的な指定**
```graphql
type Mutation {
  importPosts(posts: [PostInput!]!): [Post!]! @createMany(model: "App\\Models\\BlogPost")
}
```

**例3: リレーションを通じての作成**
```graphql
type Mutation {
  createUserPosts(
    user_id: ID!
    posts: [PostInput!]!
  ): [Post!]! @createMany(relation: "posts")
}
```

**注意事項**:
- 入力は配列形式で受け取る必要がある
- 各入力項目に対して個別のモデルが作成される
- バルクインサートではなく、複数のinsertクエリを発行する
- バリデーションエラーがあると全ての作成が失敗する

### @update

既存のEloquentモデルを提供された引数で更新します。

**オプション**:
- `model`: 更新するEloquentモデルのクラス名（デフォルトはフィールドの戻り値型から推論）
- `find`: モデルを検索するための属性（デフォルトはid）

**例1: 基本的な使用法**
```graphql
type Mutation {
  updateUser(id: ID!, name: String, email: String): User @update
}
```

**例2: カスタム検索属性の使用**
```graphql
type Mutation {
  updateUserByEmail(
    email: String! @eq
    name: String
  ): User @update(find: "email")
}
```

**例3: リレーションデータを含む更新**
```graphql
type Mutation {
  updatePost(
    id: ID!
    title: String
    content: String
    categories: UpdateCategoriesRelation @spread
  ): Post @update
}

input UpdateCategoriesRelation {
  sync: [ID!]
  connect: [ID!]
  disconnect: [ID!]
}
```

**注意事項**:
- 更新するモデルが存在しない場合はエラーが発生
- リレーションシップの更新にはconnect、create、update、delete、syncなどの操作が可能
- @findディレクティブと同様にデフォルトでidを使用してモデルを検索

### @updateMany

複数のEloquentモデルを一度に更新します。配列入力を受け取り、各モデルを更新します。

**オプション**:
- `model`: 更新するEloquentモデルのクラス名（デフォルトはフィールドの戻り値型から推論）

**例1: 基本的な使用法**
```graphql
type Mutation {
  updateUsers(input: [UserUpdateInput!]!): [User!]! @updateMany
}

input UserUpdateInput {
  id: ID!
  name: String
  email: String
}
```

**例2: モデルの明示的な指定**
```graphql
type Mutation {
  bulkUpdatePosts(posts: [PostUpdateInput!]!): [Post!]! @updateMany(model: "App\\Models\\BlogPost")
}

input PostUpdateInput {
  id: ID!
  title: String
  published: Boolean
}
```

**注意事項**:
- 各入力項目には更新するモデルのIDが必須
- 存在しないIDを指定した場合はエラーが発生
- 更新はトランザクションで行われ、1つでも失敗すると全て失敗する
- パフォーマンスのため一括操作は避け、必要な場合のみ使用する

### @delete

指定されたモデルをデータベースから削除します。ソフトデリート（論理削除）と完全削除の両方に対応します。

**オプション**:
- `model`: 削除するEloquentモデルのクラス名（デフォルトはフィールドの戻り値型から推論）
- `find`: モデルを検索するための属性（デフォルトはid）
- `softDeletes`: 論理削除を使用するかどうか（trueまたはfalse）、モデルで定義済みの場合は自動検出
- `dirtyDelete`: 関連モデルの削除を無視するかどうか（デフォルトはfalse）

**例1: 基本的な使用法**
```graphql
type Mutation {
  deleteUser(id: ID!): User @delete
}
```

**例2: カスタム検索属性の使用**
```graphql
type Mutation {
  deleteUserByEmail(email: String! @eq): User @delete(find: "email")
}
```

**例3: 完全削除の強制（ソフトデリートを無視）**
```graphql
type Mutation {
  forceDeleteUser(id: ID!): User @delete(softDeletes: false)
}
```

**例4: リレーション制約を無視して削除**
```graphql
type Mutation {
  deleteUserWithAllData(id: ID!): User @delete(dirtyDelete: true)
}
```

**注意事項**:
- モデルがソフトデリートを実装している場合は論理削除が適用される
- 削除前に返す値が保存されるため、削除後もモデルデータを返せる
- リレーションシップの制約によって削除が妨げられる場合がある
- `dirtyDelete: true` は外部キー制約がない場合のみ使用可能

### @upsert

モデルが存在する場合は更新、存在しない場合は新規作成を行います。
「アップサート」操作（update + insert）を一度に処理できます。

**オプション**:
- `model`: 対象のEloquentモデルのクラス名（デフォルトはフィールドの戻り値型から推論）
- `find`: モデルを検索するための属性（デフォルトはid）

**例1: 基本的な使用法**
```graphql
type Mutation {
  upsertUser(id: ID, name: String!, email: String!): User @upsert
}
```

**例2: カスタム検索属性の使用**
```graphql
type Mutation {
  upsertUserByEmail(
    email: String!  # これが一致するレコードを探す
    name: String
    role: String
  ): User @upsert(find: "email")
}
```

**例3: リレーションを含むアップサート**
```graphql
type Mutation {
  upsertPost(
    id: ID
    title: String!
    content: String!
    categories: UpsertCategoriesRelation @spread
  ): Post @upsert
}

input UpsertCategoriesRelation {
  connect: [ID!]
  create: [CategoryInput!]
  update: [CategoryUpdateInput!]
  sync: [ID!]
}
```

**注意事項**:
- 検索属性（デフォルトはid）がnullまたは提供されていない場合は常に新規作成される
- 検索属性に一致するレコードが見つかった場合は更新される
- 1つのミューテーションで作成と更新の両方のロジックをカバーできる

### @upsertMany

複数のモデルに対してアップサート操作（存在すれば更新、存在しなければ作成）を一度に実行します。

**オプション**:
- `model`: 対象のEloquentモデルのクラス名（デフォルトはフィールドの戻り値型から推論）
- `find`: モデルを検索するための属性（デフォルトはid）

**例1: 基本的な使用法**
```graphql
type Mutation {
  upsertUsers(input: [UserUpsertInput!]!): [User!]! @upsertMany
}

input UserUpsertInput {
  id: ID        # IDがあれば更新、なければ作成
  name: String!
  email: String
}
```

**例2: カスタム検索属性の使用**
```graphql
type Mutation {
  upsertProductsByCode(
    products: [ProductUpsertInput!]!
  ): [Product!]! @upsertMany(find: "product_code")
}

input ProductUpsertInput {
  product_code: String!  # これが一致するレコードを探す
  name: String
  price: Float
  description: String
}
```

**注意事項**:
- 各入力項目に検索属性（デフォルトはid）がある場合は更新、ない場合は作成
- 処理は個別に行われるため、一部のレコードは更新、一部は作成となる場合がある
- トランザクションは使用されるが、パフォーマンスのため大量のデータには注意
- @upsertの配列バージョンとして使用できる

## クエリ修飾子ディレクティブ

### @builder

指定されたメソッドでクエリビルダーを操作し、クエリに追加の制約を適用できます。

```graphql
type Query {
  activeUsers: [User!]! @builder(method: "active")
}
```

### @scope

クエリビルダーにスコープを追加し、引数のクライアント提供値をパラメータとして渡します。

```graphql
type Query {
  posts(trending: Boolean @scope(name: "trending")): [Post!]! @paginate
}
```

### @where

入力値をwhereフィルタとして使用し、様々な演算子やLaravelの句を許可します。

```graphql
type Query {
  posts(category_id: ID @where(operator: "=")): [Post!]! @paginate
}
```

### @whereNotNull

値がnullではないことをフィルタリングします。

```graphql
type Query {
  users(email: String @whereNotNull): [User!]! @paginate
}
```

### @whereNull

値がnullであることをフィルタリングします。

```graphql
type Query {
  users(deletedAt: DateTime @whereNull): [User!]! @paginate
}
```

### @whereBetween

カラムの値が2つの値の間にあることを検証します。

```graphql
type Query {
  posts(created_between: [Date!]! @whereBetween(key: "created_at")): [Post!]! @paginate
}
```

### @whereNotBetween

カラムの値が2つの値の範囲外にあることを検証します。

```graphql
type Query {
  posts(excluded_dates: [Date!]! @whereNotBetween(key: "published_at")): [Post!]! @paginate
}
```

## その他の便利なディレクティブ

### @with

EloquentリレーションをEager Loadします。

```graphql
type Query {
  posts: [Post!]! @all @with(relation: "comments")
}
```

### @withCount

フィールドがクエリされた場合に、EloquentリレーションのカウントをEager Loadします。

```graphql
type User {
  posts_count: Int! @withCount(relation: "posts")
}
```

### @paginate

複数のモデルインスタンスをページネーション化された形式で返します。複数のページネーション戦略をサポートしています。

**オプション**:
- `type`: ページネーションタイプ（`paginator`、`simple`、`connection`）
- `model`: 使用するEloquentモデルのクラス名 (例: `"App\\Models\\User"`)
- `defaultCount`: デフォルトの項目数 (デフォルト: 15)
- `maxCount`: 一度に取得できる最大項目数 (デフォルト: なし)
- `scopes`: 適用するモデルスコープの配列
- `builder`: カスタムクエリビルダーの名前空間付きクラス名
- `resolver`: 独自の解決ロジックを実装するリゾルバクラス
- `complexityField`: 複雑度の計算に使用するフィールド名

**例1: 基本的なページネータ**
```graphql
type Query {
  users(first: Int!, page: Int): UserPaginator! @paginate
}
```

**例2: シンプルページネーション**
```graphql
type Query {
  users(first: Int!, page: Int): UserSimplePaginator! @paginate(type: "simple")
}
```

**例3: リレーコネクション（GraphQLの標準的なページネーション）**
```graphql
type Query {
  users(first: Int!, after: String): UserConnection! @paginate(type: "connection")
}
```

**例4: カウント制限の設定**
```graphql
type Query {
  users(first: Int!, page: Int): UserPaginator! @paginate(defaultCount: 10, maxCount: 100)
}
```

**注意事項**:
- ページネーションタイプごとに異なる型が生成されます（`[Type]Paginator`、`[Type]SimplePaginator`、`[Type]Connection`）
- 認証と認可のディレクティブと組み合わせて使用できます

### @softDeletes

フィールドにtrashed引数を追加することで、ゴミ箱に入った要素を取得するかどうかをフィルタリングできるようにします。

```graphql
type Query {
  posts(trashed: Trashed @trashed): [Post!]! @softDeletes
}
```

### @orderBy

1つまたは複数の指定されたカラムで結果リストをソートします。許可されるカラムの制限や、リレーションシップの集計によるソートのオプションがあります。

```graphql
type Query {
  users(orderBy: _ @orderBy(columns: ["name", "email"])): [User!]! @all
}
```

### @in

クライアントから提供されたリスト値を使用して、データベースクエリにIN条件を追加します。

```graphql
type Query {
  users(ids: [ID!]! @in(key: "id")): [User!]! @all
}
```

### @neq

クライアントから提供された値を使用して、データベースクエリにNot-Equal条件を追加します。

```graphql
type Query {
  users(excludeId: ID! @neq(key: "id")): [User!]! @all
}
```

### @notIn

クライアントから提供された値を使用して、データベースクエリにNOT IN条件を追加します。

```graphql
type Query {
  users(excludeIds: [ID!]! @notIn(key: "id")): [User!]! @all
}
```

### @like

データベースクエリにLIKE条件を追加します。ワイルドカード検索に使用できます。

```graphql
type Query {
  users(name: String! @like): [User!]! @all
}
```

### @search

指定された入力値で全文検索を実行します（Laravel Scoutが必要です）。

```graphql
type Query {
  posts(search: String! @search): [Post!]! @all
}
```

### @whereJsonContains

入力値をwhereJsonContainsフィルタとして使用します。JSONカラムに対するクエリに役立ちます。

```graphql
type Query {
  users(preferences: String! @whereJsonContains(key: "preferences")): [User!]! @all
}
```

### @whereKey

Eloquentモデルクエリに主キーに対するwhere句を追加します。

```graphql
type Query {
  users(id: ID! @whereKey): [User!]! @all
}
```

### @whereConditions

複雑な条件を持つWHERE句をクエリに追加します。柔軟な条件式を可能にします。

```graphql
type Query {
  users(
    where: _ @whereConditions(columns: ["name", "email", "created_at"])
  ): [User!]! @all
}
```

### @whereHasConditions

リレーションシップに対して複雑な条件を持つWHERE句を追加します。

```graphql
type Query {
  posts(
    hasComments: _ @whereHasConditions(relation: "comments")
  ): [Post!]! @all
}
```

### @model

モデルクラスをオブジェクトタイプにマッピングします。モデル名がタイプ名と異なる場合に便利です。

```graphql
type AdminUser @model(class: "App\\Models\\Admin") {
  id: ID!
  name: String!
}
```

### @lazyLoad

モデルのリストのリレーションに対して遅延Eager Loadを実行します。

```graphql
type Query {
  posts: [Post!]! @all @lazyLoad(relations: ["comments", "author"])
}
```

### @trashed

ゴミ箱に入った要素を取得するかどうかをフィルタリングできるようにします。

```graphql
type Query {
  users(trashed: Trashed @trashed): [User!]! @all
}

enum Trashed {
  ONLY
  WITH
  WITHOUT
}
```

### @withoutGlobalScopes

クエリビルダーから任意の数のグローバルスコープを省略します。

```graphql
type Query {
  users: [User!]! @all @withoutGlobalScopes
  # または特定のスコープを指定
  posts: [Post!]! @all @withoutGlobalScopes(scopes: ["active", "published"])
}
```

---

LighthouseのEloquent関連ディレクティブの広範な提供は、LighthouseがLaravelアプリケーションの主要なGraphQLレイヤーとなることへの深いコミットメントを示しています。これらのディレクティブは、単純なCRUD操作を超えて、ソフトデリート、グローバルスコープ、Eager Load、複雑なWHERE句といった高度な機能をスキーマ内で直接提供します。

これにより、多くのLaravelプロジェクトにおいて、データフェッチングや操作のためのカスタムリゾルバロジックを大幅に削減、あるいは排除することが可能になります。宣言的なスキーマレベルでのこれらの複雑な操作の制御は、開発サイクルを非常に効率的にし、より宣言的なAPI定義を可能にします。

次の章では、認証と認可に関連するディレクティブについて詳しく見ていきます。
