# バリデーションディレクティブ

---

これらのディレクティブは、Laravelの強力なバリデーションルールをGraphQLスキーマ内で直接適用することで、データの整合性を強制することに特化しており、入力データが事前に定義された基準を満たしていることを保証します。

## バリデーションディレクティブ

### @rules

Laravelバリデーションルールを使用して引数を検証します。入力値が特定の条件を満たすことを確認します。

**オプション**:
- `apply`: 適用するLaravelバリデーションルールの配列
- `messages`: カスタムエラーメッセージ
- `passWhen`: 特定の条件を満たす場合のみバリデーションを実行

```graphql
type Mutation {
  createUser(
    name: String! @rules(apply: ["required", "string", "max:255"])
    email: String! @rules(apply: ["required", "email", "unique:users,email"])
    password: String! @rules(apply: ["required", "min:8"])
  ): User!
}
```

特定の条件下でのみバリデーションを実行するために、`passWhen`を使用することもできます：

```graphql
type Mutation {
  updateUser(
    id: ID!
    name: String
    email: String @rules(apply: ["email", "unique:users,email"], passWhen: { filled: "email" })
    password: String @rules(apply: ["min:8"], passWhen: { filled: "password" })
  ): User! @update
}
```

### @rulesForArray

Laravelの組み込みバリデーションを使用して配列自体にバリデーションを実行します。配列の要素数や構造を検証するのに役立ちます。

**オプション**:
- `apply`: 適用するLaravelバリデーションルールの配列（配列全体に対するバリデーション）

```graphql
type Mutation {
  createPost(
    title: String!
    tags: [String!]! @rulesForArray(apply: ["required", "min:1", "max:10"])
  ): Post!
}
```

配列の各要素にルールを適用する場合：

```graphql
type Mutation {
  attachTags(
    post_id: ID!
    tags: [String!]! @rulesForArray(apply: ["required", "distinct"], forEach: ["string", "max:50"])
  ): Post!
}
```

### @validator

PHPクラスを通じてカスタムバリデーションルールを提供します。より複雑なバリデーションロジックを実装する場合に使用します。

**オプション**:
- `class`: バリデーションロジックを含むクラスの完全修飾名

```graphql
type Mutation {
  createUser(
    input: CreateUserInput! @spread @validator(class: "App\\GraphQL\\Validators\\CreateUserInputValidator")
  ): User!
}
```

カスタムバリデータクラスの例：

```php
namespace App\GraphQL\Validators;

use Nuwave\Lighthouse\Validation\Validator;

class CreateUserInputValidator extends Validator
{
    /**
     * Return the validation rules.
     *
     * @return array<string, array<mixed>>
     */
    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'email', 'unique:users,email'],
            'password' => ['required', 'min:8', 'confirmed'],
            'password_confirmation' => ['required'],
        ];
    }
}
```

---

Laravelバリデーションディレクティブの明確な組み込みは、Lighthouseが堅牢な入力バリデーションにコミットしていることを示しています。バリデーションルールをスキーマ内で直接定義できるようにすることで、APIゲートウェイレベルでデータの整合性を確保し、無効なデータや悪意のあるデータがアプリケーションロジックに到達するのを防ぎ、より安全で信頼性の高いAPIを促進します。

入力バリデーションは、APIのセキュリティとデータの整合性にとって重要な側面です。Lighthouseは、Laravelの強力で使い慣れたバリデーションシステムをGraphQLスキーマに直接統合することで、ビジネスロジックが実行されたりデータが永続化されたりする前に、入力データが事前定義されたルールに対してチェックされることを保証します。これは、防御の第一線として機能します。

このアプローチは、バリデーションをデータ定義に近づけ、より宣言的にし、カスタムリゾルバコードで無視されたり一貫性のない実装になったりする可能性を低減します。これにより、リクエストライフサイクルの可能な限り早い段階でデータ品質を強制することで、APIの信頼性とセキュリティが大幅に向上します。

次の章では、スキーマ制御と高度なディレクティブについて詳しく見ていきます。
