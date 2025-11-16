# Publish Release Workflow

リリースPRがマージされた際に、npmパッケージの公開とGitHubリリースの作成を自動実行する再利用可能なGitHub Actionsワークフローです。

## 機能

- 📦 npmへの自動パッケージ公開（Provenanceサポート）
- 🏷️ GitHubリリースとタグの自動作成
- 💬 PRへの結果コメント自動投稿
- ✅ タグの重複チェック
- 🔄 複数のパッケージマネージャーサポート（npm / Bun）
- 🧪 ビルドとテストの実行
- ⚙️ 柔軟なカスタマイズオプション

## 使い方

### 基本的な使い方

呼び出し元のリポジトリで、以下のようなワークフローファイルを作成してください：

```yaml
name: Publish Release

on:
  pull_request:
    branches:
      - main
    types:
      - closed

jobs:
  publish:
    uses: <organization>/<repository>/.github/workflows/publish-release.yml@main
    secrets:
      NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### すべてのオプションを使った例

```yaml
name: Publish Release

on:
  pull_request:
    branches:
      - main
    types:
      - closed

jobs:
  publish:
    uses: <organization>/<repository>/.github/workflows/publish-release.yml@main
    with:
      node-version: '20.x'
      package-manager: 'bun'
      build-command: 'bun run build'
      test-command: 'bun test'
      npm-access: 'public'
      release-label: 'Type: Release'
      skip-npm-publish: false
      skip-github-release: false
      comment-on-pr: true
    secrets:
      NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

## 入力パラメータ

| パラメータ | 必須 | デフォルト | 説明 |
|-----------|------|-----------|------|
| `node-version` | ❌ | `lts/*` | 使用するNode.jsのバージョン |
| `package-manager` | ❌ | `bun` | パッケージマネージャー（npm または bun） |
| `build-command` | ❌ | `bun run build` | ビルドコマンド。空文字列でスキップ |
| `test-command` | ❌ | `bun test` | テストコマンド。空文字列でスキップ |
| `npm-access` | ❌ | `public` | npmアクセスレベル（public または restricted） |
| `release-label` | ❌ | `Type: Release` | リリースをトリガーするPRラベル |
| `skip-npm-publish` | ❌ | `false` | npm公開をスキップするか |
| `skip-github-release` | ❌ | `false` | GitHubリリース作成をスキップするか |
| `comment-on-pr` | ❌ | `true` | PRに結果をコメントするか |

## 出力

| 出力 | 説明 |
|-----|------|
| `version` | 公開されたバージョン番号 |
| `release-url` | GitHubリリースのURL |
| `npm-url` | npmパッケージのURL |

### 出力の使用例

```yaml
jobs:
  publish:
    uses: <organization>/<repository>/.github/workflows/publish-release.yml@main
    secrets:
      NPM_TOKEN: ${{ secrets.NPM_TOKEN }}

  notify:
    needs: publish
    runs-on: ubuntu-latest
    steps:
      - name: Send notification
        run: |
          echo "Published version: ${{ needs.publish.outputs.version }}"
          echo "Release URL: ${{ needs.publish.outputs.release-url }}"
          echo "npm URL: ${{ needs.publish.outputs.npm-url }}"
```

## 必要な権限

このワークフローは以下の権限が必要です：

```yaml
permissions:
  contents: write        # GitHubリリースとタグの作成
  id-token: write        # npm Provenance（来歴情報）
  pull-requests: write   # PRへのコメント
```

## 前提条件

### 必須

- リポジトリに `package.json` ファイルが存在すること
- リリース用のPRに指定されたラベル（デフォルト: `Type: Release`）が付与されていること

### npmへの公開を行う場合

- npm tokenをシークレット `NPM_TOKEN` として登録
- package.jsonに公開に必要な情報が含まれていること

### Bunを使用する場合

- `bun.lock` ファイルが存在すること

## ワークフローの動作

1. **PRマージの確認**
   - PRがマージされていること
   - 指定されたラベルが付与されていることを確認

2. **パッケージ情報の取得**
   - package.jsonからバージョンとパッケージ名を取得

3. **タグの重複チェック**
   - 同じバージョンのタグが既に存在しないか確認
   - 存在する場合は以降の処理をスキップ

4. **環境セットアップ**
   - Node.jsのセットアップ
   - パッケージマネージャーのセットアップ
   - 依存関係のインストール

