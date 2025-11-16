# 再利用可能な GitHub Actions ワークフロー

このディレクトリには、他のリポジトリから再利用可能な GitHub Actions ワークフローが含まれています。

## 利用可能なワークフロー

### 1. Reusable Draft Release

**ファイル**: `reusable-draft-release.yml`

Node.js プロジェクトのリリース PR を自動的に作成する再利用可能なワークフローです。

#### 機能

- `package.json` のバージョンを自動的にバンプ（patch/minor/major）
- 前回のリリースからの変更履歴を自動生成
- ドラフト PR を自動作成
- リリースノートを PR の本文に含める

#### 使用方法

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
    uses: your-org/your-repo/.github/workflows/reusable-draft-release.yml@main
    with:
      version: ${{ github.event.inputs.version }}
      # オプション: Node.js のバージョンを指定（デフォルト: 'lts/*'）
      node-version: 'lts/*'
      # オプション: PR のアサインを指定（デフォルト: ワークフロー実行者）
      assignees: 'username1,username2'
    secrets:
      # オプション: カスタムトークンを使用する場合
      github-token: ${{ secrets.GITHUB_TOKEN }}
    permissions:
      contents: write
      pull-requests: write
```

#### 必須権限

このワークフローを使用するには、以下の権限が必要です：

- `contents: write` - バージョンファイルの変更とコミット用
- `pull-requests: write` - PR の作成用

#### Inputs

| 名前 | 必須 | 型 | デフォルト | 説明 |
|------|------|-----|-----------|------|
| `version` | ✅ | string | - | バージョンタイプ（`patch`, `minor`, `major` のいずれか） |
| `node-version` | ❌ | string | `lts/*` | 使用する Node.js のバージョン |
| `assignees` | ❌ | string | (実行者) | PR にアサインするユーザー（カンマ区切り） |

#### Secrets

| 名前 | 必須 | 説明 |
|------|------|------|
| `github-token` | ❌ | GitHub トークン（指定しない場合は `github.token` を使用） |

#### 前提条件

- リポジトリに `package.json` ファイルが存在すること
- Node.js プロジェクトであること
- セマンティックバージョニングを使用していること

#### 動作フロー

1. リポジトリをチェックアウト
2. Git ユーザー設定（github-actions[bot]）
3. Node.js のセットアップ
4. `package.json` のバージョンをバンプ
5. 前回のリリースタグを取得
6. GitHub API でリリースノートを自動生成
7. ドラフト PR を作成（リリースノート付き）

#### 出力される PR

- **ブランチ名**: `release/vX.Y.Z`
- **タイトル**: `Release vX.Y.Z`
- **本文**: 自動生成されたリリースノート
- **コミットメッセージ**: `chore: release vX.Y.Z`
- **ラベル**: `Type: Release`
- **状態**: ドラフト
- **アサイン**: 指定されたユーザーまたはワークフロー実行者

#### 使用例

##### 基本的な使用

```yaml
name: Create Release Draft

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
  create-draft:
    uses: your-org/workflows/.github/workflows/reusable-draft-release.yml@main
    with:
      version: ${{ github.event.inputs.version }}
    permissions:
      contents: write
      pull-requests: write
```

##### カスタム設定を使用

```yaml
name: Create Release Draft

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
  create-draft:
    uses: your-org/workflows/.github/workflows/reusable-draft-release.yml@main
    with:
      version: ${{ github.event.inputs.version }}
      node-version: '20.x'
      assignees: 'maintainer1,maintainer2'
    secrets:
      github-token: ${{ secrets.CUSTOM_GITHUB_TOKEN }}
    permissions:
      contents: write
      pull-requests: write
```

#### トラブルシューティング

##### 権限エラーが発生する場合

ワークフローを呼び出す側のジョブに適切な `permissions` が設定されていることを確認してください：

```yaml
permissions:
  contents: write
  pull-requests: write
```

##### PR が作成されない場合

- `github-token` に必要な権限があることを確認
- リポジトリ設定で GitHub Actions が PR を作成できることを確認

---

### 2. Reusable Publish Release

**ファイル**: `reusable-publish-release.yml`

リリース PR がマージされた際に、npm パッケージを公開し、GitHub Release を作成する再利用可能なワークフローです。

#### 機能

- npm パッケージの公開（provenance 付き）
- GitHub Release の自動作成
- ビルドとテストの実行
- PR へのコメント通知（成功/失敗）
- タグの重複チェック
- Bun または npm のサポート

#### 使用方法

```yaml
name: Publish Release

on:
  pull_request:
    branches:
      - main
    types:
      - closed

jobs:
  release:
    if: |
      github.event.pull_request.merged == true &&
      contains(github.event.pull_request.labels.*.name, 'Type: Release')
    uses: your-org/your-repo/.github/workflows/reusable-publish-release.yml@main
    with:
      # オプション: Node.js のバージョンを指定（デフォルト: 'lts/*'）
      node-version: 'lts/*'
      # オプション: npm レジストリ URL（デフォルト: 'https://registry.npmjs.org'）
      registry-url: 'https://registry.npmjs.org'
      # オプション: Bun を使用するか（デフォルト: true）
      use-bun: true
      # オプション: ビルドコマンド（デフォルト: 'bun run build'）
      build-command: 'bun run build'
      # オプション: テストコマンド（デフォルト: 'bun test'）
      test-command: 'bun test'
      # オプション: テストをスキップするか（デフォルト: false）
      skip-tests: false
    secrets:
      # 必須: npm 認証トークン
      npm-token: ${{ secrets.NPM_TOKEN }}
      # オプション: カスタム GitHub トークン
      github-token: ${{ secrets.GITHUB_TOKEN }}
    permissions:
      contents: write
      id-token: write
      pull-requests: write
```

#### 必須権限

このワークフローを使用するには、以下の権限が必要です：

- `contents: write` - GitHub Release の作成とタグ付け用
- `id-token: write` - npm provenance（OIDC）用
- `pull-requests: write` - PR へのコメント用

#### Inputs

| 名前 | 必須 | 型 | デフォルト | 説明 |
|------|------|-----|-----------|------|
| `node-version` | ❌ | string | `lts/*` | 使用する Node.js のバージョン |
| `registry-url` | ❌ | string | `https://registry.npmjs.org` | npm レジストリの URL |
| `use-bun` | ❌ | boolean | `true` | Bun を使用するか（false の場合は npm を使用） |
| `build-command` | ❌ | string | `bun run build` | ビルドコマンド |
| `test-command` | ❌ | string | `bun test` | テストコマンド |
| `skip-tests` | ❌ | boolean | `false` | テストをスキップするか |

#### Secrets

| 名前 | 必須 | 説明 |
|------|------|------|
| `npm-token` | ✅ | npm パッケージを公開するための認証トークン |
| `github-token` | ❌ | GitHub トークン（指定しない場合は `github.token` を使用） |

#### Outputs

| 名前 | 説明 |
|------|------|
| `version` | 公開されたバージョン |
| `release-url` | GitHub Release の URL |

#### 前提条件

- リポジトリに `package.json` ファイルが存在すること
- npm パッケージとして公開可能なプロジェクトであること
- npm トークンが設定されていること
- リリース PR に `Type: Release` ラベルが付いていること

#### 動作フロー

1. リポジトリをチェックアウト
2. `package.json` からバージョンとパッケージ名を取得
3. タグが既に存在するかチェック（存在する場合はスキップ）
4. Node.js のセットアップ
5. Bun または npm のセットアップ
6. 依存関係のインストール
7. ビルドの実行
8. テストの実行（スキップ可能）
9. npm パッケージの公開（provenance 付き）
10. GitHub Release の作成
11. PR へのコメント（成功/失敗）

#### 使用例

##### 基本的な使用（Bun を使用）

```yaml
name: Publish Release

on:
  pull_request:
    branches:
      - main
    types:
      - closed

jobs:
  release:
    if: |
      github.event.pull_request.merged == true &&
      contains(github.event.pull_request.labels.*.name, 'Type: Release')
    uses: your-org/workflows/.github/workflows/reusable-publish-release.yml@main
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}
    permissions:
      contents: write
      id-token: write
      pull-requests: write
```

##### npm を使用する場合

```yaml
name: Publish Release

on:
  pull_request:
    branches:
      - main
    types:
      - closed

jobs:
  release:
    if: |
      github.event.pull_request.merged == true &&
      contains(github.event.pull_request.labels.*.name, 'Type: Release')
    uses: your-org/workflows/.github/workflows/reusable-publish-release.yml@main
    with:
      use-bun: false
      build-command: 'npm run build'
      test-command: 'npm test'
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}
    permissions:
      contents: write
      id-token: write
      pull-requests: write
```

##### テストをスキップする場合

```yaml
name: Publish Release

on:
  pull_request:
    branches:
      - main
    types:
      - closed

jobs:
  release:
    if: |
      github.event.pull_request.merged == true &&
      contains(github.event.pull_request.labels.*.name, 'Type: Release')
    uses: your-org/workflows/.github/workflows/reusable-publish-release.yml@main
    with:
      skip-tests: true
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}
    permissions:
      contents: write
      id-token: write
      pull-requests: write
```

##### 出力を使用する場合

```yaml
name: Publish Release

on:
  pull_request:
    branches:
      - main
    types:
      - closed

jobs:
  release:
    if: |
      github.event.pull_request.merged == true &&
      contains(github.event.pull_request.labels.*.name, 'Type: Release')
    uses: your-org/workflows/.github/workflows/reusable-publish-release.yml@main
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}
    permissions:
      contents: write
      id-token: write
      pull-requests: write

  notify:
    needs: release
    runs-on: ubuntu-latest
    steps:
      - name: Notify
        run: |
          echo "Released version: ${{ needs.release.outputs.version }}"
          echo "Release URL: ${{ needs.release.outputs.release-url }}"
```

#### トラブルシューティング

##### npm 公開が失敗する場合

- `NPM_TOKEN` シークレットが正しく設定されているか確認
- npm トークンに公開権限があるか確認
- パッケージ名が既に使用されていないか確認

##### GitHub Release が作成されない場合

- `github-token` に `contents: write` 権限があることを確認
- タグが既に存在していないか確認

##### PR コメントが投稿されない場合

- `github-token` に `pull-requests: write` 権限があることを確認
- ワークフローが `pull_request` イベントからトリガーされていることを確認

##### ビルドまたはテストが失敗する場合

- `build-command` と `test-command` が正しいか確認
- 依存関係が正しくインストールされているか確認
- 必要に応じて `skip-tests: true` を設定

---

## 組み合わせて使用する

この2つのワークフローを組み合わせることで、完全な自動リリースフローを構築できます：

1. **Draft Release**: 手動でトリガーしてリリース PR を作成
2. **Publish Release**: リリース PR がマージされたときに自動的にパッケージを公開

```yaml
# .github/workflows/draft-release.yml
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
  draft:
    uses: your-org/workflows/.github/workflows/reusable-draft-release.yml@main
    with:
      version: ${{ github.event.inputs.version }}
    permissions:
      contents: write
      pull-requests: write
```

```yaml
# .github/workflows/publish-release.yml
name: Publish Release

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
    uses: your-org/workflows/.github/workflows/reusable-publish-release.yml@main
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}
    permissions:
      contents: write
      id-token: write
      pull-requests: write
```

## ライセンス

これらのワークフローは MIT ライセンスの下で公開されています。
