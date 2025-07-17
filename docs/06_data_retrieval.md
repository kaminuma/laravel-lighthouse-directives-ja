# データ取得とリゾルバ（Data Retrieval & Resolvers）

### @all

**概要:** 指定したモデルの**全レコード**を取得し、そのコレクションを返すクエリ用ディレクティブです。ID指定なしで全件リストを返すシンプルなクエリを簡潔に定義できます。
**使用例:** 全ユーザー一覧を取得するクエリフィールド:

```graphql
type Query {
  users: [User!]! @all
}
```

これだけで、Userモデルの全レコードを取得し返却します。
**注意点:** デフォルトではGraphQLの型名からモデルクラス（App\Userなど）を推測しますが、命名が異なる場合は`model`引数でモデルクラスを明示できます。たとえば型名とモデルクラス名が一致しない場合 `@all(model: "App\\Blog\\BlogEntry")` のように指定します。また、`builder`引数で特定のクエリビルダ生成メソッドを呼び出すこともできます。`@all`は単純な全件取得なので、大量データを返す恐れがある点に注意し、必要に応じて@paginateなどに置き換えてください。

### @paginate

**概要:** 複数エントリを**ページネーション**付きで取得するクエリフィールドを定義するディレクティブです。クライアントがページサイズやページ番号を指定できるよう、フィールドの引数と返却型を自動的に変換します。
**使用例:** 投稿一覧をページネートで取得するフィールド:

```graphql
type Query {
  posts: [Post!]! @paginate
}
```

上記スキーマ定義は実行時に自動で次のように変換されます:

```graphql
type Query {
  posts(first: Int!, page: Int): PostPaginator
}
type PostPaginator {
  data: [Post!]!
  paginatorInfo: PaginatorInfo!
}
```

クライアントは`posts(first: 10, page: 2)`のようにページサイズ・ページ番号を指定できます。`PostPaginator.data`にPostリスト、`paginatorInfo`にページ情報（total, currentPage等）が含まれます。
**注意点:** `type`引数でページネーション方式を切り替えられ、デフォルトは`PAGINATOR`（総件数含むページネーション）です。他に`SIMPLE`（簡易、総件数なし）や`CONNECTION`（Relay風カーソル方式）があります。Relayの`CONNECTION`も内部的にはオフセットに変換される点に留意してください。`@paginate`はルートクエリ専用で、**リレーションのページネーションには**代わりに`@hasMany(type: ...)`等のリレーションディレクティブを使用します。

### @find

**概要:** 条件にマッチする**単一のモデル**を検索して返すフィールドを定義するディレクティブです。主キーやユニークキーで検索する典型的なクエリをシンプルに記述できます。
**使用例:** IDでユーザーを検索するクエリ:

```graphql
type Query {
  userById(id: ID! @whereKey): User @find
}
```

これで`id`に対応するUserモデルを1件取得し返します。`@whereKey`は主キーによる検索条件を付与するための入力ディレクティブです（後述）。
**注意点:** 複数件ヒットした場合（通常主キー検索では起こりえませんが）`@find`はエラーを投げます。一方、該当なしの場合は`null`を返す仕様です。クエリ条件によっては複数マッチの可能性がある場合、代わりに`@first`の使用を検討してください。また、デフォルトのモデル推測が効かない場合は`model`引数で対象モデルを明示できます。

### @first

**概要:** 複数ヒットする可能性のあるクエリから**最初の1件**を返すディレクティブです。`@find`と異なり、複数マッチしてもエラーとせず最初の結果を返し、該当なしならnullとなります。
**使用例:** タイトルに部分一致する投稿のうち先頭の1件を取得するクエリ:

```graphql
type Query {
  firstPost(title: String @where(operator: "like")): Post @first
}
```