5. **ビルドとテスト**
   - ビルドコマンドを実行
   - テストコマンドを実行

6. **npm公開** （`skip-npm-publish: false` の場合）
   - Provenanceを含めてnpmに公開
   - `NPM_TOKEN` シークレットを使用

7. **GitHubリリース作成** （`skip-github-release: false` の場合）
   - バージョンタグを作成
   - PRの本文をリリースノートとして使用
   - GitHubリリースを作成

8. **PR通知** （`comment-on-pr: true` の場合）
   - 成功/失敗を示すコメントをPRに投稿
   - npmパッケージURLとGitHubリリースURLを含む

## トリガー条件

このワークフローは以下の条件をすべて満たす場合に実行されます：

1. PRがマージされた（`github.event.pull_request.merged == true`）
2. PRに指定されたラベル（デフォルト: `Type: Release`）が付与されている
3. ターゲットブランチがmainまたはmaster

## npm Provenanceについて

このワークフローは[npm Provenance](https://docs.npmjs.com/generating-provenance-statements)をサポートしています。Provenanceは、パッケージがどこでビルドされたかを証明する機能で、セキュリティとトラストを向上させます。

- 自動的に `--provenance` フラグ付きで公開
- `id-token: write` 権限が必要
- npm v9.5.0以降で利用可能

## 使用例

### Node.js + npmプロジェクト

```yaml
jobs:
  publish:
    uses: <organization>/<repository>/.github/workflows/publish-release.yml@main
    with:
      package-manager: 'npm'
      build-command: 'npm run build'
      test-command: 'npm test'
    secrets:
      NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### Bunプロジェクト（デフォルト）

```yaml
jobs:
  publish:
    uses: <organization>/<repository>/.github/workflows/publish-release.yml@main
    secrets:
      NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### GitHubリリースのみ（npm公開なし）

```yaml
jobs:
  publish:
    uses: <organization>/<repository>/.github/workflows/publish-release.yml@main
    with:
      skip-npm-publish: true
```

### npm公開のみ（GitHubリリースなし）

```yaml
jobs:
  publish:
    uses: <organization>/<repository>/.github/workflows/publish-release.yml@main
    with:
      skip-github-release: true
    secrets:
      NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### ビルド不要のプロジェクト

```yaml
jobs:
  publish:
    uses: <organization>/<repository>/.github/workflows/publish-release.yml@main
    with:
      build-command: ''
      test-command: ''
    secrets:
      NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

## トラブルシューティング

### npm公開に失敗する

- `NPM_TOKEN` が正しく設定されているか確認
- トークンに公開権限があるか確認
- package.jsonの `name` フィールドが正しいか確認
- スコープ付きパッケージの場合、`npm-access` が正しく設定されているか確認

### GitHubリリース作成に失敗する

- `contents: write` 権限が付与されているか確認
- ブランチ保護ルールと競合していないか確認

### タグが既に存在する

このワークフローは自動的にタグの重複をチェックし、既存のタグがある場合は処理をスキップします。これは正常な動作です。

### PRにコメントが投稿されない

- `pull-requests: write` 権限が付与されているか確認
- `comment-on-pr: true` が設定されているか確認

## セキュリティに関する注意事項

### NPM_TOKEN の管理

- リポジトリシークレットとして安全に保管
- 必要最小限の権限を付与
- 定期的なトークンのローテーション推奨

### Provenance の利用

- Provenanceを使用することでパッケージの信頼性が向上
- 改ざん検出が可能になる
- npm公開時に自動的に生成される

## よくある質問

### Q: 複数のブランチからリリースできますか？

A: はい。ワークフローのトリガーでブランチを追加してください：

```yaml
on:
  pull_request:
    branches:
      - main
      - develop
      - 'release/**'
    types:
      - closed
```

### Q: プライベートパッケージを公開できますか？

A: はい。`npm-access: 'restricted'` を設定してください：

```yaml
with:
  npm-access: 'restricted'
```

### Q: モノレポで使用できますか？

A: 現在のバージョンはシングルパッケージ用です。モノレポの場合は、各パッケージごとにワークフローをカスタマイズする必要があります。

### Q: 異なるラベル名を使いたい

A: `release-label` パラメータで変更できます：

```yaml
with:
  release-label: 'release'
```

## ライセンス

このワークフローはMITライセンスの下で公開されています。

## 関連ワークフロー

- [draft-release](../draft-release/README.md) - リリース用のPRを自動作成するワークフロー
