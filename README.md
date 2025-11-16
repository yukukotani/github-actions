# GitHub Actions Collection

再利用可能なGitHub Actionsのコレクションです。各アクションは、Composite ActionとReusable Workflowの両方の形式で提供されています。

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

各アクションは2つの方法で使用できます：

### 1. Composite Action として（推奨）

ステップとして直接使用します。他のステップと組み合わせやすく、柔軟性が高いです。

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

### 2. Reusable Workflow として

ジョブとして使用します。権限設定が自動的に行われ、設定が簡単です。

```yaml
jobs:
  my-job:
    uses: <organization>/<repository>/draft-release/draft-release.yml@main
    with:
      version: patch
```

## 📁 ディレクトリ構造

```
.
├── README.md                      # このファイル
├── draft-release/
│   ├── action.yml                # Composite Action定義
│   ├── draft-release.yml         # Reusable Workflow定義
│   └── README.md                 # 詳細なドキュメント
└── publish-release/
    ├── action.yml                # Composite Action定義
    ├── publish-release.yml       # Reusable Workflow定義
    └── README.md                 # 詳細なドキュメント
```

## 🔑 2つの方法の使い分け

| 特徴 | Composite Action | Reusable Workflow |
|-----|-----------------|-------------------|
| 記述方法 | ステップとして記述 | ジョブとして記述 |
| 権限設定 | 呼び出し元で設定が必要 | 自動的に設定される |
| チェックアウト | 明示的に必要 | 不要 |
| 柔軟性 | 他のステップと組み合わせやすい | ジョブ単位で独立 |
| 推奨用途 | 複雑なワークフロー | シンプルなワークフロー |

## 💡 実践例

### draft-release + publish-release のフルワークフロー

#### Composite Action を使用

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
      
      - uses: <organization>/<repository>/draft-release@main
        with:
          version: ${{ github.event.inputs.version }}

---

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
      
      - uses: <organization>/<repository>/publish-release@main
        with:
          npm-token: ${{ secrets.NPM_TOKEN }}
```

#### Reusable Workflow を使用

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
    uses: <organization>/<repository>/draft-release/draft-release.yml@main
    with:
      version: ${{ github.event.inputs.version }}

---

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
    uses: <organization>/<repository>/publish-release/publish-release.yml@main
    secrets:
      NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

## 🤝 コントリビューション

このリポジトリへの貢献を歓迎します！

## 📄 ライセンス

MIT License

## 📚 関連リンク

- [GitHub Actions ドキュメント](https://docs.github.com/en/actions)
- [Composite Actions について](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action)
- [Reusable Workflows について](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
