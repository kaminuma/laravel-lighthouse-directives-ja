# リレーション関連（Relationships）

---

### @belongsTo

**概要:** Eloquentの「BelongsTo（1対多逆方向）」リレーションを通じて関連データを解決するディレクティブです。あるモデルが**属している**関連（外部キーを持つ側）を取得する際に使います。
**使用例:** 例えば投稿モデルの`author`（ユーザー）を取得するフィールド: `author: User @belongsTo`。この場合Postモデルに`author()`メソッド（`return $this->belongsTo(User::class)`）が定義されている前提です。
**注意点:** GraphQLスキーマのフィールド名とEloquentモデルのリレーション名が同じである必要があります。異なる場合、`relation`引数でモデル側のメソッド名を指定してください。また、`scopes`引数でリレーションにクエリスコープを適用することも可能です。

### @belongsToMany

**概要:** Eloquentの「BelongsToMany（多対多）」リレーションを通じて関連データを解決するディレクティブです。中間テーブルを介した多対多の関連を取得します。
**使用例:** 例えばユーザーが所属するロールを取得するフィールド: `roles: [Role!]! @belongsToMany`。Userモデルに`roles()`メソッド（`belongsToMany`）が定義されていれば、GraphQLクエリでそのユーザーのロール一覧を取得できます。
**注意点:** デフォルトでは関連名＝フィールド名を前提としますが、異なる場合は`relation`引数でモデル側の関数名を指定します。`type`引数でページネーション形式（`PAGINATOR`・`SIMPLE`・`CONNECTION`）を指定すると関連コレクションをページネートして取得できます。`defaultCount`や`maxCount`でクライアントが取得できる件数のデフォルト値・上限を設定可能です。なお、中間テーブルのカラム（pivot情報）をGraphQLで扱う場合、モデル側で`withPivot`を設定し、スキーマ上にpivot用の型を定義してフィールドを持たせることができます（pivotフィールドは経路によって存在しない場合もあるためnullableにするのが望ましいです）。

### @hasOne

**概要:** Eloquentの「HasOne（1対1）」リレーションを解決するディレクティブです。モデルが**一つだけ所有する**関連を取得します。
**使用例:** 例えばユーザーの電話番号を取得するフィールド: `phone: Phone @hasOne`。Userモデルに`phone()`メソッド（`hasOne`）が定義されていれば、GraphQLでUser型から関連するPhoneモデルを取得できます。
**注意点:** フィールド名とモデルのリレーション名が異なる場合、`relation`でリレーションメソッド名を指定できます。`scopes`引数も利用可能で、関連取得時にスコープを適用できます。

### @hasMany

**概要:** Eloquentの「HasMany（1対多）」リレーションを解決するディレクティブです。モデルが**多数持つ**関連コレクションを取得します。
**使用例:** 例えばユーザーの投稿一覧を取得するフィールド: `posts: [Post!]! @hasMany`。Userモデルに`posts()`メソッド（`hasMany`）が定義されていれば、GraphQLで投稿リストを取得できます。
**注意点:** `@hasMany`では`type`引数でページネーション形式を指定可能です。例えば `@hasMany(type: PAGINATOR)` とすればオフセットベースのページネータで結果を取得できます。フィールド名とリレーション名が異なる場合は`relation`で対応できます。その他`defaultCount`や`maxCount`、`scopes`も`@belongsToMany`同様に使用できます。

### @hasOneThrough

**概要:** Eloquentの「HasOneThrough（中間テーブル経由1対1）」リレーションを解決するディレクティブです。2段階離れた関連先（例：Mechanic -> Car -> Owner）の1対1リレーションを取得します。
**使用例:** 例えば自動車整備士(Mechanic)から、その整備士が担当する車の所有者(Owner)を取得するフィールド: `carOwner: Owner! @hasOneThrough`。Mechanicモデルに`carOwner()`メソッド（`hasOneThrough`でOwnerを返す）があれば機能します。
**注意点:** フィールド名とリレーションメソッド名が異なる場合は`relation`で指定可能です。`@hasOneThrough`も`scopes`引数でクエリスコープを適用できます。利用方法やページネーションオプションは基本的に`@hasMany`と同様です。

### @hasManyThrough

