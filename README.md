# Lighthouse PHP 6 ディレクティブ解説

---

## はじめに

本ドキュメントでは、Lighthouse PHP 6で利用可能なすべてのディレクティブを包括的に解説します。
Lighthouseは、LaravelアプリケーションにおけるGraphQL APIの構築を強力にサポートするツールであり、
ディレクティブを通じてその機能の大部分を提供しています。

## 目次

1. [Lighthouseディレクティブの概要](./docs/01_overview.md)
2. [Eloquentとデータベース操作ディレクティブ](./docs/02_eloquent_database.md)
3. [認証と認可ディレクティブ](./docs/03_auth_permission.md)
4. [パフォーマンスとキャッシュディレクティブ](./docs/04_performance_cache.md)
5. [データ変換と入力操作ディレクティブ](./docs/05_data_transform.md)
6. [バリデーションディレクティブ](./docs/06_validation.md)
7. [スキーマ制御と高度なディレクティブ](./docs/07_schema_advanced.md)
8. [総括](./docs/08_conclusion.md)

## このドキュメントについて

このドキュメントは、Lighthouse PHP 6の公式APIリファレンスに基づいて作成されています。
各ディレクティブの機能と使用方法を詳細に解説し、実際の開発における活用方法を提示します。

Laravel開発者がGraphQLの利点を最大限に活かしながら、効率的にAPIを構築するための手引きとなることを目指しています。

---

## Docsifyドキュメントの閲覧方法

この日本語解説は、GitHubリポジトリでも公開しています。
Docsifyで見やすいWebドキュメントとして閲覧するには以下の方法があります。

### ローカルでの起動方法

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

### Replitでのホスティング方法

1. [Replit](https://replit.com)でアカウントを作成またはログイン
2. 「+ Create Repl」ボタンをクリック
3. 「Static Site」テンプレートを選択
4. プロジェクト名を入力して作成
5. `public`ディレクトリに、このリポジトリの`docs`ディレクトリの内容をアップロード
6. 「Run」ボタンをクリックしてデプロイ
7. 生成されたURLでアクセス可能になります

### その他のホスティング方法

以下のサービスでも簡単にホスティングできます：

- **GitHub Pages**: リポジトリ設定でGitHub Pagesを有効化し、Source設定で「main」ブランチと「/docs」フォルダを選択
- **Netlify/Vercel**: GitHubリポジトリと連携し、ビルド設定でパブリッシュディレクトリを「docs」に設定
- **Firebase Hosting**: Firebase CLIを使い、publicディレクトリを「docs」に設定してデプロイ

### リポジトリ
- [kaminuma/lighthouse-directives](https://github.com/kaminuma/lighthouse-directives)

© 2025 Lighthouse PHP ディレクティブ解説
