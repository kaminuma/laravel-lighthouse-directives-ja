# スキーマカスタマイズ（Schema Customization）

---


### @scalar

**概要**  
カスタムスカラー型を定義し、独自のデータ型をGraphQLスキーマに追加するためのディレクティブです。

**主な用途**
- 日付、JSON、カスタム文字列フォーマットなどの型追加

**使用例**
```graphql
scalar DateTime @scalar(class: "App\\GraphQL\\Scalars\\DateTime")
scalar JSON @scalar(class: "App\\GraphQL\\Scalars\\JSON")
```

**注意点**
- スカラークラスはGraphQL型仕様に準拠する必要があります

---

### @enum

**概要**  
列挙型の内部値と外部値をマッピングするディレクティブです。

**主な用途**
- DB値とAPI表現の変換

**使用例**
```graphql
enum UserRole @enum {
    ADMIN @enum(value: 1)
    EDITOR @enum(value: 2)
    USER @enum(value: 3)
}
```

**注意点**
- 一度定義した値の変更は破壊的変更となる可能性があります

---

### @union

**概要**  
複数の型のいずれかを返すことができるユニオン型の解決方法を定義するディレクティブです。

**主な用途**
- 柔軟な多態的データ構造

**使用例**
```graphql
union SearchResult @union(resolveType: "App\\GraphQL\\Unions\\SearchResultResolver") = 
    User | Post | Comment
```

**注意点**
- resolveTypeクラスで正確な型判別ロジックの実装が必要

---

### @interface

**概要**  
インターフェース型の具象実装を解決するための方法を定義するディレクティブです。

**主な用途**
- 共通フィールドを持つ複数型の抽象化

**使用例**
```graphql
interface Node @interface(resolveType: "App\\GraphQL\\Interfaces\\NodeResolver") {
    id: ID!
}
```

**注意点**
- resolveTypeクラスで正確な型解決ロジックを実装する必要があります

---

### @feature

**概要**  
Laravel Pennantのフィーチャーフラグに基づいて、スキーマ要素の表示/非表示を制御するディレクティブです。

**主な用途**
- 段階的な機能のロールアウトやA/Bテスト

**使用例**
```graphql
type Query {
    search(term: String!): [Result!]! 
        @feature(name: "new-search-api")
}
```

**注意点**
- フィーチャーフラグの状態管理を適切に行う必要があります

---

### @deprecated

**概要**  
スキーマ要素を非推奨としてマークし、将来的な削除を予告するディレクティブです。

**主な用途**
- APIの進化を管理し、クライアントに移行期間を提供

**使用例**
```graphql
type User {
    username: String! @deprecated(reason: "代わりにnameフィールドを使用してください")
}
```

**注意点**
- 必ずdeprecation reasonを提供し、代替手段を明示してください

---

（既存の注意点・ベストプラクティス等は維持）
