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

課題を一言伝えるだけで、Issue作成 → プラン策定 → テスト設計 → 実装 → 検証 → PR → マージまでを自動実行する。

> **テスト駆動開発: テストを先に書き、AIにはテストを通す実装を書かせる。**
> テストが仕様書であり、AIへの明確な正誤判定である。

## 前提条件

| ツール | インストール | 用途 |
|--------|-------------|------|
| **Codex CLI** | `npm i -g @openai/codex` | 外部LLMによるコードレビュー |
| **GitHub CLI** | `brew install gh` | Issue・PR・マージ操作 |
| **Node.js** | `brew install node` | ビルド・テスト実行 |

## エンジニアチーム型アーキテクチャ

> 統括リーダーはコードを読まない。チームの報告とCI結果（Codex）で判断する。

```
🧠 統括リーダー（iai-orchestrator）
│  見るもの: handoff artifact・diff --stat・Codex verdict のみ
│  やること: 組織化・遷移判断・ユーザー対話
│  やらない: コード読み・コード書き・詳細レビュー
│
├─ Phase 0: Preflight（環境チェック）
│
├─ Phase 1: 🔍 調査チーム
│  ├── investigator → 構造化 artifact を返す
│  └── git-archaeologist → Git調査サマリーを返す
│
├─ Phase 2: 📋 設計チーム（内部で品質ループを完結）
│  └── planner → 6軸スコアカード合格済みプランを返す
│  → Codex最終ゲート（PASS/FAIL/ERROR/SKIPPED）
│
├─ Phase 3-4: 💻 実装チーム（内部で品質ループを完結）
│  ├── test-designer → テスト作成（Red確認済み）
│  └── implementer(s) → ファイル所有権制で並列実装
│  → Codex最終ゲート → 統括リーダーが diff --stat で承認
│  → verifier → 敵対的検証
│
├─ Phase 5-6: 🚀 リリースチーム
│  └── PR作成 → レビュー監視 → マージ
│
└─ Phase 7: 完了報告
```

### 情報フロー制約

| 役割 | 見るもの | 見ないもの |
|------|---------|-----------|
| **統括リーダー** | handoff artifact, diff --stat, Codex verdict | ファイル内容, 実コード |
| **実装チーム** | 全コード, テスト結果, Opusレビュー | 他チームの内部状態 |
| **Codexレビュアー** | git diff全文, レビュー観点 | 実装過程の試行錯誤 |
| **Verifier** | plan artifact, diff, test results | 実装者の意図 |

## ワークフロー概要

```
Phase 0 準備: Preflight（codex, gh, node, 認証, ベースラインテスト確認）
Phase 1 抜刀: 症状トリアージ → 仮説駆動調査 → Git履歴調査 → 根本原因特定 → Issue作成
Phase 2 構え: プラン策定（6軸スコアカード） → Codex最終ゲート
Phase 3 血振り: テスト設計・作成（Red） → テストレビュー → 失敗を確認
Phase 4 斬撃: 実装（Green） → Codex最終ゲート → 敵対的検証（Verification）
Phase 5 納刀: PR作成
Phase 6 残心: レビュー監視 → 対応 → マージ
Phase 7 完了: サマリー報告
```

> 仕様書(Phase 2) → テスト(Phase 3) → 実装+検証(Phase 4) の順序。TDDの実践。
> Phase 4 の検証(Verification)は「壊しにいく」敵対的テスト。レビューとは別物。

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

## Verdict 統一 Enum

すべてのゲート・レビュー結果で以下の enum を使用する（3ファイル共通の単一真実源）:

| Verdict | 意味 | Phase遷移 |
|---------|------|----------|
| **PASS** | 成功 & P0/P1 なし | OK |
| **FAIL** | 成功 & P0/P1 あり | 修正ループ |
| **ERROR** | 実行不能/タイムアウト/認証失敗 | **停止 + ユーザー判断** |
| **SKIPPED** | diff/入力が空 | PASS扱い（ただしPhase 2/4では異常→ERROR扱い） |
| **PARTIAL** | 一部未検証 | ユーザー判断 |

