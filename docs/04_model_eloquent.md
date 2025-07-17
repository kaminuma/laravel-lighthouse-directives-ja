# モデルバインディング・Eloquent

---

### @bind

**概要**  
引数で渡されたID等から対応するモデルインスタンスを自動的に取得・バインドするディレクティブです。  
Laravelのルートモデルバインディングに似ており、リゾルバ内でIDを手動で検索する手間を省けます。

**使用例**
```graphql
type Mutation {
  addUserToCompany(
    user: ID! @bind(class: "App\\Models\\User")
    company: ID! @bind(class: "App\\Models\\Company")
  ): Boolean!
}
```
この例では`user`引数としてIDを受け取りますが、`@bind`によりそのIDに該当する`App\Models\User`モデルが自動ロードされ、Resolverには`$args['user']`としてUserモデルオブジェクトが渡ります。同様に`company`もCompanyモデルになります。

**注意点**
- `class`引数でバインド対象のクラス（Eloquentモデルまたは`__invoke`メソッドを持つカスタムバインダ）を指定します。
- `required`（デフォルトtrue）をfalseにすると、該当IDが無かった場合にその引数は`null`として渡され、Resolverは実行されます（trueの場合は見つからなければバリデーションエラー）。
- 引数がリスト（配列）の場合、バインド結果はコレクションになります。
- Eloquentモデル以外のバインドも、invoke可能クラスを用意すれば可能です。

---

### @model

**概要**  
GraphQLのオブジェクト型と対応するEloquentモデルクラスを明示的にマッピングするディレクティブです。  
通常、Lighthouseは型名からモデル名を推測しますが、それが一致しない場合や別名前空間にある場合に使用します。

**使用例**
```graphql
type Post @model(class: "\\App\\BlogPost") {
  title: String!
  # ... 他フィールド
}
```
これでLighthouseはPost型を解決する際に`App\BlogPost`モデルを使います。

**注意点**
- `@model`を指定すると、他のディレクティブ（@findや@updateなど）もこのクラス名に基づいてモデルを操作します。
- `class`引数は完全修飾クラス名を文字列で記述してください。
- Laravel標準のApp\Models名前空間内で型名と同名のモデルがある場合は`@model`なしでも自動検出されます（デフォルト挙動）。

---

### @method

**概要**  
親オブジェクト（リゾルバの`$root`、通常はモデル）に定義されたメソッドを呼び出してフィールド値を解決するディレクティブです。  
モデルのアクセサでは対応できない引数付きの計算処理などに有用です。

**使用例**
```graphql
type User {
  purchasedItemsCount(year: Int!, includeReturns: Boolean): Int @method
}
```
このフィールドをクエリすると、自動的にUserモデルの`purchasedItemsCount($year, $includeReturns)`メソッドが呼ばれ、その戻り値がフィールドの結果となります。メソッド名をフィールド名と変えたい場合は`@method(name: "getItemsCount")`のように`name`引数で指定できます。

**注意点**
- フィールドの引数定義の順序・型は、対応するメソッドの引数シグネチャと一致させる必要があります。
- クライアントが引数を渡さなかった場合、その位置の引数には`null`が渡されます。
- このディレクティブは親オブジェクト（通常Eloquentモデル）のメソッドを呼び出すため、リゾルバクラスを別途用意せずにモデルのメソッドでビジネスロジックをカプセル化できます。

---

### @scope

**概要**  
指定したスコープ (scope) メソッドをクエリビルダに適用するディレクティブです。  
Laravel Eloquentモデルのローカルスコープや、カスタムビルダーメソッドをGraphQLの引数経由で利用できます。

**使用例**
```graphql
type Query {
  posts(trending: Boolean @scope): [Post!]! @all
}
```
この例では、`trending`引数が指定されるとPostモデルの`scopeTrending(Builder $q, bool $trending)`が呼ばれます。例えば`trending: true`ならスコープ内で人気順の絞り込みをした上で結果を返します。

**注意点**
- `name`引数でスコープ名を明示的に指定できます（デフォルトは引数名と同じ）。
- `@scope`は引数定義か入力フィールド定義で利用し、実際には `$model->scopeName($value)` がクエリビルダにチェーンされます。
- このディレクティブも他のWhere系と同様、内部で`$resolveInfo->enhanceBuilder()`を使うリゾルバ（@allや@hasManyなど）でのみ機能します。

