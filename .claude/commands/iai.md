---
name: iai
description: 居合 — 課題の分析からマージまでを一連の流れで自動実行
allowed-tools: Agent, Bash, Read
model: opus
---

# /iai コマンド

課題を受け取り、iai-orchestrator エージェントに委譲するエントリポイント。

## 引数パース

1. `$ARGUMENTS` から以下を抽出:
   - `--auto` フラグの有無 → `AUTO_MODE`（true/false）
   - それ以外のテキスト → `ISSUE_DESCRIPTION`（課題の説明）

2. `--auto` が含まれていなくても、ユーザーが「おまかせ」「任せた」「全自動で」と言った場合は `AUTO_MODE=true`

## 前提チェック

以下を Bash で確認し、不足があればユーザーに案内して停止:

```bash
command -v gh >/dev/null 2>&1 || echo "MISSING: gh (GitHub CLI)"
command -v codex >/dev/null 2>&1 || echo "MISSING: codex (Codex CLI)"
```

## 実行

前提チェック通過後、iai-orchestrator エージェントを起動する:

```
Agent(
  name: "iai-orchestrator",
  prompt: "以下の課題を解決してください。\n\n課題: {ISSUE_DESCRIPTION}\n自動モード: {AUTO_MODE}\n\nスキル定義(.claude/skills/iai/skill.md)を読み込み、定義に従ってPhase 1〜7を実行すること。",
  subagent_type: "general-purpose",
  mode: "bypassPermissions"
)
```

## エラー時

オーケストレーターが失敗した場合、チェックポイントファイル `.iai-checkpoint.json` の有無を確認し、
存在すれば途中再開を提案する。
