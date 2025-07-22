# Lighthouse PHP ディレクティブ解説集

> Laravel GraphQL APIのための包括的なディレクティブガイド

## はじめに

本ドキュメントでは、Lighthouse PHP 6で利用可能なすべてのディレクティブを包括的に解説します。
Lighthouseは、LaravelアプリケーションにおけるGraphQL APIの構築を強力にサポートするツールであり、
ディレクティブを通じてその機能の大部分を提供しています。

## ディレクティブカテゴリ

Lighthouse PHPのディレクティブは、以下のカテゴリに分類されます：

| カテゴリ | 説明 | 主要ディレクティブ |
|---------|------|-----------------|
| **Eloquentとデータベース操作** | EloquentモデルとSQLクエリの操作 | `@all`, `@find`, `@create`, `@update` |
| **認証と権限** | ユーザー認証と権限管理 | `@auth`, `@can`, `@guard` |
| **パフォーマンスとキャッシュ** | 応答速度とスケーラビリティの向上 | `@cache`, `@async`, `@throttle` |
| **データ変換** | 入力データの操作と変換 | `@hash`, `@trim`, `@upload` |
| **バリデーション** | 入力データの検証 | `@rules`, `@rulesForArray` |
| **スキーマ制御** | 高度なスキーマ操作と管理 | `@deprecated`, `@event`, `@subscription` |

## 主な機能

- **80+ 専用ディレクティブ** - さまざまなユースケースに対応
- **Eloquent完全統合** - LaravelのORMと緊密に連携
- **高度な認可システム** - Laravelポリシーとの統合
- **組み込みキャッシュ** - パフォーマンスの最適化
- **スキーマレベルのバリデーション** - 効率的なデータ検証
- **リアルタイム機能** - サブスクリプション対応

## このドキュメントの使い方

左側のサイドバーから各カテゴリにアクセスし、目的のディレクティブを見つけることができます。また、上部の検索バーを使用して、特定のディレクティブやキーワードで検索することも可能です。

各ディレクティブのページには、以下の情報が含まれています：
- ディレクティブの目的と機能
- 基本的な使用例
- オプションと引数の説明
- 高度な使用例とテクニック

## 始める前に

このドキュメントを最大限に活用するためには、LaravelとGraphQLの基本的な知識があることが望ましいです。

## サンプルコード

```graphql
type Query {
  users: [User!]! @paginate
  user(id: ID! @eq): User @find
  posts(category: String @where(operator: "=")): [Post!]! @all
}

type Mutation {
  createUser(
    name: String! @rules(apply: ["required", "max:255"])
    email: String! @rules(apply: ["required", "email", "unique:users,email"])
  ): User! @create
}
```

## ローカルでの閲覧

このドキュメントは、Docsifyを使用して構築されています。ローカルで閲覧する場合は、以下の手順に従ってください：

```bash
# リポジトリをクローン
git clone https://github.com/kaminuma/lighthouse-directives.git

# ディレクトリに移動
cd lighthouse-directives

# Docsifyをインストール（初回のみ）
npm install -g docsify-cli

# サーバーを起動
docsify serve docs
```

ブラウザで [http://localhost:3000](http://localhost:3000) にアクセスすると、このドキュメントが表示されます。