クライアントが`title: "%GraphQL%"`のように検索文字列を与えると、その条件にマッチした最初のPostを返します（2件目以降は無視）。
**注意点:** `@first`は内部的にはEloquentの`->first()`を使うイメージで、コレクションの先頭要素だけを取り出します。大量データから無制限に検索するとパフォーマンスに影響する場合があるため、必要に応じて別途`@limit`等で件数を制限したり適切なインデックスを用意してください。

### @count

**概要:** 関連またはモデル自体の**件数を取得**して返すディレクティブです。集計の一種ですが、特にカウント専用として用意されています。
**使用例:** ユーザーが持つ「いいね」の数を取得するフィールド: `likeCount: Int! @count(relation: "likes")` とします。これにより、そのユーザーの関連するLikeモデル数を効率よくカウントして返却します。
**注意点:** ルートクエリで全体の件数を返す用途にも使えます（例：`totalPosts: Int! @count(model: "Post")`）。`relation`引数と`model`引数は排他で、一方のみ指定します。`@count`は裏でSQLのCOUNTを実行するため高速ですが、複雑な条件付きの集計には`@aggregate`や`@whereHasConditions`と組み合わせる必要があります。また、`distinct`オプション（重複排除）をBooleanで指定できます。

### @aggregate

**概要:** モデルまたはリレーションに対して**集計（aggregate）関数**を実行し結果を返すディレクティブです。平均・合計・最小・最大値などをクエリで取得できます。
**使用例:** 楽曲の総ダウンロード数を取得するフィールド:

```graphql
type Query {
  totalDownloads: Int! 
    @aggregate(model: "Song", column: "downloads", function: SUM)
}
```

Songモデルの`downloads`カラムの合計値を返します。またアルバム型で関連楽曲の平均評価を返す例:

```graphql
type Album {
  rating: Float! @aggregate(relation: "songs", column: "rating", function: AVG)
}
```

Albumに紐づくSongの`rating`カラム平均を計算します。
**注意点:** `model`（特定モデル全体に対する集計）・`relation`（特定オブジェクトの関連に対する集計）・`builder`（任意のビルダ取得関数による集計）は**相互に排他**で一度に一つしか指定できません。`function`は`SUM/AVG/MIN/MAX`から選択します。単に件数を取得したい場合は`@count`を使った方が簡潔です。フィルタリングやスコープも`scopes`引数で併用可能なので、必要に応じて条件付き集計も実現できます。

### @orderBy

**概要:** 結果の**並び順**を指定するディレクティブです。通常、GraphQLスキーマ上にソート用の引数（例：`orderBy`）を設け、このディレクティブを付与することでクエリビルダに`ORDER BY`句を適用します。
**使用例:** Postsをタイトルで昇順/降順ソートする場合、まず列挙型`SortOrder`（値: ASC/DESC）を定義し、クエリに`orderBy: SortOrder @orderBy(column: "title")`のような引数を追加します。これにより、`orderBy: ASC`なら`ORDER BY title ASC`がSQLに付与されます。【※実装例】
**注意点:** `@orderBy`は文字列引数に対して直接使うこともできますが、多くの場合**専用の入力型や列挙型**と組み合わせます。Lighthouseは自動的に入力引数名やenumからカラム名を推測しますが、`column`引数でカラムを明示することも可能です。複数列でのソートには`columns`引数や、あるいは`@orderBy`を繰り返し（repeatable）適用する方法も提供されています（詳細はOrderingの章を参照）。ビルダ強化が必要な点は他のWhere系と同様です。

### @limit

**概要:** クエリで返す**最大件数**を制限するディレクティブです。引数として使う場合はクライアントが上限数を指定でき、フィールドに直接付けると固定の最大件数を適用します。
**使用例:** ページネーションを使わない全件取得クエリにおいて、クライアントが`limit`引数で取得件数を制御できるようにする場合:

```graphql
type Query {
  users(limit: Int @limit): [User!]!
}
```

