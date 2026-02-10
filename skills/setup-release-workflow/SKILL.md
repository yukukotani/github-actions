---
name: setup-release-workflow
description: Set up draft-release and publish-release GitHub Actions workflow files for an npm package repository using yukukotani/github-actions. Use ONLY when the user explicitly requests to run this skill or explicitly asks to set up release workflows. Do NOT invoke automatically.
---

# Setup Release Workflow

Set up `draft-release` and `publish-release` GitHub Actions workflows in the current repository using `yukukotani/github-actions` composite actions.

## Prerequisites

Verify the following before proceeding:

- `package.json` exists at the repository root
- The repository is hosted on GitHub

## Workflow

### 1. Inspect the repository

Read `package.json` and check for:

- `scripts.build` and `scripts.test` (determine build/test commands)
- `engines.node` (determine Node.js version constraint)
- Lockfile: `bun.lock` -> bun, `package-lock.json` -> npm
- Whether the package is scoped (`@scope/name`) or unscoped

Also detect the default branch name (`main` or `master`) via `git`.

### 2. Ensure `repository` field in package.json

npm provenance verification requires `package.json` to have a `repository` field whose URL matches the GitHub repository. Inspect `package.json` and fix if needed:

- If `repository` is missing or empty, add it.
- If `repository.url` does not match `https://github.com/<owner>/<repo>`, update it.

The expected format:

```json
{
  "repository": {
    "type": "git",
    "url": "https://github.com/<owner>/<repo>"
  }
}
```

The shorthand form `"repository": "github:<owner>/<repo>"` also works.

### 3. Create workflow files

Generate two files under `.github/workflows/`. Use the templates in [references/workflow-templates.md](references/workflow-templates.md) as the base, adjusting parameters based on what was found in step 1:

- **`draft-release.yml`** -- No parameters need adjustment; use the template as-is.
- **`publish-release.yml`** -- Adjust the following based on repository inspection:
  - `package-manager`: `bun` (if `bun.lock` exists) or `npm`
  - `build-command`: value of `scripts.build` prefixed with the package manager run command, or `''` if absent
  - `test-command`: value of `scripts.test` prefixed with the package manager run command, or `''` if absent
  - `npm-token` line: include `npm-token: ${{ secrets.NPM_TOKEN }}` unless the user says npm publish is not needed
  - `skip-npm-publish`: set to `'true'` if the user says npm publish is not needed

### 4. Post-setup checklist

After creating the files, present the following checklist to the user. These are manual steps the user must complete themselves.

**GitHub repository settings** (`https://github.com/<owner>/<repo>/settings/actions`):

1. Under **Workflow permissions**, select **Read and write permissions**
2. Check **Allow GitHub Actions to create and approve pull requests**

**npm Trusted Publishing** (if publishing to npm):

1. Go to `https://www.npmjs.com/package/<package-name>/access` (or Package Settings -> Trusted Publisher)
2. Select **GitHub Actions** as the publisher
3. Fill in:
   - **Organization or user**: the GitHub owner (user or org)
   - **Repository**: the repository name
   - **Workflow filename**: `publish-release.yml` (filename only, with extension)
   - **Environment name**: leave blank unless using GitHub Environments
4. Click **Set up connection**
5. Recommended: under **Publishing access**, select **Require two-factor authentication and disallow tokens** for maximum security
6. Revoke any old `NPM_TOKEN` secrets from the repository once trusted publishing is confirmed working

**Notes on Trusted Publishing**:

- Requires npm CLI >= 11.5.1 / Node.js >= 22.14.0 (the workflow handles this via `npm install -g npm@latest`)
- Workflow filename configured on npmjs.com must exactly match the file created (case-sensitive, `.yml` extension)
- Only works with GitHub-hosted runners (not self-hosted)
- Provenance is auto-generated; no `--provenance` flag needed when using trusted publishing
- Private repositories will not get provenance attestations even with trusted publishing
