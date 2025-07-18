# フィールド制御（Field & Schema Control）

---

### @hide

**概要**  
指定したスキーマ要素（フィールドや型など）を**スキーマから除外**し、クライアントから見えなくするディレクティブです。  
特定の環境でのみ隠したいフィールドがある場合に利用します。

**使用例**
```graphql
password: String! @hide(environments: ["production"])
```
この例では、production環境ではこのフィールドがスキーマ定義から削除されます（開発では見える）。環境指定を省略した場合は常に非公開となります。

**注意点**
- 環境判定には`APP_ENV`等Laravelの環境設定が使われます。
- `@hide`を付与した要素は対象環境では**スキーマレベルで存在しなくなる**ため、クライアントはそのフィールドを問い合わせることもできなくなります。
- 機密情報や将来的に廃止予定だが当面は使う内部フィールドなどに付与すると良いでしょう。

---

### @show

**概要**  
`@hide`と対になるディレクティブで、指定環境でのみ**スキーマに表示**する要素を定義します。  
つまり、デフォルトは非表示で、列挙した環境のときだけ有効になります。

**使用例**
```graphql
debugInfo: String @show(environments: ["local", "staging"])
```
この例ではローカルとステージング環境でのみスキーマに現れますが、本番では見えません。

**注意点**
- `@show`と`@hide`を同時に付ける必要はなく、要件に合わせてどちらか一方で十分です。
- `@show`を使った場合、指定環境以外では**スキーマから除外**される点は同じです。
- フィールドだけでなく型や引数にも適用できます（v6.46.0以降、タイプ・引数・入力型にも利用可）。

---

### @feature

**概要**  
機能フラグ（Feature Flag）によってスキーマ要素の公開/非公開を制御するディレクティブです。  
アプリケーションのある機能が有効なときだけフィールドを提供し、それ以外では隠すことができます。

**使用例**
```graphql
newFeatureField: String @feature(name: "new-feature")
```
サーバー側で`new-feature`フラグをオンにした環境ではこのフィールドがスキーマに現れ、オフなら隠されます。

**注意点**
- フラグの判定方法（環境変数、設定値など）はLighthouseの設定によります。
- このディレクティブを使うと、デプロイ後にフラグを切り替えるだけでスキーマの一部を有効化/無効化できるため、段階的リリースに便利です。ただしスキーマが変化するとクライアントとの互換性に影響するため、管理には注意してください。

---

### @rename

**概要**  
フィールドや引数の**内部名**を変更するディレクティブです。GraphQL上の名称と、Laravel内部（モデル属性やリゾルバ引数名）とのマッピングをカスタマイズできます。

**使用例**
```graphql
type Post {
  userId: ID @rename(name: "author_id")
}
```
この例ではGraphQLクエリでは`userId`でアクセスできますが、Lighthouseは内部でそれを`author_id`としてモデルに問い合わせます。引数の場合も同様に、例えばミューテーションで`userId: ID @rename(name: "author_id")`とすれば、受け取った`userId`を`author_id`キーに変換してEloquentに渡します。

**注意点**
- このディレクティブは**スキーマの外見を変えず**に裏側の名前だけを揃えるためのものです。クライアントにはリネーム前の名前で使わせつつ、サーバー側コードとの不整合を解消できます。
- 既存DBカラム名とGraphQLの命名規約を統一したい場合などに便利ですが、多用すると把握が難しくなるので必要最小限に留めましょう。

---

### @deprecated

**概要**  
スキーマの要素（フィールド、引数、列挙値など）を**非推奨**としてマークする標準ディレクティブです。将来的に削除予定のものに対し、クライアントに警告を与えます。

**使用例**
```graphql
type Query {
  allUsers: [User!]! @deprecated(reason: "Use `users`")
  users: [User!]!
}
```
クライアントが`allUsers`フィールドを見ると、非推奨である旨と代替として`users`を使うようメッセージが表示されます（GraphQLのイントロスペクションに含まれる）。

**注意点**
- `@deprecated`には省略可能な`reason`引数があります。可能な限り**理由と代替**を記載することが推奨されます。
- 非推奨化しても直ちにスキーマから消えるわけではなく、クライアントからも引き続き利用は可能です。ただし通常のGraphQLツールではdeprecated要素は隠されるため、互換維持期間を設けて最終的に削除する運用が望まれます。

---

### @enum

**概要**  
GraphQLのEnum値に**内部で対応する値**を割り当てるディレクティブです。GraphQL上の列挙子名と、アプリケーション内部で扱う値（文字列や数値）をズレなく対応させたい場合に使います。

**使用例**
```graphql
enum Role {
  ADMIN @enum(value: "admin")
  USER  @enum(value: "user")
}
```
クライアントは`ADMIN`/`USER`で指定しますが、サーバー上では`"admin"`/`"user"`という値として認識されます。これは例えばデータベースに`user.role = "admin"`と保存しているケース等で便利です。

**注意点**
- LighthouseにおいてはPHP 8.1+のバックドEnum（`UnitEnum`型）とも統合できますが、その場合は別途LaravelのEnum映射設定が必要になることがあります。`@enum`ディレクティブはあくまでGraphQL側のenum値と内部値の対応付けに留まり、バリデーションや変換はGraphQL実行時に自動で行われます。列挙子の数や名前を変更すると互換性に影響するため、慎重に運用してください。

---

### @scalar

**概要**  
独自のGraphQLスカラー型と、その実装クラス（`GraphQL\Type\Definition\ScalarType`を継承するPHPクラス）を関連付けるディレクティブです。既定のものにないカスタムスカラー（例: DateTimeなど）を定義する際に利用します。

