# 認証・認可（Auth & Permissions）

---

### @auth

**概要**  
GraphQLフィールドや型に認証（LaravelのAuth）を適用するディレクティブです。

**使用例**
```graphql
type Query {
  me: User @auth
}
```

**注意点**
- guard引数で認証ガードを指定可能。

---

### @guard

**概要**  
認証ガードを明示的に指定するディレクティブです。

**使用例**
```graphql
type Query {
  me: User @guard(name: "api")
}
```

**注意点**
- name引数でガード名を指定。

---

### @can

**概要**  
認可ポリシー（LaravelのPolicy）をGraphQLフィールドに適用するディレクティブです。

**使用例**
```graphql
type Query {
  user(id: ID!): User @can(ability: "view")
}
```

**注意点**
- ability引数で許可するアクションを指定。
- @can* family（@canIf, @canUnless等）も同様の認可制御に利用。

---

### @can* family

**概要**  
@canIf, @canUnlessなど、条件付き認可を実現するディレクティブ群です。

**使用例**
```graphql
type Query {
  user(id: ID!): User @canIf(ability: "view", condition: true)
}
```

**注意点**
- ability, condition等で柔軟な認可制御。

---

### @rules, @rulesForArray, @validator

これらのバリデーション系ディレクティブの詳細は「05_field_control.md」を参照してください。

---

### @canModel

**概要**  
Laravelポリシーの`モデル`権限（ability）チェックを行うディレクティブです。  
主に**作成**や**モデル全体**に対する操作の認可に使われ、現在のユーザーに指定モデルに対するabilityがあるか検証します。

**使用例**
```graphql
type Mutation {
  createPost(input: PostInput!): Post @canModel(ability: "create")
}
```
この例ではLaravelの`PostPolicy@create`が呼ばれ、`false`ならフィールド解決が拒否されます（例：`PostPolicy::create()`が管理者のみtrueを返す）。

**注意点**
- `@canModel`はフィールドの戻り値や`model`引数で推測されるモデルに対してポリシーをチェックします。
- スキーマ上の返却型と実際のモデル名が異なる場合、`model`引数でモデルクラス名を指定可能です。

---

### @canFind

**概要**  
引数で指定されたIDなどでモデルを**検索取得**し、そのモデルに対するポリシーabilityチェックを行うディレクティブです。  
主に**更新**や**削除**ミューテーション前に、対象データへのアクセス権を確認する用途に使われます。

**使用例**
```graphql
type Mutation {
  updatePost(
    input: UpdatePostInput! @spread
  ): Post 
    @canFind(ability: "edit", find: "input.id")
    @update
}
```
`find: "input.id"`は`input`引数内の`id`フィールドを使ってモデルを検索する指定です。
この例では`PostPolicy@edit`が、提供された`id`のPostモデルを引数に実行され、認可されなければ処理が中断されます。

**注意点**
- `find`引数でどの引数に主キーが含まれるかをドット表記で指定します。
- デフォルトではモデルが見つからないとエラーになりますが、`findOrFail: Boolean = true`で挙動を制御できます（falseにするとモデル未発見時にスキップ可能）。
- `@canFind`は**繰り返し適用可能 (repeatable)** なので、複数モデルに対するチェックもチェーンできます。

---

### @canQuery

**概要**  
指定した**検索条件**でモデルをクエリし、その結果に対してポリシーabilityチェックを行うディレクティブです。  
主に**一覧取得クエリ**を特定ユーザーの権限内に絞りたい場合に用います（例: 「自分がアクセスできるものだけ一覧取得」）。

**使用例**
```graphql
type Query {
  allUsers: [User!]! @canQuery(ability: "viewAny") @all
}
```
この場合、`UserPolicy@viewAny`で許可されたユーザーだけが全ユーザーを取得できます（許可されない場合空リストやエラーになります）。
また、`@canQuery`は引数に`@eq`や`@where`等が付与されたものを自動的にクエリ条件に組み込み、その結果に対してポリシーを適用します。

**注意点**
- `@canQuery`ではオプションで`scopes`を指定しクエリビルダにスコープを適用できます。
- `@canFind`との違いは、複数件の取得に対応しポリシーチェックを行う点です。
- クエリ結果が空の場合でもエラーとならず、そのまま空データを返す挙動となります。

---

### @canResolved

**概要**  
フィールドリゾルバが解決した**モデルインスタンス**に対してポリシーabilityチェックを行うディレクティブです。  
すでに取得済みのオブジェクトに対する**ビュー権限**の確認などに使います。

**使用例**
```graphql
type Query {
  post(id: ID! @whereKey): Post 
    @canResolved(ability: "view") 
    @find @softDeletes
}
```
まず`@find`で`id`に対応するPostモデルを取得し、その結果に対して`PostPolicy@view`を`@canResolved`が実行します。認可されなければエラーが発生し、許可されればPostデータが返ります。

**注意点**
- `@canResolved`はフィールド解決**後に**チェックを行うため、**ミューテーションでは使用しない**でください（副作用が既に起きた後に権限エラーとなる危険があるため）。クエリなど**データ取得系**フィールドでのみ利用し、ポリシーは`(User $actor, Model $resolved)`のように定義します。

---

### @canRoot

**概要**  
**親オブジェクト（ルートオブジェクト）**に対するポリシーabilityチェックで、特定のフィールドの閲覧を制限するディレクティブです。  
主に、ユーザー自身のみ閲覧可能な敏感情報フィールドに適用します。

**使用例**
```graphql
type Query {
  user(id: ID! @whereKey): User @find
}
type User {
  email: String! @canRoot(ability: "viewEmail")
}
```
`UserPolicy@viewEmail`がログインユーザー（アクター）とその`email`フィールドの親User（対象）を渡して呼ばれ、例えば「アクターのIDと対象UserのIDが一致するか」で許可可否を判定します。

**注意点**
- フィールドの親オブジェクトに対するポリシーを定義し、許可されない場合はそのフィールドのみ`null`が返るか、エラーになる挙動になります（設定による）。
- `@canRoot`も他の`@can`系と同様`repeatable`なので複数abilityを一度にチェック可能です。

---

### @whereAuth

**概要**  
現在の認証ユーザーに**紐づくデータのみ**結果をフィルタするディレクティブです。  
内部的には `whereHas` クエリでユーザーIDを紐付けてデータを絞り込みます。

**使用例**
```graphql
type Query {
  posts: [Post!]! @all @whereAuth(relation: "user")
}
```
ここではPostモデルが持つ`user`リレーション（投稿者）を使って、`user_id`が認証ユーザーと一致するPostのみを取得します。

**注意点**
- `relation`引数にはユーザーを参照するリレーション名を指定します（例：モデルによっては `"author"` など）。
- `guards`引数でどのガードのユーザーIDを使うか指定も可能です。
- このディレクティブは内部でクエリビルダを拡張するため、`@all`や`@hasMany`など**ビルダを強化するリゾルバ**と併用してください。 