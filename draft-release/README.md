# Draft Release Action

リリース用のPRを自動作成するComposite Actionです。npmを使用してpackage.jsonのバージョンをバンプし、リリースノートを自動生成してドラフトPRを作成します。

## 機能

- 📦 package.jsonのバージョン自動バンプ（patch/minor/major）
- 📝 GitHubのリリースノート自動生成機能を使用
- 🏷️ 前回のリリースタグから変更履歴を自動抽出
- 📬 ドラフトPRの自動作成
- 🎯 カスタマイズ可能なラベルとアサイン
- 🔧 npm固定で動作

## 使い方

### 基本的な使い方

呼び出し元のリポジトリで、以下のようなワークフローファイルを作成してください：

```yaml
name: Draft Release

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version type'
        required: true
        type: choice
        options:
          - patch
          - minor
          - major

jobs:
  draft-release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - name: Checkout
        uses: actions/checkout@v5

      - name: Create Draft Release PR
        uses: yukukotani/github-actions/draft-release@main
        with:
          version: ${{ github.event.inputs.version }}
```

### すべてのオプションを使った例

```yaml
name: Draft Release

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version type'
        required: true
        type: choice
        options:
          - patch
          - minor
          - major

jobs:
  draft-release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - name: Checkout
        uses: actions/checkout@v5

      - name: Create Draft Release PR
        uses: yukukotani/github-actions/draft-release@main
        with:
          version: ${{ github.event.inputs.version }}
          node-version: '20.x'
          pr-labels: 'Type: Release,automated'
          draft-pr: 'true'
```

## 入力パラメータ

| パラメータ | 必須 | デフォルト | 説明 |
|-----------|------|-----------|------|
| `version` | ✅ | - | バージョンタイプ（patch/minor/major） |
| `node-version` | ❌ | `lts/*` | 使用するNode.jsのバージョン |
| `pr-labels` | ❌ | `Type: Release` | PRに付与するラベル（カンマ区切り） |
| `draft-pr` | ❌ | `true` | PRをドラフトとして作成するか |
| `github-token` | ❌ | `${{ github.token }}` | GitHub Token |

## 出力

| 出力 | 説明 |
|-----|------|
| `version` | バンプ後の新しいバージョン番号 |
| `pr-number` | 作成されたPRの番号 |
| `pr-url` | 作成されたPRのURL |

### 出力の使用例

```yaml
jobs:
  draft-release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v5
      
      - name: Create Draft Release PR
        id: draft-pr
        uses: yukukotani/github-actions/draft-release@main
        with:
          version: ${{ github.event.inputs.version }}
      
      - name: Show outputs
        run: |
          echo "New version: ${{ steps.draft-pr.outputs.version }}"
          echo "PR number: ${{ steps.draft-pr.outputs.pr-number }}"
          echo "PR URL: ${{ steps.draft-pr.outputs.pr-url }}"
```

## 必要な権限

呼び出し元のジョブに以下の権限を設定してください：

```yaml
permissions:
  contents: write        # バージョンファイルの変更とコミット
  pull-requests: write   # PRの作成
```

## 前提条件

- リポジトリに `package.json` ファイルが存在すること
- Node.jsプロジェクトであること（npmを使用）

## ワークフローの動作

1. リポジトリをチェックアウト
2. Gitの設定（github-actions[bot]として）
3. Node.jsのセットアップ
4. npmを使用してpackage.jsonのバージョンをバンプ
5. GitHub APIを使用してリリースノートを自動生成
   - 前回のリリースタグを自動検出
   - 前回のリリースから現在までの変更履歴を抽出
   - 初回リリースの場合は全コミット履歴から生成
6. リリース用のブランチとドラフトPRを作成
   - ブランチ名: `release/v{version}`
   - コミットメッセージ: `chore: release v{version}`
   - PRにラベルを自動付与
   - PRをトリガーしたユーザーにアサイン

## リリースノートの自動生成について

このワークフローは[GitHub Releases API](https://docs.github.com/en/rest/releases/releases#generate-release-notes-content-for-a-release)を使用してリリースノートを自動生成します。

- リポジトリの `.github/release.yml` 設定に基づいてノートを生成
- 前回のリリースタグが存在する場合、そこからの差分を抽出
- 初回リリースの場合、全コミット履歴から生成
- PRのマージコミットを自動的にグループ化

## トラブルシューティング

### package.jsonが見つからない

このアクションはNode.js/npmプロジェクトを前提としています。package.jsonが存在することを確認してください。

### 権限エラーが発生する

呼び出し元のワークフローに適切な `permissions` が設定されているか確認してください。

### PRが作成されない

- リポジトリの設定でGitHub Actionsにワークフロー権限が付与されているか確認
- `GITHUB_TOKEN` に適切な権限があるか確認

## カスタマイズ例

### 異なるブランチをベースにする

```yaml
jobs:
  draft-release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v5
        with:
          ref: develop
      
      - uses: yukukotani/github-actions/draft-release@main
        with:
          version: ${{ github.event.inputs.version }}
```

### プライベートリポジトリでの使用

プライベートリポジトリでこのアクションを使用する場合、呼び出し元のリポジトリに適切なアクセス権限が必要です。

## ライセンス

このアクションはMITライセンスの下で公開されています。

## 関連アクション

- [publish-release](../publish-release/README.md) - リリースPRがマージされた際に自動的にnpmパッケージを公開し、GitHubリリースを作成するアクション