**使用例**
```graphql
scalar DateTime @scalar(class: "App\\GraphQL\\Scalars\\DateTimeScalar")
```
この例で、自作の`DateTimeScalar`クラス（ScalarTypeを拡張しserialize/parse処理を定義）をGraphQLのDateTime型に紐付けます。

**注意点**
- Lighthouseではデフォルトで`app/GraphQL/Scalars`ネームスペース下を探す等のネーミング規約があります。その規約通りクラスを配置・命名している場合、`@scalar`ディレクティブは付与不要です（自動検出されます）。規約外の場所にクラスを置く場合や、複数のスカラーを一つのクラスで扱う特殊ケースでこのディレクティブを使ってください。

---

### @interface

**概要**  
GraphQLインターフェースの具体的な型解決（どの実装オブジェクト型に対応するか）をカスタム関数で行うディレクティブです。通常はLighthouseが自動解決しますが、特殊な場合に手動でタイプ判別ロジックを指定できます。

**使用例**
```graphql
interface Commentable @interface(resolveType: "App\\GraphQL\\Interfaces\\Commentable@resolveType") {
  id: ID!
}
```
`resolveType`にはクラス@メソッド形式でタイプ判別関数を指定します。この関数は実装オブジェクト（例えばあるコメントの`commentable`フィールドが実際PostなのかVideoなのか）を受け取り、対応するGraphQL型を返す必要があります。

**注意点**
- 通常、Lighthouseはインターフェースについては**型名ベース**か**`typename`フィールド**で解決します。`@interface`はそれでは不十分なケース（例えば単一テーブル継承など）でのみ利用してください。解決関数内ではLighthouseのTypeRegistryを使って適切なTypeオブジェクトを返します。高度な機能のため、基本的には公式ドキュメントのインターフェース章を理解した上で使いましょう。

---

### @union

**概要**  
GraphQLユニオン型の具体的な型解決をカスタム関数で行うディレクティブです。ユニオン型は複数の型のどれかになるフィールドを表しますが、その判別ロジックを明示できます。

**使用例**
```graphql
union SearchResult @union(resolveType: "App\\GraphQL\\Unions\\SearchResult@resolveType") = Post | User
```
`resolveType`で指定した関数が、実際の値（例えばある検索結果オブジェクト）がPost型かUser型かを判定し、対応する型を返します。

**注意点**
- インターフェースと同様、Lighthouseは通常自動でユニオン型を判別しますが、オブジェクトがどちらにも共通点がなく判定が難しい場合に本ディレクティブでカスタムロジックを書けます。解決関数の実装方法はインターフェースの場合と同じく、TypeRegistry等から型を取得して返す形になります。既定の挙動で足りるケースが多いため、必要な場合のみ導入してください。

--- 

### @field

**概要**  
フィールドにカスタムリゾルバ関数（クラス@メソッド形式）を割り当てるディレクティブです。Eloquentモデルやデフォルトの自動解決ではなく、独自のPHP関数でフィールド値を返したい場合に利用します。

**主な用途**
- 複雑なビジネスロジックや外部API連携など、モデル自動解決では対応できない場合
- 型やフィールドごとに独自の処理を割り当てたい場合

**使用例**
```graphql
type Query {
  customGreeting(name: String!): String @field(resolver: "App\\GraphQL\\Queries\\Greeting@resolve")
}
```
この例では、`App\GraphQL\Queries\Greeting`クラスの`resolve`メソッドが呼ばれ、引数`name`を使って任意の文字列を返せます。

**注意点**
- `resolver`引数で「クラス@メソッド」形式を指定します。クラスは`__invoke`メソッドでもOKです。
- クラスは`app/GraphQL/Queries`や`app/GraphQL/Mutations`等、Lighthouseの自動検出パスに置くのが推奨です。
- 型や引数の一致に注意し、GraphQLスキーマとPHP側のシグネチャを揃えてください。
- Eloquentモデルの自動解決や@method等と併用する場合は、どちらが優先されるか設計に注意しましょう。
- 複雑なロジックや外部サービス連携、認可・バリデーションなどもこのカスタムリゾルバ内で自由に実装できます。 

---

### @method

**概要**  
Eloquentモデルの特定メソッドをリゾルバとして割り当てるディレクティブです。

**使用例**
```graphql
type User {
  fullName: String! @method(name: "getFullName")
}
```

**注意点**
- name引数でメソッド名を指定。

---

### @model

**概要**  
GraphQL型とEloquentモデルを明示的に紐付けるディレクティブです。

**使用例**
```graphql
type User @model(class: "App\\Models\\User") {
  ...
}
```

**注意点**
- class引数でモデルクラスを指定。

---

### @rules

**概要**  
入力値のバリデーションルールを指定するディレクティブです。

**使用例**
```graphql
type Mutation {
  createUser(input: CreateUserInput! @rules(apply: ["required", "email"])): User
}
```

**注意点**
- apply引数でバリデーションルールを配列で指定。

---

### @rulesForArray

**概要**  
配列入力の各要素にバリデーションルールを適用するディレクティブです。

**使用例**
```graphql
type Mutation {
  createUsers(input: [CreateUserInput!]! @rulesForArray(apply: ["required"])): [User]
}
```

**注意点**
- apply引数でルール指定。

---

### @validator

**概要**  
カスタムバリデータクラスを指定してバリデーションを行うディレクティブです。

**使用例**
```graphql
type Mutation {
  createUser(input: CreateUserInput! @validator(class: "App\\GraphQL\\Validators\\UserValidator")): User
}
```

**注意点**
- class引数でバリデータクラスを指定。 