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
ミューテーションの結果をGraphQLの**サブスクリプション**経由でブロードキャスト（配信）するディレクティブです。特定のサブスクリプションイベント名を指定し、そのミューテーションが成功した際に購読中のクライアントへ通知を送ります。

**使用例**
```graphql
type Mutation {
  createPost(input: CreatePostInput!): Post 
    @broadcast(subscription: "postCreated")
}
```
対応するサブスクリプションフィールド（例：`postCreated: Post`）をクライアントが購読していれば、このミューテーション完了時にリアルタイム通知を受け取れます。`shouldQueue`引数をtrueにするとブロードキャスト自体をキューに乗せ非同期送信できます。

**注意点**
- `subscription`引数にはサブスクリプションフィールド名を文字列で指定します。これはミューテーションとサブスクリプションの対応を取るために必須です。
- 不特定多数へのリアルタイム通知となるため、ブロードキャストドライバ（Pusher等）の設定と、Lighthouseのサブスクリプション設定が正しく行われている必要があります。

---

### @subscription

**概要**  
GraphQLサブスクリプションフィールドに対して、ブロードキャストの方法を指定するディレクティブです。通常は規約に従えば自動処理されますが、カスタム実装が必要な場合にこのディレクティブでハンドラクラスを登録します。

**使用例**
```graphql
type Subscription {
  postUpdated(author: ID!): Post 
    @subscription(class: "App\\GraphQL\\Blog\\PostUpdatedSubscription")
}
```
`class`引数には`Nuwave\Lighthouse\Schema\Types\GraphQLSubscription`を継承したクラスを指定します。これによりサブスクリプション発行時やフィルタリングのロジックを自前で書くことができます。

**注意点**
- 規約上、サブスクリプションフィールド名に対応するクラス（例: `PostUpdatedSubscription`）を`app/GraphQL/Subscriptions`以下に用意すれば`@subscription`なしでも検出されます。
- ディレクティブはあくまで**命名規約から外れる場合**や**複数のサブスクリプションを1クラスで扱う特殊ケース**等で使います。
- サブスクリプションはサーバー設定（WebSocket等）も必要なため、詳しくはLighthouseのSubscriptionsガイドを参照してください。

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

### @event

**概要**  
フィールドの解決完了後にLaravelの**イベント**をディスパッチするディレクティブです。ミューテーション結果を他システムに伝えたり、非同期処理のトリガーとしたい場合に利用できます。

**使用例**
```graphql
type Mutation {
  createPost(input: CreatePostInput!): Post 
    @event(class: "App\\Events\\PostCreated")
    @create
}
```
または`@event(name: "PostCreated")`のようにイベント名だけ指定し、Lighthouse側でイベントクラスのインスタンス生成方法を知っている場合も利用可能です（正確なシグネチャはLighthouseドキュメント参照）。

**注意点**
- `@event`ディレクティブを使うと、指定イベントが`Event::dispatch()`されます。`class`でクラス名、`name`で文字列名の指定ができますが、`name`の場合はイベントクラスのインスタンス生成方法をLighthouseが知っている必要があります。
- イベントにはリゾルバから得られた結果や引数を渡すこともできます。GraphQLの応答には影響しませんが、イベントリスナー側で例外が出るとミューテーション自体が失敗する点に注意してください。

--- 