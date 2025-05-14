# Pocket Sync Action

Pocketに保存されたアイテムを定期的に取得し、マークダウン形式でリポジトリに保存してPRを作成します。

## 機能

- Pocket APIを使用して保存済みアイテムを取得
- **リポジトリ内の最新記事以降の記事のみを取得**（重複を防止）
- **ページネーション機能により最新までのすべての記事を取得可能**（制限なし）
- **Pocket Access Tokenの自動取得機能**（OAuth認証フローを内蔵）
- 取得した記事データをマークダウン形式でリポジトリに保存
- 保存した記事データを含むPRを自動的に作成
- 様々なフィルタリングオプション（タグ、お気に入り、状態など）をサポート
- 定期実行のためのワークフロー設定例を提供

## 使用方法

### 引数

- `consumer_key`(required): PocketアプリのConsumer Key
- `access_token`(optional): ユーザーのPocket Access Token（省略時は認証フローを開始します）
- `redirect_uri`(optional): Pocket認証後のリダイレクトURI（認証フロー使用時のみ必要）
  - default: `https://github.com`
- `github_token`(required): GitHub APIにアクセスするためのトークン。通常は github.token を使用します。
- `output_dir`(optional): マークダウンファイルを保存するディレクトリ
  - default: `articles`
- `state`(optional): 取得するアイテムの状態（"unread", "archive", "all"）
  - default: `all`
- `favorite`(optional): お気に入りフィルター（"0" = 非お気に入りのみ, "1" = お気に入りのみ, "" = すべて）
  - default: `""`（すべて）
- `tag`(optional): 特定のタグでフィルタリング
  - default: `""`（すべて）
- `count`(optional): 1回のリクエストで取得するアイテムの最大数（Pocket APIの制限に近い値）
  - default: `100`
- `max_items`(optional): 取得する総アイテム数の上限（0=制限なし、すべての記事を取得）
  - default: `0`
- `sort`(optional): ソート順（"newest", "oldest", "title", "site"）
  - default: `newest`
- `pr_title`(optional): 作成するPRのタイトル
  - default: `Pocketから記事データを追加`
- `pr_branch_prefix`(optional): PRのブランチ名のプレフィックス
  - default: `pocket-articles`

### 設定例（毎日の記事取得 - 既存のアクセストークンを使用）

```yaml
name: Daily Pocket Articles Retrieval
on:
  schedule:
    - cron: '0 9 * * *'  # 毎日午前9時に実行

permissions:
  contents: write
  pull-requests: write

jobs:
  retrieve-pocket-articles:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: tsukulink/actions/pocket-retrieve@pocket-retrieve/v1
        with:
          consumer_key: ${{ secrets.POCKET_CONSUMER_KEY }}
          access_token: ${{ secrets.POCKET_ACCESS_TOKEN }}
          github_token: ${{ github.token }}
          output_dir: 'articles'
          state: 'unread'
          count: '100'        # 1回のリクエストで取得する記事数
          max_items: '0'      # 0=すべての記事を取得（制限なし）
          sort: 'newest'
          pr_title: '毎日のPocket記事を追加'
```
### 設定例（自動認証フローを使用）
```yaml
name: Daily Pocket Articles Retrieval with Auto Auth
on:
  workflow_dispatch:  # 手動実行（初回認証用）

permissions:
  contents: write
  pull-requests: write

jobs:
  retrieve-pocket-articles:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: tsukulink/actions/pocket-retrieve@pocket-retrieve/v1
        with:
          consumer_key: ${{ secrets.POCKET_CONSUMER_KEY }}
          # access_tokenを省略すると自動認証フローが開始されます
          github_token: ${{ github.token }}
          redirect_uri: 'https://github.com/your-org/your-repo'
          output_dir: 'articles'
          state: 'unread'
          count: '100'
          max_items: '0'
          sort: 'newest'
          pr_title: '毎日のPocket記事を追加'
```
### 設定例（お気に入り記事の週次取得）
```yaml
name: Weekly Favorite Pocket Articles
on:
  schedule:
    - cron: '0 0 * * 0'  # 毎週日曜日の午前0時に実行

permissions:
  contents: write
  pull-requests: write

jobs:
  export-favorite-articles:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: tsukulink/actions/pocket-retrieve@pocket-retrieve/v1
        with:
          consumer_key: ${{ secrets.POCKET_CONSUMER_KEY }}
          access_token: ${{ secrets.POCKET_ACCESS_TOKEN }}
          github_token: ${{ github.token }}
          output_dir: 'favorite-articles'
          state: 'all'
          favorite: '1'
          count: '100'        # 1回のリクエストで取得する記事数
          max_items: '0'      # 0=すべての記事を取得（制限なし）
          pr_title: 'お気に入りのPocket記事を追加'
          pr_branch_prefix: 'pocket-favorites'
```
## マークダウンファイルの形式
取得した記事は以下の形式でマークダウンファイルとして保存されます：
```markdown
# 記事タイトル

- URL: https://example.com/article
- 追加日時: 2025-05-14
- タグ: tech, programming

## 概要

記事の概要テキスト...
```

ファイル名は `YYYY-MM-DD-{記事ID}-{タイトル}.md` の形式で生成されます。

## Pocket APIの認証情報取得方法

### 方法1: アクションによる自動認証（推奨）

1. [Pocket Developer Portal](https://getpocket.com/developer/apps/new)でアプリを登録
2. Consumer Keyを取得
3. ワークフローで`access_token`パラメータを省略すると、アクションが自動的に認証フローを開始します
4. ワークフロー実行時に認証URLが表示されるので、そのURLにアクセスしてアプリを認証します
5. 認証後、アクションは自動的にアクセストークンを取得し、それを使用してPocket APIからデータを取得します
6. 取得したアクセストークンは、今後の実行のためにGitHub Secretsに保存することを推奨します

### 方法2: 手動でのAccess Token取得

1. [Pocket Developer Portal](https://getpocket.com/developer/apps/new)でアプリを登録
2. Consumer Keyを取得
3. [Pocket Authentication API](https://getpocket.com/developer/docs/authentication)の手順に従って手動でAccess Tokenを取得
4. 取得したAccess TokenをGitHub Secretsに保存

## 注意事項

- Pocket APIの利用制限に注意してください
- 認証情報は必ずGitHub Secretsとして安全に管理してください
- このアクションを使用するには、ワークフローに `contents` と `pull-requests` の書き込み権限が必要です
- 自動認証フローを使用する場合：
  - 初回実行時は手動でワークフローを実行し、表示される認証URLにアクセスしてアプリを認証する必要があります
  - 認証後に取得したアクセストークンは、今後の実行のためにGitHub Secretsに保存することを強く推奨します
  - 認証フローはGitHub Actionsの制約内で実装されているため、認証完了を待機する時間は30秒に設定されています
- このアクションは自動的にリポジトリ内の最新記事の日時を検出し、その日時以降に追加された記事のみを取得します
  - 初回実行時や記事が存在しない場合は、すべての記事を取得します
  - ページネーション機能により、最新までのすべての記事を取得できます（制限なし）
  - 既存の記事と同じ内容の場合は変更がないと判断され、PRは作成されません
