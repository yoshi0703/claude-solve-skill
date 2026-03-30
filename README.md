# /solve — Claude Code 完全自動開発サイクル

課題を一言伝えるだけで、Issue作成 → プラン策定 → マルチLLMレビュー → 実装 → テスト → PR → マージまでを自動実行する Claude Code スキル。

## 特徴

- **マルチLLMレビュー**: Claude Opus（反復ループ） + Codex CLI（最終ゲート）の2段階品質チェック
- **エージェント組織体制**: 調査・プラン・実装・レビュー・テスト・PR を専門サブエージェントに委譲
- **コンテキスト保護**: メインエージェントは指揮に専念、実作業はサブエージェントが担当
- **通常/全自動モード**: 確認しながら or `--auto` で一気通貫

## 使い方

### 1. インストール

```bash
# プロジェクトの .claude/ ディレクトリにコピー
cp -r .claude/skills/solve/ <your-project>/.claude/skills/solve/
cp -r .claude/agents/ <your-project>/.claude/agents/
cp -r .claude/rules/ <your-project>/.claude/rules/
```

### 2. 前提ツールのインストール

```bash
npm i -g @openai/codex    # Codex CLI
brew install gh            # GitHub CLI
```

### 3. カスタマイズ

`.claude/rules/review.md` の `Project-Specific Check Items` セクションに、プロジェクト固有のレビュールールを追加:

```markdown
## Project-Specific Check Items

- React Hooks must only be called at the top level of components
- All database queries must use `owner_id` (not `user_id`) for the stores table
- No usage of font-weight 300 (font-light)
```

### 4. 実行

```bash
# 通常モード（確認しながら）
/solve ダッシュボードのグラフが表示されない

# 全自動モード（マージまで一気に）
/solve --auto ログインボタンが反応しない
```

## ファイル構成

```
.claude/
├── skills/
│   └── solve/
│       └── skill.md          # メインスキル定義
├── agents/
│   ├── codex-code-review.md  # Codex CLI レビューエージェント
│   └── opus-code-review.md   # Claude Opus レビューエージェント
└── rules/
    └── review.md             # レビュールール（カスタマイズ用）
```

## ワークフロー

```
Phase 1: 課題分析 → Issue作成 → ブランチ作成
Phase 2: プラン策定 → Claude Opus レビューループ → Codex 最終ゲート
Phase 3: 実装 → Claude Opus diff レビューループ → Codex 最終ゲート
Phase 4: テスト実行 → (不足時: テスト作成サイクル)
Phase 5: PR作成
Phase 6: レビュー監視 → 対応 → マージ
Phase 7: 完了報告
```

## 2段階レビューゲートの設計思想

```
Claude Opus ループ（安い・反復向き）
    ↓ P0/P1 なしで合格
Codex CLI 最終ゲート（高い・外部視点・1回のみ）
    ↓ 合格 → 次の Phase へ
    ↓ 不合格 → Claude Opus ループに戻る
```

- **Claude Opus**: トークンコストが低く、修正→再レビューの反復に最適
- **Codex CLI**: 異なるLLM（GPT系）の視点で、Claude が見逃す問題を発見

## ライセンス

MIT