### コンポーネント別の返却値

各コンポーネントが返す verdict の範囲は異なる:

| コンポーネント | 返す値 | 返さない値 |
|---------------|--------|-----------|
| **codex-code-review** | PASS, FAIL, ERROR, SKIPPED | PARTIAL |
| **iai-verifier** | PASS, FAIL, PARTIAL | ERROR, SKIPPED |
| **統括リーダー schema** | 全5値を受信可能 | — |

> **fail-closed原則**: ERROR は絶対に合格扱いしない。checkpoint保存して停止する。

### Codex 入力モード

| input_type | 用途 | Phase |
|------------|------|-------|
| **diff** | コードレビュー（git diff） | Phase 4 |
| **plan** | プランレビュー（テキスト） | Phase 2 |

### 2段階レビューゲート

```
チーム内 Opus ループ（安い・反復向き）
    ↓ P0/P1 なしで合格
統括リーダーが Codex CLI 最終ゲート起動（外部視点・1回のみ）
    ↓ PASS → 次の Phase へ
    ↓ FAIL → チームに差し戻し → Opus ループから再開
    ↓ ERROR → 停止 + ユーザー判断
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

## プラン品質スコアカード（6軸）

プランナーが**内部で**全軸合格するまでループし、合格して初めて統括リーダーに返す。

| 軸 | チェック内容 | 合格条件 |
|----|------------|---------|
| **完全性** | 受け入れ条件→テストケース変換可能か | 全条件カバー |
| **最小性** | スコープクリープなし | 変更ファイル≤10 |
| **リスク** | ロールバック手順が明記されているか | 手順あり |
| **テスト戦略** | Red→Green のケース設計 | 正常/境界/異常 各1+ |
| **依存整理** | ステップ間依存が明確 | 循環依存なし |
| **観測可能性** | 問題検知のログ・テスト手段が明記されているか | 検知手段あり |

## Handoff Artifact Schema

Phase間の受け渡しは自然文ではなく**構造化データ**:

**Phase 1 → orchestrator:**
```json
{ "root_cause": "...", "evidence": [], "candidate_files": [], "acceptance_criteria": [] }
```

**Phase 2 → orchestrator:**
```json
{ "plan_steps": [], "test_matrix": [], "risks": [], "rollback_procedure": "...", "scorecard": {} }
```

**Phase 4 → orchestrator:**
```json
{ "changed_files": [], "test_results": {}, "codex_verdict": "...", "verifier_verdict": "...", "residual_risks": [] }
```

## エージェント構成

| ファイル | 役割 |
|---------|------|
| `.claude/commands/iai.md` | エントリポイント（引数パース・前提チェック） |
| `.claude/agents/iai-orchestrator.md` | Phase 0〜7 の進行管理・品質ゲート |
| `.claude/agents/iai-investigator.md` | Phase 1 深層調査プロトコル |
| `.claude/agents/iai-planner.md` | Phase 2 プラン策定（6軸スコアカード内蔵） |
| `.claude/agents/iai-verifier.md` | Phase 4 敵対的検証 + 受け入れ条件トレーサビリティ |
| `.claude/agents/opus-code-review.md` | Claude Opus レビューループ（チーム内部用） |
| `.claude/agents/codex-code-review.md` | Codex CLI 最終ゲート（4値verdict） |

## カスタマイズ

プロジェクトへの導入時に設定すべき項目:

1. **`.claude/rules/review.md`** — プロジェクト固有のレビューチェック項目（必須）
2. **`.claude/rules/coding-rules.md`** — コーディングルール（任意）
3. **`.claude/rules/testing.md`** — テストルール（任意）
4. **`.claude/settings.json`** — パーミッション・Hooks設定（推奨）
