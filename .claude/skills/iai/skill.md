---
name: iai
description: 居合 — 課題の分析からマージまでを一連の流れで自動実行する完全自動開発サイクル
user-invocable: true
argument-hint: "<課題の説明> [--auto] - 例: ダッシュボードのグラフが表示されない / --auto で全自動モード"
tools: Agent, Bash, Read, Grep, Glob
model: opus
---

# iai — 居合

> 抜いて、斬って、納める。一つの流れで課題を解決する。

課題を一言伝えるだけで、Issue作成 → プラン策定 → マルチLLMレビュー → 実装 → テスト → PR → マージまでを自動実行する。

## 前提条件

| ツール | インストール | 用途 |
|--------|-------------|------|
| **Codex CLI** | `npm i -g @openai/codex` | 外部LLMによるコードレビュー |
| **GitHub CLI** | `brew install gh` / [公式](https://cli.github.com/) | Issue・PR・マージ操作 |
| **Node.js** | `brew install node` | ビルド・テスト実行 |

## ワークフロー概要

```
Phase 1 抜刀: 症状トリアージ → 仮説駆動調査 → Git履歴調査 → 根本原因特定 → Issue作成
Phase 2 構え: プラン策定 → マルチLLMレビュー
Phase 3 斬撃: 実装 → マルチLLMレビュー
Phase 4 血振り: テスト実行
Phase 5 納刀: PR作成
Phase 6 残心: レビュー監視 → 対応 → マージ
Phase 7 完了: サマリー報告
```

詳細なPhase定義は `iai-orchestrator` エージェントが管理する。

## 課題タイプ分類表

| タイプ | 判定基準 | 調査戦略 |
|--------|----------|----------|
| **CRASH** | スタックトレース・エラーメッセージあり | コールスタック逆引き + 直近変更の相関分析 |
| **WRONG_DATA** | 期待値と実際の値が異なる | データフロー追跡（入力→変換→出力） |
| **SLOW** | パフォーマンス劣化 | レイヤー別ボトルネック分離 |
| **INTERMITTENT** | 再現性が不安定 | 共有状態・競合条件の洗い出し |
| **UI_BROKEN** | 表示・操作の不具合 | コンポーネント状態検査 + レンダリングフロー追跡 |
| **INTEGRATION** | 外部サービス連携の不具合 | コントラクト検証 |
| **NEW_FEATURE** | 新規機能の実装 | 既存アーキテクチャ理解 + パターン調査 |
| **REFACTOR** | コードの構造改善 | 依存グラフ解析 + 影響範囲マッピング |

複数タイプに該当する場合は、データ整合性に関わるものを最優先で調査する。
タイプ別の詳細プロトコルは `iai-investigator` エージェントが保持する。

## 2段階レビューゲート

```
Claude Opus ループ（安い・反復向き）
    ↓ P0/P1 なしで合格
Codex CLI 最終ゲート（高い・外部視点・1回のみ）
    ↓ 合格 → 次の Phase へ
    ↓ 不合格 → Claude Opus ループに戻る
```

- **Claude Opus**: トークンコストが低く、修正→再レビューの反復に最適
- **Codex CLI**: 異なるLLM（GPT系）の視点で、Claude が見逃す問題を発見

### 重要度定義

| 重要度 | 説明 | 対応 |
|--------|------|------|
| **P0** | セキュリティ、データ整合性、クラッシュ | 必ず修正 |
| **P1** | パフォーマンス、型安全性、UX | 基本的に修正 |
| **P2** | コードスタイル、命名規則 | 報告のみ、任意 |

### 合格基準
- P0/P1 の指摘が 0 件で合格
- 最大リトライ: Codex 5回不合格でユーザーに判断を仰ぐ

## エージェント構成

| ファイル | 役割 |
|---------|------|
| `.claude/commands/iai.md` | エントリポイント（引数パース・前提チェック） |
| `.claude/agents/iai-orchestrator.md` | Phase 1〜7 の進行管理・品質ゲート |
| `.claude/agents/iai-investigator.md` | Phase 1 深層調査プロトコル |
| `.claude/agents/iai-planner.md` | Phase 2 プラン策定 |
| `.claude/agents/opus-code-review.md` | Claude Opus レビューループ |
| `.claude/agents/codex-code-review.md` | Codex CLI 最終ゲート |

## カスタマイズ

プロジェクトへの導入時に設定すべき項目:

1. **`.claude/rules/review.md`** — プロジェクト固有のレビューチェック項目（必須）
2. **`.claude/rules/coding-rules.md`** — コーディングルール（任意）
3. **`.claude/rules/testing.md`** — テストルール（任意）
4. **`.claude/settings.json`** — パーミッション・Hooks設定（推奨、`settings.json.example` を参照）

詳細は [CUSTOMIZATION.md](../../CUSTOMIZATION.md) を参照。
