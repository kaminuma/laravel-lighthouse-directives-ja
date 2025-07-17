# 入力と検証（Input & Validation）

---

### @spread

**概要:** ネストした入力オブジェクトのフィールド群を親の引数に展開（マージ）するディレクティブです。これにより、クライアントからはネストした入力を渡しつつ、リゾルバではフラットな引数配列を扱えます。
**使用例:** 例えば更新ミューテーションでInput型を使う場合:

```graphql
type Mutation {
  updatePost(id: ID!, input: PostInput! @spread): Post! @update
}
input PostInput {
  title: String!
  content: PostContent @spread
}
input PostContent {
  imageUrl: String
}
```

クライアントは`input { title, content { imageUrl } }`の形で送りますが、Lighthouse内部では`id`, `title`, `imageUrl`が同一階層にフラット化され、リゾルバに渡されます。
**注意点:** `@spread`による展開は他の引数ディレクティブ適用後に行われます。つまりバリデーションや変換（@rulesや@trim等）をすべて適用した後でフィールドをマージします。スキーマ定義上はネスト構造をそのまま記述し、クライアント利用も通常通りですが、サーバー内部処理のみ平坦化される点を押さえておきましょう。

### @rules

**概要:** Laravelのバリデーションルールを用いて引数や入力フィールドの検証を行うディレクティブです。`apply`引数にルール文字列（またはカスタムルールクラス名）を配列で指定します。
**使用例:** 国コード（2文字）を検証する例:

```graphql
type Query {
  users(countryCode: String @rules(apply: ["string", "size:2"])): [User!]! @all
}
```

この場合、`countryCode`が文字列かつ長さ2であることを自動検証します。違反した場合、リクエストは処理されずバリデーションエラーが返ります。
**注意点:** `apply`にはLaravelのビルトインルール（例: "email", "min:5"など）やカスタムルールクラス名を指定できます。メッセージのカスタマイズは`messages`引数でルール毎に設定可能です。`exclude_if`や`exclude_unless`のように入力自体を除外するタイプのルールはサポートされないので注意してください（その場合はArgTransformerやFieldMiddlewareを使用）。

### @rulesForArray

**概要:** 配列（リスト）そのものに対してLaravelバリデーションを適用するディレクティブです。配列の要素数の制約など、リスト型特有の検証に用います。
**使用例:** 例えばフレーバーの一覧が3種類以上指定されていることを検証するミューテーション:

```graphql
type Mutation {
  saveIcecream(
    flavors: [IcecreamFlavor!]! @rulesForArray(apply: ["min:3"])
  ): Icecream
}
```

`flavors`配列の要素数が3未満ならバリデーションエラーになります（`min:3`ルール）。
**注意点:** 基本的な指定方法や`apply`/`attribute`/`messages`引数の使い方は`@rules`と同様です。`@rulesForArray`は配列そのものへのルール適用（要素数、配列全体の必須/空許可など）に使い、配列中の各要素の検証（例えば各要素がメールアドレス形式か等）は`@rules`を要素型に付与する形で組み合わせて使うことになります。

### @validator

**概要:** PHPクラスによるカスタムバリデーションを行うディレクティブです。複雑な検証ロジックやコンテキスト依存のチェックが必要な場合に、フォームリクエスト風のValidatorクラスを用意して適用できます。
**使用例:** `RegisterInputValidator`クラスを使って会員登録入力を検証する場合:

```graphql
input RegisterInput @validator(class: "App\\GraphQL\\Validators\\RegisterInputValidator") {
  email: String!
  password: String!
  password_confirmation: String!
}
```

このように`@validator`の`class`引数にバリデータクラス名を指定すると、該当クラスで定義されたrulesやメッセージで入力全体を検証します（LaravelのFormRequestに近い）。
**注意点:** `class`を省略した場合、デフォルトで入力オブジェクト名＋"Validator"というクラスを自動解決しようとします。例えば上記`RegisterInput`なら`App\GraphQL\Validators\RegisterInputValidator`を探します。フィールド単位に付与した場合は`親タイプ名{FieldName}Validator`を探します。バリデータクラスではLaravelの Validator インスタンス相当の機能（rulesメソッドやメッセージ、authorizeなど）を実装できます。

### @trim

**概要:** 入力文字列の前後の空白を取り除く（trimする）ディレクティブです。トリミングは入力値のクレンジングとしてしばしば必要になるため、スキーマ上で宣言的に適用できます。
**使用例:** 例えばユーザー名入力の前後空白を自動除去するには: `name: String @trim` とします。以下のようにクライアントが空白付きで送信しても、Resolverには両端の空白が除去された`name`が渡ります。

