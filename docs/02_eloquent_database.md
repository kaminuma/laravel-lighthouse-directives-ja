# Eloquentとデータベース操作ディレクティブ

---

このカテゴリには、LaravelのEloquent ORMと基盤となるデータベースと直接対話し、一般的なデータフェッチ、作成、更新、削除操作を容易にするディレクティブが含まれます。

## 検索と取得ディレクティブ

### @all

全てのEloquentモデルを取得し、コレクションを返します。モデルクラス名またはビルダー関数を指定できます。

```graphql
type Query {
  users: [User!]! @all
}
```

### @find

提供された引数（通常は主キー）に基づいてモデルを検索します。

```graphql
type Query {
  user(id: ID! @eq): User @find
}
```

### @first

Eloquentモデルのコレクションから最初のクエリ結果を取得します。

```graphql
type Query {
  userByEmail(email: String! @eq): User @first
}
```

## 集計ディレクティブ

### @aggregate

指定されたリレーションシップまたはモデル内のカラムの集計値を返します。カラム、集計関数（AVG、SUM、MIN、MAX）、リレーション、モデル、またはビルダー関数を指定できます。

```graphql
type Query {
  averageUserAge: Float! @aggregate(model: "App\\Models\\User", field: "age", function: AVG)
}
```

### @count

指定されたリレーションシップまたはモデルのカウントを返します。カラムやDISTINCT値のオプションがあります。

```graphql
type Query {
  userCount: Int! @count(model: "App\\Models\\User")
}
```

## リレーションシップディレクティブ

### @belongsTo

EloquentのBelongsToリレーションシップを通じてフィールドを解決します。フィールド名とリレーションシップメソッド名が同じであると仮定しますが、異なるリレーション名を指定することも可能です。

```graphql
type Post {
  author: User! @belongsTo
}
```

### @belongsToMany

EloquentのBelongsToManyリレーションシップを通じてフィールドを解決します。ページネーションタイプ（PAGINATOR、SIMPLE、CONNECTION）をサポートし、中間テーブルのカラムを取得することも可能です。

```graphql
type User {
  roles: [Role!]! @belongsToMany
}
```

### @hasMany

EloquentのHasManyリレーションシップに対応し、ページネーションをサポートします。

```graphql
type User {
  posts: [Post!]! @hasMany
}
```

### @hasOne

EloquentのHasOneリレーションシップに対応します。

```graphql
type User {
  profile: Profile! @hasOne
}
```

### @hasManyThrough

EloquentのHasManyThroughリレーションシップに対応し、@hasManyと同様の使い方が可能です。

```graphql
type Country {
  posts: [Post!]! @hasManyThrough
}
```

### @hasOneThrough

EloquentのHasOneThroughリレーションシップに対応します。

```graphql
type User {
  latestPost: Post @hasOneThrough
}
```

### @morphMany

EloquentのMorphManyリレーションシップに対応し、ページネーションをサポートします。

```graphql
type Post {
  comments: [Comment!]! @morphMany
}
```

### @morphOne

EloquentのMorphOneリレーションシップに対応します。

```graphql
type Post {
  image: Image @morphOne
}
```

### @morphTo

EloquentのMorphToリレーションシップに対応します。

```graphql
type Comment {
  commentable: Commentable! @morphTo
}
```

### @morphToMany

EloquentのManyToMany-Polymorphicリレーションシップに対応し、ページネーションをサポートします。

```graphql
type Post {
  tags: [Tag!]! @morphToMany
}
```

## データ操作ディレクティブ

### @create

与えられた引数で新しいEloquentモデルを作成します。

```graphql
type Mutation {
  createUser(name: String!, email: String!, password: String!): User @create
}
```

### @createMany

複数の新しいEloquentモデルを作成します。

```graphql
type Mutation {
  createUsers(input: [UserInput!]!): [User!]! @createMany
}
```

### @update

与えられた引数でEloquentモデルを更新します。

```graphql
type Mutation {
  updateUser(id: ID!, name: String, email: String): User @update
}
```

### @updateMany

複数のEloquentモデルを更新します。

```graphql
type Mutation {
  updateUsers(input: [UserUpdateInput!]!): [User!]! @updateMany
}
```

### @delete

1つまたは複数のモデルを削除します。

```graphql
type Mutation {
  deleteUser(id: ID!): User @delete
}
```

### @upsert

与えられた引数でEloquentモデルを作成または更新します。

```graphql
type Mutation {
  upsertUser(id: ID, name: String!, email: String!): User @upsert
}
```

### @upsertMany

複数のEloquentモデルを作成または更新します。

```graphql
type Mutation {
  upsertUsers(input: [UserUpsertInput!]!): [User!]! @upsertMany
}
```

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

複数のエントリをページネーションリストとしてクエリします。PAGINATOR、SIMPLE、CONNECTIONタイプをサポートし、モデル、ビルダー、リゾルバ、スコープ、デフォルトカウント、最大カウント、複雑度リゾルバのオプションがあります。

```graphql
type Query {
  users(first: Int!, page: Int): UserPaginator! @paginate
}
```

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
