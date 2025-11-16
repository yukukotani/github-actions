# GitHub Actions Collection

再利用可能なComposite Actionsのコレクションです。

## 📦 提供されるアクション

### 🚀 [draft-release](./draft-release/)

リリース用のPRを自動作成するアクションです。

**主な機能:**
- package.jsonのバージョン自動バンプ（patch/minor/major）
- GitHubのリリースノート自動生成
- ドラフトPRの自動作成

**使い方:**
```yaml
- uses: <organization>/<repository>/draft-release@main
  with:
    version: patch
```

[詳細なドキュメント](./draft-release/README.md)

### 📦 [publish-release](./publish-release/)

リリースPRがマージされた際に、npmパッケージの公開とGitHubリリースを作成するアクションです。

**主な機能:**
- npmへの自動パッケージ公開（Provenanceサポート）
- GitHubリリースとタグの自動作成
- ビルドとテストの実行
- 複数のパッケージマネージャーサポート（npm / Bun）

**使い方:**
```yaml
- uses: <organization>/<repository>/publish-release@main
  with:
    npm-token: ${{ secrets.NPM_TOKEN }}
```

[詳細なドキュメント](./publish-release/README.md)

## 🎯 使用方法

各アクションはComposite Actionとして提供されており、ステップとして直接使用できます。

```yaml
jobs:
  my-job:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v5
      
      - uses: <organization>/<repository>/draft-release@main
        with:
          version: patch
```

## 📁 ディレクトリ構造

```
.
├── README.md                      # このファイル
├── draft-release/
│   ├── action.yml                # Composite Action定義
│   └── README.md                 # 詳細なドキュメント
└── publish-release/
    ├── action.yml                # Composite Action定義
    └── README.md                 # 詳細なドキュメント
```

## 💡 実践例：フルリリースワークフロー

draft-release と publish-release を組み合わせたフルリリースワークフローの例：

### リリースPR作成ワークフロー

```yaml
# .github/workflows/release.yml
name: Release

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
  draft:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v5
      
      - name: Create Release PR
        uses: <organization>/<repository>/draft-release@main
        with:
          version: ${{ github.event.inputs.version }}
```

### リリース公開ワークフロー

```yaml
# .github/workflows/publish.yml
name: Publish

on:
  pull_request:
    branches:
      - main
    types:
      - closed

jobs:
  publish:
    if: |
      github.event.pull_request.merged == true &&
      contains(github.event.pull_request.labels.*.name, 'Type: Release')
    runs-on: ubuntu-latest
    permissions:
      contents: write
      id-token: write
    steps:
      - uses: actions/checkout@v5
      
      - name: Publish Release
        uses: <organization>/<repository>/publish-release@main
        with:
          npm-token: ${{ secrets.NPM_TOKEN }}
```

## 🔑 主な特徴

- **シンプル**: ステップとして使用でき、他のステップと簡単に組み合わせ可能
- **柔軟**: 豊富なカスタマイズオプション
- **安全**: npm Provenanceサポートでパッケージの来歴を証明
- **自動化**: バージョンバンプからリリースまでを完全自動化

## 📋 各アクションで必要な権限

### draft-release

```yaml
permissions:
  contents: write        # バージョンファイルの変更とコミット
  pull-requests: write   # PRの作成
```

### publish-release

```yaml
permissions:
  contents: write        # GitHubリリースとタグの作成
  id-token: write        # npm Provenance（来歴情報）
```

## 🤝 コントリビューション

このリポジトリへの貢献を歓迎します！

## 📄 ライセンス

MIT License

## 📚 関連リンク

- [GitHub Actions ドキュメント](https://docs.github.com/en/actions)
- [Composite Actions について](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action)
- [npm Provenance について](https://docs.npmjs.com/generating-provenance-statements)
