# Workflow Templates

## draft-release.yml

```yaml
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
      - uses: actions/checkout@v4

      - name: Create Release PR
        uses: yukukotani/github-actions/draft-release@main
        with:
          version: ${{ github.event.inputs.version }}
```

## publish-release.yml

Default template using bun. Adjust `package-manager`, `build-command`, `test-command`, and npm-related inputs based on repository inspection.

```yaml
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
      - uses: actions/checkout@v4

      - name: Publish Release
        uses: yukukotani/github-actions/publish-release@main
        with:
          npm-token: ${{ secrets.NPM_TOKEN }}
```

### Customization parameters for publish-release

When the repository uses npm instead of bun (default), add:

```yaml
        with:
          package-manager: 'npm'
          build-command: 'npm run build'
          test-command: 'npm test'
```

When build or test scripts are absent in package.json, set the corresponding command to empty string:

```yaml
          build-command: ''
          test-command: ''
```

When the default branch is not `main`, change the `branches` filter:

```yaml
    branches:
      - master
```

When npm publish is not needed (e.g. CLI-only tool distributed via GitHub releases):

```yaml
        with:
          skip-npm-publish: 'true'
```
