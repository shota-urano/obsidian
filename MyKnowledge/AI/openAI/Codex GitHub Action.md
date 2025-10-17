# 概要

- **Codex GitHub Action（[openai/codex-action](https://github.com/openai/codex-action)）** は、GitHub Actions 上で Codex（OpenAI のコーディングエージェント）を安全に実行する公式アクションです。リポジトリをチェックアウトした後、PR の差分やコンテキストを渡してレビュー・要約・自動修正提案などを行わせ、結果を出力します。アクション内部で Codex CLI のセットアップと Responses API への安全なプロキシ構成を行います。
- 利用には **`OPENAI_API_KEY` を GitHub Secrets として提供**するのが基本要件です（アクションの README に明記）。
---

# できること（代表例）

- **PR タイトル／本文の自動生成**: PR の base/head SHA と差分を渡し、JSON スキーマに沿った出力で安定化 → そのまま PR を更新。
- **コードレビュー自動化**: 変更点に限定したレビューコメント生成 → PR へコメント投稿。
- **出力スキーマ指定（`output-schema`）**: JSON Schema を渡して出力形式を強制でき、後続ステップの機械処理が安定。
- **権限とサンドボックス制御**: `safety-strategy`/`sandbox` で Codex の権限を細かく制御（既定は `drop-sudo`）。
---

# やり方（最短手順）

## 0) 前提

- GitHub リポジトリの **Settings → Secrets and variables → Actions → New repository secret** で  `OPENAI_API_KEY` を登録。

## 1) 最小ワークフロー（PR タイトル・本文自動生成）

`.github/workflows/pr-description.yml` を作成：

```
name: PR description
on:
  pull_request:
    types: [opened]

jobs:
  codex:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    outputs:
      final_message: ${{ steps.run_codex.outputs.final-message }}
    steps:
      - uses: actions/checkout@v5
        with:
          ref: refs/pull/${{ github.event.pull_request.number }}/merge
      - name: Pre-fetch base and head refs
        run: |
          git fetch --no-tags origin \
            ${{ github.event.pull_request.base.ref }} \
            +refs/pull/${{ github.event.pull_request.number }}/head
      - name: Run Codex
        id: run_codex
        uses: openai/codex-action@v1
        with:
          openai-api-key: ${{ secrets.OPENAI_API_KEY }}
          # 必要なら model や codex-args も指定可
          output-schema: |
            {
              "type": "object",
              "properties": {
                "title": {"type": "string"},
                "body":  {"type": "string"}
              },
              "required": ["title", "body"],
              "additionalProperties": false
            }
          prompt: |
            Repo: ${{ github.repository }}
            PR #${{ github.event.pull_request.number }}
            Base SHA: ${{ github.event.pull_request.base.sha }}
            Head SHA: ${{ github.event.pull_request.head.sha }}
            このPRの**変更点のみ**を要約し、上記スキーマに一致するJSONを出力。
            body は GitHub Flavored Markdown で。

  update-pr:
    runs-on: ubuntu-latest
    needs: codex
    if: needs.codex.outputs.final_message != ''
    permissions:
      pull-requests: write
    steps:
      - name: Update PR title/body
        uses: actions/github-script@v7
        env:
          CODEX_OUTPUT: ${{ needs.codex.outputs.final_message }}
        with:
          github-token: ${{ github.token }}
          script: |
            const out = JSON.parse(process.env.CODEX_OUTPUT);
            const number = context.payload.pull_request.number;
            await github.rest.pulls.update({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: number,
              title: out.title ?? context.payload.pull_request.title,
              body:  out.body  ?? context.payload.pull_request.body
            });

```
---

# 実用メモ（安全性・拡張）

- **実行順序**: 必ず `actions/checkout@v5` の後に `openai/codex-action` を置く（リポジトリ内容にアクセスさせるため）。
- **入力パラメータ**:
	- `openai-api-key`（必須）、`prompt`/`prompt-file`、`output-schema`、`model`、`codex-args`、`sandbox`、`safety-strategy` など。
- **サンドボックス／権限**:
    - 既定は `drop-sudo`（安全寄り）。Windows ランナーは `unsafe` が必要になる制約あり。
- **パターン拡張**:
    - レビューコメント投稿（`issues: write` 権限で `actions/github-script` からコメント）例が README にあり。
- **Zenn の実例**:
    - `output-schema` で JSON を固定 → 後段の PR 更新が安定。サンプル YAML が公開。