# データ取得とリゾルバ（Data Retrieval & Resolvers）

---

### @all

**概要**  
指定したモデルの**全レコード**を取得し、そのコレクションを返すクエリ用ディレクティブです。ID指定なしで全件リストを返すシンプルなクエリを簡潔に定義できます。

**使用例**
```graphql
type Query {
  users: [User!]! @all
}
```
この例だけで、Userモデルの全レコードを取得し返却します。

**注意点**
- デフォルトではGraphQLの型名からモデルクラス（App\Userなど）を推測しますが、命名が異なる場合は`model`引数でモデルクラスを明示できます。
- 例えば型名とモデルクラス名が一致しない場合 `@all(model: "App\\Blog\\BlogEntry")` のように指定します。
- `builder`引数で特定のクエリビルダ生成メソッドを呼び出すことも可能です。
- @allは単純な全件取得なので、大量データを返す恐れがある点に注意し、必要に応じて@paginate等に置き換えてください。

---

### @paginate

**概要**  
複数のエントリをページネーションしてクエリするディレクティブです。

**使用例**
```graphql
type Query {
  users: [User!]! @paginate(type: "paginator", defaultCount: 10)
}
```

**注意点**
- type（paginator, connection, simple）でページネーション方式を選択可能。
- defaultCountやmaxCountで件数制御。

---

### @find

**概要**  
主キーで単一モデルを検索するディレクティブです。該当IDがなければnullを返します。

**使用例**
```graphql
type Query {
  user(id: ID!): User @find
}
```

**注意点**
- 複数ID指定時はエラー。

---

### @first

**概要**  
Eloquentモデルコレクションから最初の結果を取得するディレクティブです。

**使用例**
```graphql
type Query {
  firstUser: User @first
}
```

**注意点**
- 複数結果でもエラーにならず、最初の1件のみ返す。

---

### @count

**概要**  
関連またはモデル自体の**件数を取得**して返すディレクティブです。集計の一種ですが、特にカウント専用として用意されています。

**使用例**
```graphql
likeCount: Int! @count(relation: "likes")
```
この例では、そのユーザーの関連するLikeモデル数を効率よくカウントして返却します。

**注意点**
- ルートクエリで全体の件数を返す用途にも使えます（例：`totalPosts: Int! @count(model: "Post")`）。
- `relation`引数と`model`引数は排他で、一方のみ指定します。
- @countは裏でSQLのCOUNTを実行するため高速ですが、複雑な条件付き集計には`@aggregate`や`@whereHasConditions`と組み合わせる必要があります。
- また、`distinct`オプション（重複排除）をBooleanで指定できます。

---

### @aggregate

**概要**  
モデルまたはリレーションに対して**集計（aggregate）関数**を実行し結果を返すディレクティブです。平均・合計・最小・最大値などをクエリで取得できます。

**使用例**
```graphql
type Query {
  totalDownloads: Int! 
    @aggregate(model: "Song", column: "downloads", function: SUM)
}
```
Songモデルの`downloads`カラムの合計値を返します。アルバム型で関連楽曲の平均評価を返す例：
```graphql
type Album {
  rating: Float! @aggregate(relation: "songs", column: "rating", function: AVG)
}
```
Albumに紐づくSongの`rating`カラム平均を計算します。

**注意点**
- `model`（特定モデル全体への集計）・`relation`（特定オブジェクトの関連への集計）・`builder`（任意のビルダ取得関数による集計）は**相互に排他**で一度に一つしか指定できません。
- `function`は`SUM/AVG/MIN/MAX`から選択します。
- 単に件数を取得したい場合は`@count`の方が簡潔です。
- フィルタリングやスコープも`scopes`引数で併用可能なので、必要に応じて条件付き集計も実現できます。

---

### @orderBy

**概要**  
クエリ結果の並び順を指定するディレクティブです。

**使用例**
```graphql
type Query {
  users: [User!]! @all @orderBy(column: "created_at", direction: DESC)
}
```

**注意点**
- column, direction引数で制御。

---

### @limit

**概要**  
取得件数の上限を指定するディレクティブです。

**使用例**
```graphql
type Query {
  users: [User!]! @all @limit(max: 100)
}
```

**注意点**
- max引数で上限指定。

---

### フィルタディレクティブ (@eq, @neq, @in など)

**概要**  
Lighthouseは複数の**フィルタリング用ディレクティブ**を提供し、引数を使った柔軟な検索条件指定を可能にしています。これらはいずれもクエリビルダに`WHERE`句を追加する働きをします。