**概要:** Eloquentの「HasManyThrough（中間テーブル経由1対多）」リレーションを解決するディレクティブです。2段階離れた関連先（例：Country -> Mechanic -> Car）の多対多（正確には1対多を中継した多対多）コレクションを取得します。
**使用例:** （例）ある国が保有する全ての車一覧を取得するフィールド: `cars: [Car!]! @hasManyThrough` （Countryモデルに`cars()`メソッドが定義され、内部でCountry -> Mechanic -> Carのリレーションを返す）。
**注意点:** `@hasManyThrough`も`@hasMany`と同様に`type`によるページネーション指定や`relation`の名前調整、`scopes`の適用が可能です。「Usage is the same as @hasMany」とある通り使い方は`@hasMany`と同じです。

### @morphOne

**概要:** Eloquentの「MorphOne（ポリモorphicな1対1）」リレーションを解決するディレクティブです。多態(one-to-one)関係で、1つのモデルが別の複数タイプのモデルに属するケースの1対1を取得します。
**使用例:** 例えば画像(Image)モデルが投稿(Post)またはユーザー(User)に属する場合、投稿側から関連画像を取得するフィールド: `image: Image! @morphOne`。Postモデルに`image()`メソッド（`morphOne`）が定義されていれば、GraphQLで該当Imageを取得できます。
**注意点:** 他のリレーション同様、フィールド名とリレーション名が異なる場合は`relation`で指定し、`scopes`でクエリスコープを適用できます。逆側（画像側）では通常`@morphTo`を使って対になる多態リレーションを解決します。

### @morphMany

**概要:** Eloquentの「MorphMany（ポリモorphicな1対多）」リレーションを解決するディレクティブです。例えば1つのモデル（投稿など）に複数種類のモデル（画像やコメントなど）が属するケースの1対多を取得します。
**使用例:** 投稿が複数の画像を持つ場合、投稿モデルのフィールド: `images: [Image!] @morphMany` によって関連画像のリストを取得できます。Postモデルには`images()`メソッド（`morphMany`）が必要です。
**注意点:** `@morphMany`も`type`引数によるページネーションが可能で、`PAGINATOR`・`SIMPLE`・`CONNECTION`形式を選べます。`relation`や`scopes`引数の使い方も他のリレーション同様です。多態リレーションの場合、関連先が複数のモデル型になるため、GraphQL上は`union`型やインターフェイスで表現することがあります（例ではImageの反対側を`union Imageable = Post | User`で定義し、Image側で`imageable: Imageable! @morphTo`としています）。

### @morphTo

**概要:** Eloquentの「MorphTo（ポリモorphicな所属）」リレーションを解決するディレクティブです。多態リレーションにおいて、**属している側**から、その属先（複数の可能性がある）を取得します。
**使用例:** 例えばImageモデルがPostまたはUserに属しうる場合、Image型のフィールド: `imageable: Imageable! @morphTo` とし、GraphQL上でImageableを`union Imageable = Post | User`等で定義します。これにより、Imageからそれが属する具体的なオブジェクト(PostまたはUser)を取得できます。
**注意点:** `@morphTo`では`relation`引数でモデル側のメソッド名を変更できます。なお、多態リレーションの解決にはバックエンドでタイプを判別する必要がありますが、Lighthouseが自動で型を解決します（状況によっては`__typename`や`@interface`/`@union`での型解決カスタマイズが利用されます）。

### @morphToMany

**概要:** Eloquentの「MorphToMany（ポリモorphicな多対多）」リレーションを解決するディレクティブです。例えば、タグ(Tag)モデルが投稿(Post)やビデオ(Video)など複数種にまたがって多対多で関連付けられるような場合に使います。
**使用例:** 例えばPost型にタグ一覧フィールドを持たせる場合: `tags: [Tag!]! @morphToMany`。Postモデルに`tags()`（`morphToMany`）が定義されていれば、GraphQLで関連タグを取得できます（同じタグがVideoにも付いている可能性がある）。
**注意点:** `@morphToMany`も他の*Many系と同様にページネーション用の`type`（`MorphToManyType`）や`defaultCount`/`maxCount`を指定できます。`relation`や`scopes`引数もサポートされています。タグ付けのようなシナリオではPivot（中間テーブル）にメタ情報があることも多いですが、その場合はPivotデータを型として別途定義し、Edge拡張や`pivot`フィールドで提供することも可能です（詳細はリレーションディレクティブのドキュメント参照）。 