```graphql
type Mutation {
  createUser(name: String @trim): User
}
```

**注意点:** フィールド定義（FIELD_DEFINITION）に`@trim`を付与すると、そのフィールドに渡される全ての文字列入力に再帰的にtrimを適用します。一つ一つの引数に付けなくても、例えば`createUser(input: UserInput!): User @trim`のようにすると、`UserInput`内の全ての文字列型がトリミング対象となります。全フィールドでこの処理をしたい場合は、`lighthouse.php`設定の`field_middleware`にTrimDirectiveを登録する方法もあります。

### @hash

**概要:** Laravelのハッシュ機能を用いて引数の値を変換（ハッシュ化）するディレクティブです。主にパスワード入力のハッシュ化など、保存前に値を安全に変換する用途に使われます。
**使用例:** パスワードをハッシュ化して保存するミューテーションでは次のようにします:

```graphql
type Mutation {
  createUser(name: String!, password: String! @hash): User!
}
```

クライアントから平文パスワードを受け取っても、リゾルバには自動的にハッシュ化された文字列が渡されます（Laravelの`Hash::make`と同等）。
**注意点:** 使用されるハッシュアルゴリズムはLaravelの`config/hashing.php`で定義されたデフォルトドライバーです。`@hash`は一方向の不可逆変換ですので、パスワードやトークンなど平文を保存しないフィールドにのみ使ってください。

### @convertEmptyStringsToNull

**概要:** 空文字列 (`""`) を自動的に`null`に置き換えるディレクティブです。`null`と空文字を同等に扱いたい場合に利用します（Laravelの`ConvertEmptyStringsToNull`ミドルウェアと同様の機能をフィールド単位で適用するイメージです）。
**使用例:** 例えばタイトルが空文字の場合は未入力と見なしたいとき: `title: String @convertEmptyStringsToNull` とします。`title: ""`が送られた場合、リゾルバには`title: null`として渡ります。
**注意点:** フィールド（FIELD_DEFINITION）に付与するとそのフィールド配下のすべての入力について再帰的に空文字→null変換を適用します。ただしnon-nullな引数については挙動に注意が必要です。非null型引数に対しフィールドに`@convertEmptyStringsToNull`を付けただけでは空文字はそのまま通ります（nullは許されないため）。非null引数にも強制的に適用したい場合は、その引数定義自体に直接`@convertEmptyStringsToNull`を付与してください。

### @inject

**概要:** GraphQLのコンテキストから値を取り出し、引数に自動注入するディレクティブです。これによりクライアントから渡させたくない値（例：認証ユーザーID）をサーバ側で強制的にセットできます。
**使用例:** 現在ログイン中ユーザーのIDを`user_id`引数として注入するミューテーション:

```graphql
type Mutation {
  createPost(title: String!, content: String!): Post!
    @inject(context: "user.id", name: "user_id")
    @create
}
```

上記では`context: "user.id"`によりGraphQLコンテキスト中の`user->id`プロパティを取得し、引数名`user_id`にセットしています。クライアントは`user_id`を指定せずに済み、サーバ側で確実に自分のIDが付与されます。
**注意点:** `context`にはコンテキストオブジェクト内のプロパティパスをドット区切りで指定します。`name`は注入先の引数名で、ネストした入力に入れる場合ドット表記（例：`"input.user_id"`）が使えます。この仕組みにより、例えばミューテーションで他人のIDを強制しようとしても入力が上書きされるためセキュリティ向上に繋がります。

### @drop

**概要:** ユーザーが指定した引数の値を無視（破棄）し、リゾルバには渡さないようにするディレクティブです。クライアントから受け取るが使用しないフィールドや、サーバ側で強制決め打ちするため受け取る意味がない引数などで用います。
**使用例:** 例えばミューテーションで`userId`引数を受け取るが、実際には利用せずコンテキストのユーザーIDを使いたい場合: `userId: ID @drop` と宣言できます。クライアントは一応`userId`を渡せますが、サーバ側ではその値は破棄され`null`として扱われます（代わりに上記の@inject等でコンテキストIDを使うことが可能）。
**注意点:** `@drop`が付いた引数は常にリゾルバへ`null`として渡されます。これによりユーザー入力を事実上無効化できます。単に無視されるだけなので、クライアントには特別なエラーは返りません。意図しない値をクライアントが送っても影響を与えないようにするためのセキュリティ的用途で活用できます。 