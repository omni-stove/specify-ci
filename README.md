# Specify CI Workflows

Specify (spec-kit) をCI環境で実行するための再利用可能なGitHub Actionsワークフロー集です。

## 概要

このリポジトリは、[spec-kit](https://github.com/github/spec-kit) を使用したSpec-Driven Development (SDD) のワークフローをCI/CDパイプラインに統合するための汎用的なソリューションを提供します。

## 機能

### 1. Tasks変更時の自動実装 (`tasks-changed-reusable.yml`)

tasks.mdが更新されたPRがマージされた際に、自動的に実装プロセスを開始します。

- **トリガー**: tasks.mdを含むPRがmainブランチにマージ
- **処理**:
  - 変更されたtasks.mdファイルを検出
  - spec IDを自動抽出
  - Claude Code Actionで実装を実行
  - 実装後、自動でコミット・プッシュ・PR作成

### 2. 手動実装トリガー (`manual-implement-reusable.yml`)

任意のタイミングでspec IDを指定して実装を開始できます。

- **トリガー**: workflow_dispatch（手動実行）
- **入力**: spec_id
- **処理**:
  - tasks.mdの存在を検証
  - Claude Code Actionで実装を実行
  - 実装後、自動でコミット・プッシュ・PR作成

## セットアップ

### 前提条件

1. **spec-kit**がプロジェクトに導入済み
   ```bash
   uvx --from git+https://github.com/github/spec-kit.git specify init --here
   ```

2. **Claude Code OAuth Token**の取得
   - Claude Code CLIで認証を実施
   - トークンをGitHub Secretsに登録

### 導入手順

#### 1. ワークフローファイルの作成

**方法A: 直接参照(推奨)**

このリポジトリのワークフローを直接参照することで、ファイルコピー不要で最新版を利用できます。

プロジェクトの `.github/workflows/` に以下のファイルを作成:

```yaml
# .github/workflows/tasks-changed.yml
name: Tasks Changed

on:
  pull_request:
    types: [closed]
    paths:
      - 'specs/**/tasks.md'
    branches:
      - main

permissions:
  contents: write
  pull-requests: write
  issues: write
  id-token: write

jobs:
  execute-tasks:
    if: github.event.pull_request.merged == true
    uses: omni-stove/specify-ci/.github/workflows/tasks-changed-reusable.yml@main
    with:
      specs_dir: 'specs'          # specsディレクトリのパス
      base_branch: 'main'         # ベースブランチ
    secrets:
      claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

```yaml
# .github/workflows/manual-implement.yml
name: Manual Implementation Trigger

on:
  workflow_dispatch:
    inputs:
      spec_id:
        description: 'Spec ID (e.g., 001-feature-name)'
        required: true
        type: string

permissions:
  contents: write
  pull-requests: write
  issues: write
  id-token: write

jobs:
  trigger-implementation:
    uses: omni-stove/specify-ci/.github/workflows/manual-implement-reusable.yml@main
    with:
      spec_id: ${{ inputs.spec_id }}
      specs_dir: 'specs'
      base_branch: 'main'
    secrets:
      claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

**方法B: ファイルをコピー**

高度なカスタマイズが必要な場合は、以下のファイルをプロジェクトにコピーします:

```bash
# 再利用可能なワークフロー
.github/workflows/tasks-changed-reusable.yml
.github/workflows/manual-implement-reusable.yml

# 実装例
.github/workflows/tasks-changed.yml
.github/workflows/manual-implement.yml
```

その後、ローカル参照に変更:

```yaml
# tasks-changed.yml の例
jobs:
  execute-tasks:
    if: github.event.pull_request.merged == true
    uses: ./.github/workflows/tasks-changed-reusable.yml  # ローカル参照
    with:
      specs_dir: 'specs'          # specsディレクトリのパス
      base_branch: 'main'         # ベースブランチ
    secrets:
      claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

#### 2. GitHub Secretsの設定

リポジトリの Settings > Secrets and variables > Actions で以下を追加:

- `CLAUDE_CODE_OAUTH_TOKEN`: Claude Code OAuth トークン

## 使用方法

### 自動実装（Tasks変更検知）

1. tasks.mdを更新するPRを作成
2. PRをmainブランチにマージ
3. ワークフローが自動実行され、実装用のPRが作成される

### 手動実装

1. GitHub Actionsページで "Manual Implementation Trigger" を選択
2. "Run workflow" をクリック
3. spec_idを入力（例: `001-feature-name`）
4. ワークフローが実行され、実装用のPRが作成される

## ワークフローのカスタマイズ

### パラメータ

再利用可能なワークフローは以下のパラメータをサポートします:

| パラメータ | 説明 | デフォルト値 |
|-----------|------|------------|
| `specs_dir` | specsディレクトリのパス | `specs` |
| `base_branch` | PRのベースブランチ | `main` |
| `claude_args` | Claude Code追加引数 | `--allowedTools "Bash,SlashCommand,Edit,Read,Write,Glob,Grep"` |
| `implement_command` | 実装コマンド | `/implement` |

### カスタマイズ例

#### 異なるディレクトリ構造

```yaml
with:
  specs_dir: 'documentation/specs'
  base_branch: 'develop'
```

#### Claude Codeの制限変更

```yaml
with:
  claude_args: '--allowedTools "Bash,SlashCommand,Edit,Read"'
```

## トラブルシューティング

### ワークフローが実行されない

- PRパスが正しいか確認: `specs/**/tasks.md`
- ベースブランチが正しいか確認
- `CLAUDE_CODE_OAUTH_TOKEN` が設定されているか確認

### 実装が失敗する

1. GitHub Actionsのログを確認
2. tasks.mdが正しい形式か確認
3. spec-kitのコマンドが正しく実行できるか確認

## 他のプロジェクトでの使用

このリポジトリのワークフローは、他のプロジェクトから直接参照できます。

### 利点

- ファイルコピー不要で、常に最新版を利用可能
- メンテナンスが容易
- バージョン管理が簡単(`@main`, `@v1.0.0` などでバージョン指定可能)

### 使用例

詳細は「導入手順 > 1. ワークフローファイルの作成 > 方法A: 直接参照(推奨)」を参照してください。

```yaml
jobs:
  execute-tasks:
    uses: omni-stove/specify-ci/.github/workflows/tasks-changed-reusable.yml@main
    with:
      specs_dir: 'specs'
      base_branch: 'main'
    secrets:
      claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

## ライセンス

MIT

## 参考リンク

- [spec-kit](https://github.com/github/spec-kit)
- [Spec-Driven Development Guide](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/)
- [Claude Code Action](https://github.com/anthropics/claude-code-action)
