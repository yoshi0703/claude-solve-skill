# /iai — 居合

> 抜いて、斬って、納める。一つの流れで課題を解決する。

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
cp -r .claude/skills/iai/ <your-project>/.claude/skills/iai/
cp -r .claude/agents/ <your-project>/.claude/agents/
cp -r .claude/rules/ <your-project>/.claude/rules/
```

### 2. 前提ツール・サービスのセットアップ

```bash
npm i -g @openai/codex    # Codex CLI
brew install gh            # GitHub CLI
```

#### Devin Review（推奨）

Phase 6 の「レビュー監視」は、PRに対して外部レビューが届くことを前提としています。
このスキルでは [Devin Review](https://devin.ai/) の利用を想定しており、PR作成後に Devin がレビューコメントを投稿するのを待ち、指摘があれば自動で修正・再プッシュします。

Devin Review を使わない場合は、Phase 6 のレビュー監視をスキップするか、他のレビューbot に合わせてカスタマイズしてください。

### 3. プロジェクトに最適化

**[CUSTOMIZATION.md](CUSTOMIZATION.md)** に詳細なカスタマイズガイドがあります。最低限やるべきことは:

1. `.claude/rules/review.md` にプロジェクト固有のレビュールールを追加
2. テスト・ビルドコマンドをプロジェクトに合わせて確認
3. （任意）コーディングルール・テストルールを追加

言語別の設定例（React, Python, Go, Swift）は [CUSTOMIZATION.md](CUSTOMIZATION.md) を参照。

### 4. 実行

```bash
# 通常モード（確認しながら）
/iai ダッシュボードのグラフが表示されない

# 全自動モード（マージまで一気に）
/iai --auto ログインボタンが反応しない
```

## ファイル構成

```
.claude/
├── skills/
│   └── iai/
│       └── skill.md          # メインスキル定義
├── agents/
│   ├── codex-code-review.md  # Codex CLI レビューエージェント
│   └── opus-code-review.md   # Claude Opus レビューエージェント
└── rules/
    └── review.md             # レビュールール（カスタマイズ用）
```

## ワークフロー

```
抜刀（Phase 1）: 課題分析 → Issue作成 → ブランチ作成
構え（Phase 2）: プラン策定 → マルチLLMレビュー
斬撃（Phase 3）: 実装 → マルチLLMレビュー
血振り（Phase 4）: テスト実行
納刀（Phase 5）: PR作成
残心（Phase 6）: レビュー監視 → 対応 → マージ
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