* **@eq / @neq:** 引数の値と等しい/等しくないレコードに絞り込みます。例：`name: String @eq`なら`name = ?`条件、`@neq`なら`name <> ?`条件になります。特定値一致/不一致の簡易指定に使います。※`@eq`は`@where(operator: "=")`に相当。
* **@in / @notIn:** 引数にリスト（配列）を取り、カラムがそのリスト内に含まれる/含まれないレコードを取得します。例：`ids: [ID!] @in(key: "id")`ならSQLで`WHERE id IN (...)`になります。
* **@like:** 引数の文字列を使って部分一致検索を行います（SQLの`LIKE`句）。`template`引数でワイルドカード位置を指定可能、例：`template: "%{}%"`なら値を前後%で囲んで検索します。指定しない場合、ユーザー入力に`%`や`_`を含めて直接渡せます。
* **@where:** オペレータやクエリメソッドを細かく指定できる汎用フィルタです。`operator`引数で`=`や`>`等を指定、`clause`で`whereYear`等Laravelの追加クエリを使えます。例：`@where(operator: ">")`ならその引数値より大きいレコードを取得。フィールド自体に付与する場合は`key`（カラム名）と`value`を指定し、固定条件を与えることも可能。`ignoreNull: true`で明示的に`null`が渡されたときフィルタを無視する挙動も。
* **@whereBetween / @whereNotBetween:** 2つの値（範囲）を取る入力型に付与し、その範囲内/外のレコードを抽出します。例：日付範囲でフィルタする場合に`DateRange`型（`from`, `to`を持つ）を作り、`createdBetween: DateRange @whereBetween(key: "created_at")`のように使います。
* **@whereNull / @whereNotNull:** Boolean引数と組み合わせ、値がNULLかどうかでフィルタします。例：`deleted: Boolean @whereNotNull(key: "deleted_at")`なら、`deleted: true`で削除日時がセットされているレコードのみ取得。`@whereNull`は逆です。これらもenhanceBuilder対応フィールドでのみ機能します。
* **@whereJsonContains:** JSON型カラムに対し、指定値が含まれるレコードをフィルタします。例：`tags: [String]! @whereJsonContains(key: "tags")`なら、JSON列`tags`に全指定タグが含まれるものを検索。`key`でJSON内のキー（パス）も指定可能。
* **@whereConditions / @whereHasConditions:** 複雑な条件の組み合わせ（AND/ORグループなど）を指定するための高度なディレクティブです。`WhereConditions`入力型を使い、ネストしたwhere句を構築できます。例：
```graphql
type Query {
  posts(
    title: String @like(template: "%{}%"),        # タイトル部分一致
    ids: [Int!] @in(key: "id"),                   # 複数ID指定
    authorId: ID @eq(key: "author_id"),           # 著者ID一致
    excludeTag: String @whereNotNull(key: "tags") # tagsがNULLでない（例として）
  ): [Post!]! @all
}
```

**注意点**
- これらフィルタ系ディレクティブは**単独では動作せず**、`@all`や`@paginate`、`@hasMany`等のデータ取得系ディレクティブと併用する必要があります。
- また、無防備に多くのフィルタを開放すると複雑なクエリでパフォーマンスに影響する可能性があるため、必要なものだけを提供し、適宜複合インデックスを貼るなどの対策を検討してください。

---

### @search

**概要**  
全文検索やLIKE検索を簡単に実装できるディレクティブです。

**使用例**
```graphql
type Query {
  searchUsers(query: String!): [User!]! @search(fields: ["name", "email"])
}
```

**注意点**
- fields引数で検索対象カラムを指定。

---

### @node

**概要**  
Relay仕様のNode型を定義するディレクティブです。

**使用例**
```graphql
interface Node @interface(resolveType: "App\\GraphQL\\Interfaces\\Node@resolveType") {
  id: ID! @globalId
}
type User implements Node @model(class: "App\\Models\\User") @node {
  id: ID! @globalId
  name: String!
}
```
ルートクエリに
```graphql
type Query {
  node(id: ID!): Node @field(resolver: "Nuwave\\Lighthouse\\GlobalId\\NodeRegistry@resolve")
}
```
のようなフィールドを定義します。これにより、クライアントは例えば`node(id: "<base64ID>")`クエリで、Userか他の実装型かわからないIDを一意に取得できます。

**注意点**
- `@node`ディレクティブ自体は型へのマーカーのような役割で、内部ではLighthouseの`NodeRegistry`が機能します。これと対になるのが次の`@globalId`で、IDのエンコード/デコードを扱います。
- Relayを使わない場合は必須ではありませんが、GraphQLのインターフェースを横断してノード取得する場面では非常に便利です。

---

### @globalId

**概要**  
Relay仕様のグローバルIDを生成・解決するディレクティブです。

**使用例**
```graphql
type User {
  id: ID! @globalId
}
```

**注意点**
- Relay互換API構築時に利用。

---

### @hash

