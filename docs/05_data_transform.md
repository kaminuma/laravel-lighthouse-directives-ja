# データ変換と入力操作ディレクティブ

---

このカテゴリには、リゾルバによって処理されたりデータベースに保存されたりする前に、入力データを変更または変換するディレクティブが含まれており、データの一貫性を確保し、入力処理を簡素化します。

## データ変換ディレクティブ

### @convertEmptyStringsToNull

入力された空文字列""をnullに置き換えます。フォームデータの処理に役立ちます。

**オプション**: なし

```graphql
type Mutation {
  createUser(
    name: String
    email: String! @convertEmptyStringsToNull
  ): User!
}
```

### @trim

指定された入力の先頭と末尾から空白を削除します。ユーザー入力のクリーニングに最適です。

**オプション**: なし

```graphql
type Mutation {
  createUser(
    name: String! @trim
    email: String! @trim
  ): User!
}
```

### @hash

Laravelのハッシュ機能を使用して引数値を変換します。パスワードの安全な保存に不可欠です。

**オプション**: なし

```graphql
type Mutation {
  createUser(
    name: String!
    email: String!
    password: String! @hash
  ): User!
}
```

### @globalId

ID/タイプとグローバルIDの間で変換を行い、フィールドではエンコードし、引数ではデコードします。Relay仕様に準拠したグローバルIDを実装するために使用します。

**オプション**: なし

フィールド上での使用:

```graphql
type User {
  id: ID! @globalId
}
```

引数での使用:

```graphql
type Mutation {
  updateUser(id: ID! @globalId): User!
}
```

## 入力操作ディレクティブ

### @spread

ネストされた入力オブジェクトのフィールドをその親の引数にマージします。複雑な入力構造を扱う際に便利です。

**オプション**: なし

```graphql
type Mutation {
  createUser(
    input: CreateUserInput! @spread
  ): User!
}

input CreateUserInput {
  name: String!
  email: String!
  password: String!
}
```

### @rename

クライアントの視点からスキーマを変更することなく、フィールドまたは引数の内部使用名を変更します。データベースのカラム名とGraphQLフィールド名の違いを橋渡しするのに役立ちます。

**オプション**:
- `attribute`: 元のフィールド名からマップする内部的な名前

```graphql
type User {
  fullName: String! @rename(attribute: "name")
}
```

引数での使用:

```graphql
type Mutation {
  createUser(
    fullName: String! @rename(attribute: "name")
  ): User!
}
```

### @inject

コンテキストオブジェクトから引数に値を注入します。

```graphql
type Mutation {
  createPost(
    title: String!
    content: String!
    user_id: ID! @inject(context: "user.id")
  ): Post!
}
```

### @drop

ユーザーが与えた値を無視し、リゾルバに渡されないようにします。

```graphql
type Mutation {
  createUser(
    name: String!
    email: String!
    password: String!
    adminFlag: Boolean @drop
  ): User!
}
```

### @bind

Laravelのルートモデルバインディングと同様に、引数値を対応するモデルまたは他の値に置き換えてからリゾルバに渡します。

```graphql
type Mutation {
  updatePost(
    postId: ID! @bind(argument: "post")
    title: String!
  ): Post! @update(relation: "post")
}
```

### @upload

指定されたファイルをストレージにアップロードし、引数を削除して、返されたパスを属性キーに設定します。

```graphql
type Mutation {
  userAvatar(
    id: ID!
    image: Upload! @upload(disk: "public", path: "avatars")
  ): User! @update
}
```

---

Lighthouseは、スキーマ内で直接入力データを変換および操作するための豊富なディレクティブセットを提供します。これにより、スキーマレベルでのデータ衛生、標準化、および変換（例：パスワードのハッシュ化、空文字列からnullへの変換、ファイルアップロードの処理、グローバルID変換）が可能になります。

これにより、カスタムリゾルバにおける反復的で潜在的に一貫性のないデータ処理ロジックの必要性が大幅に削減されます。様々なリゾルバでカスタムPHPコードを記述して入力データをクリーンアップ、検証、変換、または準備する代わりに、これらの操作はGraphQLスキーマで直接宣言できます。

これにより、データ操作ロジックが一元化され、スキーマがデータ期待値に関してより自己文書化され、API全体でデータの一貫性が確保されます。この機能は、一般的なデータ操作タスクをアプリケーションロジックからスキーマレイヤーにオフロードすることで開発を合理化します。

これにより、散在するデータ変換コードから生じる可能性のある不整合やエラーの可能性が減少し、より堅牢で予測可能なAPIに貢献します。

次の章では、バリデーションに関連するディレクティブについて詳しく解説します。