クライアントが`limit: 4`を指定すれば最大4件までが返ります。
**注意点:** 上記のように引数定義に付与した場合、Resolverが取得した結果コレクションの先頭N件のみ返します。一方、`builder: true`を指定すると、SQLレベルで`LIMIT`句を付与します。`builder: true`は主に**ルートクエリ**で用いるべきで、リレーションに対して使うとバッチロードに支障が出る場合があります。大量データの誤取得を防ぐ安全措置として活用できます。

### フィルタディレクティブ (@eq, @neq, @in など)

**概要:** Lighthouseは複数の**フィルタリング用ディレクティブ**を提供し、引数を使った柔軟な検索条件指定を可能にしています。これらはいずれもクエリビルダに`WHERE`句を追加する働きをします。

* **@eq / @neq:** 引数の値と等しい/等しくないレコードに絞り込みます。例えば`name: String @eq`なら`name = ?`の条件が追加され、`@neq`なら`name <> ?`になります。特定の値一致/不一致の簡易指定に使います。※`@eq`は`@where(operator: "=")`に相当します。
* **@in / @notIn:** 引数にリスト（配列）を取り、カラムがそのリスト内に含まれる/含まれないレコードを取得します。例えば`ids: [ID!] @in(key: "id")`ならSQLで`WHERE id IN (...)`になります。
* **@like:** 引数の文字列を使用して部分一致検索を行います（SQLの`LIKE`句）。`template`引数でワイルドカード位置を指定可能で、例えば`template: "%{}%"`なら値を前後%で囲んで検索します。指定しない場合、ユーザー入力に`%`や`_`を含めて直接渡せます。
* **@where:** オペレータやクエリメソッドを細かく指定できる汎用フィルタです。`operator`引数で`=`や`>`等を指定したり、`clause`で`whereYear`等Laravelの追加クエリを使えます。例えば`@where(operator: ">")`ならその引数値より大きいレコードを取得します。フィールド自体に付与する場合は`key`（カラム名）と`value`を指定し、固定条件を与えることも可能です。`ignoreNull: true`を指定すると、明示的に`null`が渡されたときフィルタを無視する挙動となります。
* **@whereBetween / @whereNotBetween:** 2つの値（範囲）を取る入力型に付与し、その範囲内または外にあるレコードを抽出します。例えば日時範囲でフィルタする場合に`DateRange`入力型（`from`, `to`を持つ）を作り、`createdBetween: DateRange @whereBetween(key: "created_at")`のように使用します。引数を与えなければ無条件、与えればBETWEEN句が適用されます。
* **@whereNull / @whereNotNull:** Boolean引数と組み合わせ、値がNULLかどうかでフィルタします。例：`deleted: Boolean @whereNotNull(key: "deleted_at")`とすれば、`deleted: true`で削除日時がセットされているレコードのみ取得します。`@whereNull`はその逆です。これらもenhanceBuilder対応のフィールドで機能します。
* **@whereJsonContains:** JSON型カラムに対し、指定した値が含まれるレコードをフィルタします。例：`tags: [String]! @whereJsonContains(key: "tags")`なら、JSON列`tags`にすべての指定タグが含まれるものを検索します。`key`でJSON内のキー（パス）も指定可能。
* **@whereConditions / @whereHasConditions:** 複雑な条件の組み合わせ（AND/ORグループなど）を指定するための高度なディレクティブです。`WhereConditions`入力型を使ってネストしたwhere句を構築できます。例えば

  ```graphql
  posts(filter: WhereConditions @whereConditions): [Post!]! @all
  ```

  として、`filter`引数に複数条件を含むオブジェクト（AND/ORやカスタム演算子）を受け取ることが可能です。`@whereHasConditions`はリレーションに対する条件をネストして指定する場合に使用します。これらの詳細は公式ドキュメントの「Complex Where Conditions」で解説されています。

**使用例:** 上記各ディレクティブの簡単な例として:

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

クエリ例:

```graphql
{ posts(title: "GraphQL", ids: [1,2,3]) { id title } }
```

