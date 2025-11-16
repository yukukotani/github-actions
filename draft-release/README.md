# Draft Release Action

リリース用のPRを自動作成するGitHub Actionです。package.jsonのバージョンをバンプし、リリースノートを自動生成してドラフトPRを作成します。

このアクションは2つの方法で使用できます：
- **Composite Action** として（推奨）
- **Reusable Workflow** として

## 機能

- 📦 package.jsonのバージョン自動バンプ（patch/minor/major）
- 📝 GitHubのリリースノート自動生成機能を使用
- 🏷️ 前回のリリースタグから変更履歴を自動抽出
- 📬 ドラフトPRの自動作成
- 🎯 カスタマイズ可能なラベルとアサイン

## 使い方

### 方法1: Composite Action として使用（推奨）

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
        uses: <organization>/<repository>/draft-release@main
        with:
          version: ${{ github.event.inputs.version }}
```

### 方法2: Reusable Workflow として使用

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
    uses: <organization>/<repository>/draft-release/draft-release.yml@main
    with:
      version: ${{ github.event.inputs.version }}
```

**注意**: Reusable Workflowの場合、権限は自動的に設定されます。

### すべてのオプションを使った例（Composite Action）

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
        uses: <organization>/<repository>/draft-release@main
        with:
          version: ${{ github.event.inputs.version }}
          node-version: '20.x'
          package-manager: 'npm'
          pr-labels: 'Type: Release,automated'
          draft-pr: 'true'
```

## 入力パラメータ

| パラメータ | 必須 | デフォルト | 説明 |
|-----------|------|-----------|------|
| `version` | ✅ | - | バージョンタイプ（patch/minor/major） |
| `node-version` | ❌ | `lts/*` | 使用するNode.jsのバージョン |
| `package-manager` | ❌ | `npm` | 使用するパッケージマネージャー（npm または none） |
| `pr-labels` | ❌ | `Type: Release` | PRに付与するラベル（カンマ区切り、Composite Action）または JSON配列形式（Reusable Workflow） |
| `draft-pr` | ❌ | `true` | PRをドラフトとして作成するか |
| `github-token` | ❌ | `${{ github.token }}` | GitHub Token（Composite Actionのみ） |

## 出力

| 出力 | 説明 |
|-----|------|
| `version` | バンプ後の新しいバージョン番号 |
| `pr-number` | 作成されたPRの番号 |
| `pr-url` | 作成されたPRのURL（Composite Actionのみ） |

### 出力の使用例（Composite Action）

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
        uses: <organization>/<repository>/draft-release@main
        with:
          version: ${{ github.event.inputs.version }}
      
      - name: Show outputs
        run: |
          echo "New version: ${{ steps.draft-pr.outputs.version }}"
          echo "PR number: ${{ steps.draft-pr.outputs.pr-number }}"
          echo "PR URL: ${{ steps.draft-pr.outputs.pr-url }}"
```

### 出力の使用例（Reusable Workflow）

```yaml
jobs:
  draft-release:
    uses: <organization>/<repository>/draft-release/draft-release.yml@main
    with:
      version: ${{ github.event.inputs.version }}

  notify:
    needs: draft-release
    runs-on: ubuntu-latest
    steps:
      - name: Notify
        run: |
          echo "New version: ${{ needs.draft-release.outputs.version }}"
          echo "PR number: ${{ needs.draft-release.outputs.pr-number }}"
```

## 必要な権限

### Composite Actionの場合

呼び出し元のジョブに以下の権限を設定してください：

```yaml
permissions:
  contents: write        # バージョンファイルの変更とコミット
  pull-requests: write   # PRの作成
```

### Reusable Workflowの場合

権限は自動的に設定されるため、呼び出し元での設定は不要です。

## 前提条件

- リポジトリに `package.json` ファイルが存在すること
- `package-manager` に `npm` を指定する場合、Node.jsプロジェクトであること

## ワークフローの動作

1. リポジトリをチェックアウト
2. Gitの設定（github-actions[bot]として）
3. Node.jsのセットアップ（指定されたpackage-managerが `none` でない場合）
4. package.jsonのバージョンをバンプ
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

このワークフローはNode.js/npmプロジェクトを前提としています。package.jsonが存在しない場合は、`package-manager: 'none'` を指定して独自のバージョンバンプロジックを実装してください。

### 権限エラーが発生する

呼び出し元のワークフローに適切な `permissions` が設定されているか確認してください。

### PRが作成されない

- リポジトリの設定でGitHub Actionsにワークフロー権限が付与されているか確認
- `GITHUB_TOKEN` に適切な権限があるか確認

## 2つの方法の使い分け

| 特徴 | Composite Action | Reusable Workflow |
|-----|-----------------|-------------------|
| 記述の簡潔さ | ステップとして記述 | ジョブとして記述 |
| 権限設定 | 呼び出し元で設定が必要 | 自動的に設定される |
| チェックアウト | 明示的に必要 | 不要 |
| 柔軟性 | 他のステップと組み合わせやすい | ジョブ単位で独立 |
| 推奨用途 | 他のステップと組み合わせる場合 | 単独で完結する場合 |

## カスタマイズ例

### 異なるブランチをベースにする（Composite Action）

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
      
      - uses: <organization>/<repository>/draft-release@main
        with:
          version: ${{ github.event.inputs.version }}
```

### プライベートリポジトリでの使用

プライベートリポジトリでこのアクションを使用する場合、呼び出し元のリポジトリに適切なアクセス権限が必要です。

## ライセンス

このワークフローはMITライセンスの下で公開されています。

## 関連ワークフロー

- [publish-release](../publish-release/README.md) - リリースPRがマージされた際に自動的にnpmパッケージを公開し、GitHubリリースを作成するワークフロー
