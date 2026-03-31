# カスタマイズガイド

このスキルをクローンしたら、自分のプロジェクトに合わせて以下の3ステップでセットアップしてください。

---

## Step 1: レビュールールの定義（必須）

`.claude/rules/review.md` を開き、`Project-Specific Check Items` セクションにプロジェクト固有のチェック項目を追加します。

### 何を書くべきか

プロジェクトで**過去にバグの原因になった規約違反**や、**チームで合意済みのルール**を記述します。

### 記述例

<details>
<summary>React + TypeScript プロジェクト</summary>

```markdown
## Project-Specific Check Items

- React Hooks must only be called at the top level of components
- Stateful UI must be extracted into independent components
- No `font-light` (weight 300) — our custom font doesn't have this weight
- No usage of `user_id` for stores table — always use `owner_id`
- No phrases like "クレジットカード不要" — our service requires credit card
```
</details>

<details>
<summary>Python + Django プロジェクト</summary>

```markdown
## Project-Specific Check Items

- All views must use `@login_required` or explicit permission checks
- Database queries must use `.select_related()` or `.prefetch_related()` to avoid N+1
- No raw SQL queries — use Django ORM
- All model changes must include a migration file
- No `print()` statements — use `logging.getLogger(__name__)`
```
</details>

<details>
<summary>Go + gRPC プロジェクト</summary>

```markdown
## Project-Specific Check Items

- All exported functions must have godoc comments
- Error wrapping must use `fmt.Errorf("context: %w", err)` pattern
- Context must be the first parameter of all functions
- No `panic()` in library code — return errors instead
- All gRPC handlers must validate input with `validate` tags
```
</details>

<details>
<summary>Swift iOS プロジェクト</summary>

```markdown
## Project-Specific Check Items

- UI updates must be on the main thread (`@MainActor` or `DispatchQueue.main`)
- No force unwrapping (`!`) — use `guard let` or `if let`
- All network requests must handle timeout and cancellation
- Core Data operations must use a background context for writes
- Accessibility labels must be set on all interactive elements
```
</details>

---

## Step 2: テスト・ビルドコマンドの確認（推奨）

スキル内の以下の箇所を、プロジェクトのビルドシステムに合わせて確認します。

### skill.md 内の該当箇所

Phase 4 と Phase 5 で実行されるテスト・ビルドコマンド:

```bash
# デフォルト（Node.js / TypeScript プロジェクト想定）
npm run test
npx tsc --noEmit
npm run lint
```

### プロジェクト別の変更例

| プロジェクト種別 | テストコマンド | 型チェック | リンター |
|----------------|--------------|-----------|---------|
| Node.js + TS | `npm run test` | `npx tsc --noEmit` | `npm run lint` |
| Python | `pytest` | `mypy .` | `ruff check .` |
| Go | `go test ./...` | （不要） | `golangci-lint run` |
| Rust | `cargo test` | （`cargo build` に含む） | `cargo clippy` |
| Swift | `swift test` | （ビルドに含む） | `swiftlint` |

> **注意**: スキル本体（skill.md）を直接編集してもよいですが、
> コマンドはサブエージェントへの指示として渡されるため、
> CLAUDE.md にビルド・テストコマンドを記載しておけばサブエージェントが自動的に参照します。

---

## Step 3: コーディングルールの追加（任意）

より細かいコーディングルールがある場合、以下のファイルを作成するとサブエージェントが自動的に参照します。

### `.claude/rules/coding-rules.md`（任意）

```markdown
# Coding Rules

## 必須
- TypeScript の strict mode を有効にすること
- 変更は最小限に保つ
- エラーハンドリングを適切に行う

## 禁止
- `any` 型の使用（やむを得ない場合はコメントで理由を明記）
- `console.log` のデバッグ残し
- ハードコードされた API キーやシークレット
```

### `.claude/rules/testing.md`（任意）

```markdown
# Testing Rules

## テストランナー
- Vitest / Jest / pytest / go test（プロジェクトに合わせて）

## ルール
- AAA パターン（Arrange-Act-Assert）
- 正常系・境界値・異常系の最低3ケース
- モックは最小限
- テストデータはテスト内で完結
```

---

## 応用: 独自のレビューエージェント

Codex CLI 以外の外部LLMをレビューゲートに使いたい場合、
`.claude/agents/codex-code-review.md` を参考に独自エージェントを作成できます。

### 例: Gemini CLI を使う場合

```markdown
---
name: gemini-code-review
description: Gemini CLI を使ったコードレビューエージェント
tools: Bash, Glob, Grep, Read
model: haiku
---

# Gemini Code Review Agent

## 実行手順
1. レビュー観点を `.claude/rules/review.md` から取得
2. `git diff main...HEAD` で差分を取得
3. Gemini CLI でレビューを実行:

\```bash
gemini exec "以下のdiffをレビューしてください: {diff}"
\```
```

その場合、`skill.md` 内の `codex-code-review` への参照を新しいエージェント名に置き換えてください。

---

## Step 4: パーミッション・Hooks の設定（推奨）

`.claude/settings.json.example` をコピーしてプロジェクトに配置します。

```bash
cp .claude/settings.json.example .claude/settings.json
```

### パーミッション設定

スキルが使用するコマンドを事前許可することで、実行時の確認ダイアログを減らせます:

```json
{
  "permissions": {
    "allow": [
      "Bash(gh *)",
      "Bash(codex *)",
      "Bash(git *)",
      "Bash(npm run *)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force *)"
    ]
  }
}
```

### Hooks 設定

特定のツール呼び出しの前後にカスタム処理を挟めます:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash(git push *)",
        "hooks": [{ "type": "command", "command": "echo 'Pushing...'" }]
      }
    ]
  }
}
```

活用例:
- `git push` 前に Slack 通知
- PR 作成後にデプロイプレビュー起動
- テスト失敗時にエラーログ収集

---

## チェックリスト

セットアップ完了の確認に使ってください:

- [ ] `.claude/rules/review.md` にプロジェクト固有チェック項目を追加した
- [ ] Codex CLI がインストールされている（`codex --version`）
- [ ] GitHub CLI がインストールされている（`gh --version`）
- [ ] OPENAI_API_KEY が環境変数に設定されている
- [ ] `.claude/settings.json` をプロジェクトに配置した（settings.json.example からコピー）
- [ ] （任意）CLAUDE.md にビルド・テストコマンドが記載されている
- [ ] （任意）`.claude/rules/coding-rules.md` を作成した
- [ ] （任意）`.claude/rules/testing.md` を作成した

すべて完了したら `/iai <課題の説明>` で実行できます。