この場合SQLでは`WHERE title LIKE "%GraphQL%" AND id IN (1,2,3)`が自動組み立てされます（他の条件は指定なしなので無視）。

**注意点:** これらフィルタ系ディレクティブは**単独では動作せず**、`@all`や`@paginate`、`@hasMany`等のデータ取得系ディレクティブと併用する必要があります。また、無防備に多くのフィルタを開放すると複雑なクエリでパフォーマンスに影響する可能性があります。必要なものだけを提供し、適宜複合インデックスを貼るなどの対策を検討してください。

### @search

**概要:** **全文検索**（フルテキストサーチ）を行うためのディレクティブです。データベースの全文検索インデックスやLaravel Scoutなどを利用して、テキストフィールドに対する検索を簡潔に実装します。
**使用例:** 投稿をタイトル・本文から全文検索するフィールド:

```graphql
type Query {
  searchPosts(query: String @search): [Post!]! @all
}
```

クライアントが`query: "GraphQL Lighthouse"`のように検索語を渡すと、データベースの全文検索機能（例えばMySQLのMATCH AGAINSTやPostgresのTSVなど）を用いて関連するPostレコードを取得します。
**注意点:** このディレクティブを利用するには対応するドライバーやインデックスの準備が必要です。Lighthouse自体はLaravel Scout経由でAlgoliaやMeiliSearch等を扱うこともできますが、`@search`ディレクティブ単体では基本的に**MySQL InnoDBのFULLTEXT**や**PostgreSQLのto_tsvector**に対応した実装となっています。大量のデータに対するLIKE検索を避けるために使う一方、結果のスコアリングや並び替えには別途実装上の工夫が必要です。

### @node

**概要:** Relay規約に基づく**グローバルID**によるノード取得を実装するディレクティブです。`Node`インターフェースを各型に適用し、グローバルに一意なIDでフェッチできるようにします。
**使用例:** `Node`インターフェースを実装する型に`@node`を付与します:

```graphql
interface Node @interface(resolveType: "App\\GraphQL\\Interfaces\\Node@resolveType") {
  id: ID! @globalId
}
type User implements Node @model(class: "App\\Models\\User") @node {
  id: ID! @globalId
  name: String!
}
```

そしてルートに

```graphql
type Query {
  node(id: ID!): Node @field(resolver: "Nuwave\\Lighthouse\\GlobalId\\NodeRegistry@resolve")
}
```

のようなフィールドを定義します。これにより、クライアントは例えば`node(id: "<base64ID>")`クエリで、Userか他の実装型かわからないIDを一律に取得できます。
**注意点:** `@node`ディレクティブ自体は型へのマーカーのような役割で、内部ではLighthouseの`NodeRegistry`が機能します。これと対になるのが次の`@globalId`で、IDのエンコード/デコードを扱います。Relayを使わない場合は必須ではありませんが、GraphQLのインターフェースを横断してノード取得する場面では非常に便利です。

### @globalId

**概要:** GraphQLの`ID`フィールドを**グローバルID形式**（Relay方式）でエンコード/デコードするディレクティブです。具体的には、「Base64エンコードされた 'モデル名:ID'」の形式と通常の数値IDを相互変換します。
**使用例:** Nodeインターフェース実装型の`id`フィールドなどに付与します:

```graphql
type User implements Node {
  id: ID! @globalId
  ...
}
```

上記により、クライアントに返される`id`は例えば`"VXNlcjoz"`のようなBase64文字列になります（`User:3`をエンコード）。クライアントからこのグローバルIDを`node`クエリなどで送ると、自動的にデコードされ適切なモデル（UserのID=3）を取得します。
**注意点:** `@globalId`は**エンコードとデコード両方**を面倒見ます。スキーマ上の型はID!のままで、クライアントにはBase64文字列、内部では通常のID数値として扱われます。既存のスキーマに後から導入するとID形式が変わるため、導入は慎重に。Relay準拠クライアント（Apolloなど）を使う場合に主に必要となる機能です。 