**概要**  
値をハッシュ化して保存・返却するディレクティブです。

**使用例**
```graphql
type Mutation {
  setPassword(password: String! @hash): User
}
```

**注意点**
- パスワード等のセキュアな値に利用。

---

### @inject

**概要**  
サービスコンテナから依存オブジェクトを注入するディレクティブです。

**使用例**
```graphql
type Query {
  currentUser: User @inject(service: "App\\Services\\UserService")
}
```

**注意点**
- service引数でクラス名指定。

---

### @like

**概要**  
LIKE検索を行うディレクティブです。

**使用例**
```graphql
type Query {
  usersByName(name: String!): [User!]! @like(column: "name")
}
```

**注意点**
- column引数で対象カラム指定。

---

### @throttle

**概要**  
リクエスト回数制限（レートリミット）を適用するディレクティブです。

**使用例**
```graphql
type Mutation {
  updateUser(input: UpdateUserInput!): User @throttle(maxAttempts: 5, decayMinutes: 1)
}
```

**注意点**
- maxAttempts, decayMinutes等で制御。

---

### @union

**概要**  
GraphQLのUnion型を定義するディレクティブです。

**使用例**
```graphql
union SearchResult @union {
  User
  Post
}
```

**注意点**
- 型の組み合わせに注意。

---

### @withoutGlobalScopes

**概要**  
クエリビルダーからグローバルスコープを除外するディレクティブです。

**使用例**
```graphql
type Query {
  users: [User!]! @all @withoutGlobalScopes(scopes: ["active"])
}
```

**注意点**
- scopes引数で除外するスコープ名を指定。

---

### @rename

**概要**  
GraphQLスキーマ上のフィールド名と、実際のEloquentモデルやDBカラム名をマッピングするディレクティブです。

**使用例**
```graphql
type User {
  fullName: String! @rename(attribute: "full_name")
}
```

**注意点**
- attribute引数でDBカラム名を指定。

---

### @spread

**概要**  
入力型のフィールドを親の引数として展開するディレクティブです。

**使用例**
```graphql
type Mutation {
  updateUser(input: UpdateUserInput! @spread): User
}
```

**注意点**
- 入力型のネストをフラットに扱いたい場合に便利。

---

### @trashed

**概要**  
ソフトデリートされたレコードも含めて取得するディレクティブです。

**使用例**
```graphql
type Query {
  users: [User!]! @all @trashed
}
```

**注意点**
- only, with, without引数で取得範囲を制御。

---

### @softDeletes

**概要**  
ソフトデリート機能を有効にするディレクティブです。

**使用例**
```graphql
type User @softDeletes {
  ...
}
```

**注意点**
- EloquentモデルでSoftDeletesトレイトが必要。

---

### @scope

**概要**  
EloquentスコープをGraphQLクエリで適用できるディレクティブです。

**使用例**
```graphql
type Query {
  users: [User!]! @all @scope(name: "active")
}
```

**注意点**
- name引数でスコープ名を指定。

---

### @convertEmptyStringsToNull

**概要**  
空文字列をnullに変換するディレクティブです。

**使用例**
```graphql
type Mutation {
  updateUser(input: UpdateUserInput! @convertEmptyStringsToNull): User
}
```

**注意点**
- 入力値の前処理に便利。

---

### @trim

**概要**  
入力値の前後の空白を自動でトリムするディレクティブです。

**使用例**
```graphql
type Mutation {
  updateUser(input: UpdateUserInput! @trim): User
}
```

**注意点**
- 入力値の前処理に便利。

---

### @drop

**概要**  
フィールドや型をスキーマから除外するディレクティブです。

**使用例**
```graphql
type User {
  password: String @drop
}
```

**注意点**
- セキュリティや不要フィールドの除外に活用。 

---

### @whereConditions

**概要**  
複雑なAND/OR条件やネストしたwhere句をGraphQL入力型で柔軟に指定できるディレクティブです。

**使用例**
```graphql
type Query {
  users(where: WhereConditions @whereConditions): [User!]! @all
}

input WhereConditions {
  AND: [WhereConditions!]
  OR: [WhereConditions!]
  column: String
  operator: String
  value: String
}
```

**注意点**
- @allや@hasMany等のデータ取得系ディレクティブと併用します。
- 入力型の設計に注意。

---

### @whereHasConditions

**概要**  
リレーション先に対する複雑なwhere条件を指定できるディレクティブです。

**使用例**
```graphql
type Query {
  users(whereHas: WhereHasConditions @whereHasConditions): [User!]! @all
}

input WhereHasConditions {
  relation: String!
  condition: WhereConditions
}
```

**注意点**
- @allや@hasMany等のデータ取得系ディレクティブと併用します。
- relation引数でリレーション名を指定。 