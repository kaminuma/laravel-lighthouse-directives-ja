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
EloquentリレーションをEagerロードするディレクティブです。N+1問題の回避や関連データの事前取得に利用します。

**使用例**
```graphql
type Query {
  users: [User!]! @all @with(relations: ["posts", "profile"])
}
```

**注意点**
- relations引数でリレーション名を配列で指定。

---

### @withCount

**概要**  
Eloquentリレーションの件数をEagerロードするディレクティブです。

**使用例**
```graphql
type Query {
  users: [User!]! @all @withCount(relations: ["posts"])
}
```

**注意点**
- relations引数でリレーション名を配列で指定。

---

### @lazyLoad

**概要**  
必要なタイミングでリレーションを遅延ロードするディレクティブです。

**使用例**
```graphql
type User {
  posts: [Post!]! @hasMany @lazyLoad
}
```

**注意点**
- パフォーマンス最適化やメモリ節約に有効。

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

### @morphTo

**概要**  
Eloquentの「MorphTo（ポリモーフィックな多対一）」リレーションを解決するディレクティブです。

**使用例**
```graphql
type Image {
  imageable: Imageable! @morphTo
}
```

**注意点**
- relationやscopes引数でカスタマイズ可能。

---

### @morphToMany

**概要**  
Eloquentの「MorphToMany（ポリモーフィックな多対多）」リレーションを解決するディレクティブです。

**使用例**
```graphql
type Post {
  tags: [Tag!]! @morphToMany
}
```

**注意点**
- typeやrelation、scopes引数が利用可能。

---

### @namespace

**概要**  
他のディレクティブで使用されるデフォルトの名前空間を再定義するディレクティブです。

**使用例**
```graphql
schema @namespace(field: "App\\GraphQL\\Fields")
```

**注意点**
- ディレクティブ名を名前空間にマッピングする繰り返し可能なディレクティブです。

---

### @namespaced

**概要**  
クエリやミューテーションのネストを可能にするno-opリゾルバを提供し、関心の分離による名前空間化に役立ちます。

**使用例**
```graphql
type Query {
  admin: AdminQuery @namespaced
}
```

**注意点**
- 論理的な名前空間化や関心の分離に利用します。

---

### @nest

**概要**  
子のArgResolverディレクティブに呼び出しを委譲するno-opネストされた引数リゾルバで、論理的なグループ化に役立ちます。

**使用例**
```graphql
type Mutation {
  updateUser(input: UpdateUserInput! @nest): User! @update
}
```

**注意点**
- ネストされた入力のグループ化やバリデーションに利用します。 