---

### @with

**概要**  
リゾルバ実行時に関連リレーションをEagerロード（高速化のため事前読み込み）するディレクティブです。  
N+1問題を防ぐため、あるフィールドの解決に必要な関連データをあらかじめ取得する用途で使われます。

**使用例**
```graphql
type User {
  taskSummary: String! @with(relation: "tasks") @method(name: "getTaskSummary")
}
```
まず`@with(relation: "tasks")`によりUserモデルの`tasks`リレーションがEagerロードされ、続いて`getTaskSummary`メソッド（@method）が実行されます。これにより、各Userごとにタスクを個別クエリで取りに行くことなく効率的にフィールドを解決できます。

**注意点**
- `relation`引数でロードするリレーション名を指定します（省略時はフィールド名と同じと見なします）。
- `scopes`引数で関連クエリにスコープを適用することも可能です。
- `@with`はフィールド自体の値を直接返すものではなく、あくまで最適化のために関連データを取得するだけです。単に関連コレクションを返したいだけなら、`@hasMany`など該当リレーション用ディレクティブを使う方が適切です。

---

### @withCount

**概要**  
フィールドがクエリされた際に、指定したリレーションの件数のみをEagerロード（事前取得）するディレクティブです。  
フィールド自体の結果として値を返すわけではなく、主に内部計算用に関連数をあらかじめ取得しておくために使います。

**使用例**
```graphql
type User {
  activityStatistics: ActivityStatistics! @withCount(relation: "posts")
}
```
この例では`@withCount(relation: "posts")`により各User毎の`posts_count`（投稿件数）がクエリの段階で取得されます。Resolverではこの件数を使って`activityStatistics`を組み立てることができます。

**注意点**
- このディレクティブ自体はフィールドに直接値を提供しません。単に`posts_count`のような集計値をEagerロードするだけで、GraphQLスキーマ上そのフィールドからcountが返ることはありません（countを返したいだけなら`@count`ディレクティブがあります）。
- 内部で`withCount('relation')`が実行されるため、利用時は対応するリレーションがモデルに存在する必要があります。

---

### @withoutGlobalScopes

**概要**  
Eloquentモデルに設定されたグローバルスコープをクエリから除外するディレクティブです。  
例えばソフトデリートや公開フラグなどの自動フィルタを一時的に無効化してデータ取得したい場合に使います。

**使用例**
```graphql
type Query {
  posts(includeUnscheduled: Boolean @withoutGlobalScopes(names: ["scheduled"])): [Post!]! @all
}
```
この例では、クライアントが`includeUnscheduled: true`を指定するとPostモデルの`scheduled`スコープが外され、`schedule_at`カラムによる既定フィルタを無効にしてクエリを実行します。

**注意点**
- `@withoutGlobalScopes`はBoolean型の引数定義に付与し、値が`true`のときに指定したスコープ名一覧を外すという挙動になります。
- 複数のグローバルスコープ名を配列で指定可能です。
- 一部データのみ表示/非表示を切り替える管理者向け機能などに有用ですが、デフォルトのデータ不整合を招く恐れもあるため、必要最小限の利用に留めましょう（ソフトデリートの復元表示など）。

---

### @lazyLoad

**概要**  
取得済みの複数モデルに対し、後から遅延Eagerロード（Lazy Eager Loading）を行うディレクティブです。  
すでに得た結果集合に対し、関連をまとめてロードすることでN+1クエリを防ぎます。

**使用例**
```graphql
type Query {
  posts: [Post!]! @all @lazyLoad
}
```
このようにフィールドに付与して、resolve完了後に一括で関連をロードする、といった利用が可能です。

**注意点**
- `@lazyLoad`はリストフィールドに対して適用し、得られたコレクションに対してLaravelの`load()`的な処理を行います。
- 引数等で具体的なリレーション名を指定する仕様になっているはずなので、利用時はドキュメントを確認してください（例: `@lazyLoad(relation: "author")`など）。
- このディレクティブも`enhanceBuilder()`を介すフィールドでのみ意味があります。複数の関連をまとめてロードする際に便利ですが、初回取得クエリとは別に追加クエリが走る点は留意してください。

--- 