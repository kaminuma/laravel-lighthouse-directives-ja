# リレーション（Relationships）

---

### @belongsTo

**概要**  
Eloquentの「BelongsTo（1対多逆方向）」リレーションを通じて関連データを解決するディレクティブです。  
あるモデルが**属している**関連（外部キーを持つ側）を取得する際に使います。

**使用例**
```graphql
# 投稿モデルのauthor（ユーザー）を取得
author: User @belongsTo
```
この場合、Postモデルに`author()`メソッド（`return $this->belongsTo(User::class)`）が定義されている前提です。

**注意点**
- GraphQLスキーマのフィールド名とEloquentモデルのリレーション名が同じである必要があります。
- 異なる場合は`relation`引数でモデル側のメソッド名を指定してください。
- `scopes`引数でリレーションにクエリスコープを適用することも可能です。

---

### @belongsToMany

**概要**  
Eloquentの「BelongsToMany（多対多）」リレーションを通じて関連データを解決するディレクティブです。  
中間テーブルを介した多対多の関連を取得します。

**使用例**
```graphql
# ユーザーが所属するロール一覧を取得
roles: [Role!]! @belongsToMany
```
Userモデルに`roles()`メソッド（`belongsToMany`）が定義されていれば、GraphQLクエリでそのユーザーのロール一覧を取得できます。

**注意点**
- デフォルトでは関連名＝フィールド名が前提ですが、異なる場合は`relation`引数でモデル側の関数名を指定します。
- `type`引数でページネーション方式（`PAGINATOR`・`SIMPLE`・`CONNECTION`）を指定可能。
- `defaultCount`や`maxCount`でクライアントが取得できる件数のデフォルト値・上限を設定できます。
- 中間テーブルのカラム（pivot情報）をGraphQLで扱う場合、モデル側で`withPivot`を設定し、スキーマ上にpivot用の型・フィールドを定義してください（pivotフィールドはnullable推奨）。

---

### @hasOne

**概要**  
Eloquentの「HasOne（1対1）」リレーションを解決するディレクティブです。  
モデルが**一つだけ所有する**関連を取得します。

**使用例**
```graphql
# ユーザーの電話番号を取得
phone: Phone @hasOne
```
Userモデルに`phone()`メソッド（`hasOne`）が定義されていれば、User型から関連するPhoneモデルを取得できます。

**注意点**
- フィールド名とモデルのリレーション名が異なる場合、`relation`でリレーションメソッド名を指定できます。
- `scopes`引数も利用可能で、関連取得時にスコープを適用できます。

---

### @hasMany

**概要**  
Eloquentの「HasMany（1対多）」リレーションを解決するディレクティブです。  
モデルが**多数持つ**関連コレクションを取得します。

**使用例**
```graphql
# ユーザーの投稿一覧を取得
posts: [Post!]! @hasMany
```
Userモデルに`posts()`メソッド（`hasMany`）が定義されていれば、GraphQLで投稿リストを取得できます。

**注意点**
- `type`引数でページネーション方式を指定可能（例：`@hasMany(type: PAGINATOR)`）。
- フィールド名とリレーション名が異なる場合は`relation`で対応。
- `defaultCount`や`maxCount`、`scopes`も`@belongsToMany`同様に利用できます。

---

### @hasOneThrough

**概要**  
Eloquentの「HasOneThrough（中間テーブル経由1対1）」リレーションを解決するディレクティブです。  
2段階離れた関連先（例：Mechanic → Car → Owner）の1対1リレーションを取得します。

**使用例**
```graphql
# 整備士が担当する車の所有者を取得
carOwner: Owner! @hasOneThrough
```
Mechanicモデルに`carOwner()`メソッド（`hasOneThrough`でOwnerを返す）があれば機能します。

**注意点**
- フィールド名とリレーションメソッド名が異なる場合は`relation`で指定可能。
- `scopes`引数でクエリスコープも適用できます。
- ページネーションやオプションは基本的に`@hasMany`と同様です。

---

### @hasManyThrough

**概要**  
Eloquentの「HasManyThrough（中間テーブル経由1対多）」リレーションを解決するディレクティブです。

**使用例**
```graphql
type Country {
  cars: [Car!]! @hasManyThrough
}
```

**注意点**
- @hasManyと同様にtypeやrelation、scopesが利用可能。

---

### @morphOne

**概要**  
Eloquentの「MorphOne（ポリモーフィックな1対1）」リレーションを解決するディレクティブです。

**使用例**
```graphql
type Post {
  image: Image! @morphOne
}
```

**注意点**
- relationやscopes引数でカスタマイズ可能。

---

### @morphMany

**概要**  
Eloquentの「MorphMany（ポリモーフィックな1対多）」リレーションを解決するディレクティブです。

**使用例**
```graphql
type Post {
  images: [Image!] @morphMany
}
```

**注意点**
- typeやrelation、scopes引数が利用可能。

---

### @morphTo

**概要**  
Eloquentの「MorphTo（ポリモーフィックな所属）」リレーションを解決するディレクティブです。  
多態リレーションにおいて、**属している側**からその属先（複数の可能性あり）を取得します。

**使用例**
```graphql
# 画像モデルが投稿またはユーザーに属する場合
imageable: Imageable! @morphTo
```
GraphQL上でImageableを`union Imageable = Post | User`などで定義します。

**注意点**
- `@morphTo`では`relation`引数でモデル側のメソッド名を変更可能。
- 多態リレーションの解決にはバックエンドでタイプ判別が必要ですが、Lighthouseは自動で型を解決します（状況によっては`__typename`や`@interface`/`@union`での型解決カスタマイズも利用されます）。

---

### @morphToMany

**概要**  
Eloquentの「MorphToMany（ポリモーフィックな多対多）」リレーションを解決するディレクティブです。  
例えば、タグ（Tag）モデルが投稿（Post）やビデオ（Video）など複数種にまたがって多対多で関連付けられる場合に使います。

**使用例**
```graphql
# 投稿型にタグ一覧フィールドを持たせる場合
tags: [Tag!]! @morphToMany
```
Postモデルに`tags()`（`morphToMany`）が定義されていれば、GraphQLで関連タグを取得できます。

**注意点**
- `@morphToMany`も他の*Many系と同様にページネーション用の`type`や`defaultCount`/`maxCount`を指定できます。
- `relation`や`scopes`引数もサポートされています。
- タグ付けのようなシナリオではPivot（中間テーブル）にメタ情報がある場合も多いですが、その場合はPivotデータを型として別途定義し、Edge拡張や`pivot`フィールドで提供することも可能です（詳細はリレーションディレクティブのドキュメント参照）。

--- 