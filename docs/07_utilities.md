# その他ユーティリティ

---

### @async

**概要**  
ミューテーションの実行を**非同期ジョブ**としてキューに委譲するディレクティブです。重い処理を含むミューテーションでもクライアントへのレスポンスを即座に返し、後続処理はバックグラウンドで行えます。

**使用例**
```graphql
type Mutation {
  processBigData(input: BigDataInput!): Boolean @async
}
```
この例を実行すると、Resolver内では結果として即座に`true`などを返し、実際の`processBigData`処理はLaravelのジョブ（`Nuwave\Lighthouse\Async\AsyncMutation`）としてキューに積まれます。クライアントはジョブ投入が成功したことだけ取得し、処理完了は別途通知やポーリングで確認します。

**注意点**
- `@async`は**ルートミューテーションフィールド**でのみ使用可能です。
- Laravelのキューシステムが有効である必要があります。
- ジョブにはデフォルトでミューテーション名に対応するクラスが使われ、ジョブ内で実際のリゾルバ処理が行われます。キューの設定（接続や失敗時挙動）はLaravel標準に従います。

---

### @broadcast

**概要**  
ミューテーションの結果をGraphQLの**サブスクリプション**経由でブロードキャスト（配信）するディレクティブです。

**主な用途**
- ミューテーション完了時にクライアントへリアルタイム通知

**使用例**
```graphql
type Mutation {
    createPost(input: CreatePostInput!): Post 
        @broadcast(subscription: "postCreated")
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
    createPost(input: CreatePostInput!): Post 
        @event(class: "App\\Events\\PostCreated")
        @create
}
```

**注意点**
- `class`引数でイベントクラス名を指定します
- イベントにはリゾルバから得られた値や引数を渡すことができます

---

### @subscription

**概要**  
GraphQLサブスクリプションのブロードキャスト処理方法を定義するディレクティブです。

**主な用途**
- サブスクリプション型の定義
- クライアントへのリアルタイム通知

**使用例**
```graphql
type Subscription {
    postUpdated(author: ID!): Post 
        @subscription(class: "App\\GraphQL\\Blog\\PostUpdatedSubscription")
}
```

**注意点**
- サブスクリプションクラスはNuwave\Lighthouse\Schema\Types\GraphQLSubscriptionのサブクラスである必要があります
- 認証・認可の実装に注意

---

### Caching関連ディレクティブ (@cache, @cacheControl, @cacheKey, @clearCache)

**概要**  
フィールドの解決結果を**サーバーサイドキャッシュ**するためのディレクティブ群です。パフォーマンス向上のため、一定時間結果を保存したり、HTTPクライアント側でのキャッシュ制御ヒントを与えたりできます。

* **@cache:** フィールド結果をキャッシュストアに保存します。`ttl`（秒）引数で有効期間を指定可能、`cacheKey`やタグでコントロールもできます。
* **@cacheControl:** フィールドに対するHTTPキャッシュヘッダのヒントを設定します。例：`@cacheControl(maxAge: 60, scope: PUBLIC)`のように使い、ブラウザやゲートウェイキャッシュに利用させます（Apollo Cache Control仕様）。
* **@cacheKey:** フィールド結果のキャッシュキーをカスタマイズします。デフォルトでは型名や引数からキー生成しますが、特定のキーに紐付けたい場合に利用します。
* **@clearCache:** キャッシュを**無効化（クリア）**するためのディレクティブです。特定の型・フィールドに関連付けたキャッシュタグを消去できます。例：ミューテーションでデータ更新後に該当一覧のキャッシュを削除する用途など。

**使用例**
```graphql
type Query {
  posts: [Post!]! 
    @cache(ttl: 300)                   # 5分間結果をキャッシュ
    @cacheKey(key: "posts_list")       # 固定キーを使用
}
type Mutation {
  createPost(...): Post! 
    @clearCache(type: "Query", field: "posts")  # 投稿一覧キャッシュをクリア
    @create
}
```
この例では、`posts`クエリ結果が5分間キャッシュされます。新規投稿作成時には、その直後に`Query.posts`のキャッシュを無効化し、次回クエリで最新一覧が取得されます。

**注意点**
- キャッシュディレクティブを適用すると、GraphQLサーバー内部で結果の保存・取得処理が追加されます。環境によってはキャッシュクリアが難しいシナリオもあるため、特に`@cache`の利用は静的データや更新頻度の低いデータに限定するのが安全です。
- `@clearCache`の`idSource`引数では、クリアすべき対象のIDを引数や結果から動的に指定できます（例：「更新したIDのキャッシュのみ削除」等）。
- これら機能はアプリのキャッシュ戦略に応じて組み合わせ、パフォーマンスチューニングに役立ててください。

---

### @complexity

**概要**  
フィールドの**クエリ複雑度スコア**計算をカスタマイズするディレクティブです。デフォルトでは子要素数に固定コストを掛けた計算ですが、特定フィールドでより精密な計算をしたい場合に導入します。

**使用例**
```graphql
type Query {
  posts(includeFullText: Boolean): [Post!]!
    @complexity(resolver: "App\\GraphQL\\Security\\ComplexityAnalyzer@userPosts")
    @all
}
```
`ComplexityAnalyzer@userPosts`メソッド内で、例えば`includeFullText`がtrueならコストを子要素数×3＋定数、falseなら×2＋定数…といった独自ロジックで整数スコアを算出します。このスコアはクエリ全体の複雑度合計に組み込まれ、閾値超過クエリを抑制するのに役立ちます。

**注意点**
- クラスとメソッドは `ComplexityFunction($childrenComplexity, array $args): int` のシグネチャで実装します。`childrenComplexity`はネストしたフィールドの合計コスト、`$args`はクライアント指定の引数です。
- @complexityは**セキュリティ/パフォーマンス**に関わる機能なので、特に公開APIでは適切に設定しましょう（デフォルトでは複雑度上限は無制限、必要に応じ`lighthouse.php`で最大複雑度を設定してください）。

---

### @throttle

**概要**  
フィールドへのアクセス回数を**レート制限**するディレクティブです。Laravelの`ThrottleRequests`ミドルウェア相当の機能をGraphQLフィールド単位で適用します。

**使用例**
```graphql
type Mutation {
  login(email: String!, password: String!): AuthPayload 
    @throttle(maxAttempts: 5, decayMinutes: 15)
}
```
この例では、同一クライアント（IPまたは定義したキー）から`login`ミューテーションが15分間に5回以上呼ばれた場合、残り時間エラーを返すようになります。`name`引数でLaravel定義のレートリミッタ名を指定すれば、それを利用することも可能です。

**注意点**
- GraphQLフィールドの組み合わせごとに細かい制御が必要な場合、`prefix`引数でグループ化もできます。
- Laravelで`Response`を返すようなレートリミッタはLighthouseではサポートされない点に注意してください。
- 制限カウントの単位はデフォルトではIPアドレスですが、Passport等と組み合わせユーザーIDベースにするなど調整可能です。

--- 