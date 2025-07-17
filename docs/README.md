# Lighthouse ディレクティブ解説集

このドキュメントは、Lighthouse（Laravel用GraphQLサーバー）の主要ディレクティブをカテゴリごとに日本語で解説したものです。

## 目次
- [01 認証・認可](01_auth_permission.md)
- [02 リレーション](02_relationships.md)
- [03 入力と検証](03_input_validation.md)
- [04 モデルバインディング・Eloquent](04_model_eloquent.md)
- [05 フィールド制御](05_field_control.md)
- [06 データ取得とリゾルバ](06_data_retrieval.md)
- [07 その他ユーティリティ](07_utilities.md)

---

各章のサイドバーやリンクから詳細なディレクティブ解説をご覧ください。

---

## ローカルでDocsifyドキュメントを閲覧したい方へ

この日本語解説は、GitHubリポジトリでも公開しています。
Docsifyで見やすいWebドキュメントとしてローカル起動も可能です。

### 起動方法

1. リポジトリをクローン
   ```sh
   git clone https://github.com/kaminuma/lighthouse-directives.git
   cd lighthouse-directives
   ```
2. Docsifyをインストール（初回のみ）
   ```sh
   npm install -g docsify-cli
   ```
   ※権限エラーが出る場合は `npx docsify-cli serve docs` でもOK
3. ドキュメントサーバーを起動
   ```sh
   docsify serve docs
   ```
   または
   ```sh
   npx docsify-cli serve docs
   ```
4. ブラウザで [http://localhost:3000](http://localhost:3000) を開く

### リポジトリ
- [kaminuma/lighthouse-directives](https://github.com/kaminuma/lighthouse-directives) 