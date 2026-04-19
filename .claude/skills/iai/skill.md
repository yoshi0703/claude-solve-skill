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
| **GitHub CLI** | `brew install gh` / [公式](https://cli.github.com/) | Issue・PR・マージ操作 |
| **Node.js** | `brew install node` | ビルド・テスト実行 |

## ワークフロー概要

```
Phase 1 抜刀: 症状トリアージ → 仮説駆動調査 → Git履歴調査 → 根本原因特定 → Issue作成
Phase 2 構え: プラン策定（テスト方針含む） → マルチLLMレビュー
Phase 3 血振り: テスト設計・作成（Red） → テストレビュー → 失敗を確認
Phase 4 斬撃: 実装（Green） → ステップ別レビュー → Codex最終ゲート → 敵対的検証（Verification）
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

## 失敗モードカタログ（⛔ MANDATORY — 過去の事故から学ぶ）

> **原則**: 実際に起きた失敗を、具体的な防止ゲートとして登録する。
> 抽象ルール（「気をつけよう」）は効果が薄い。「こういう時はこのチェックを実行する」という機械的な対応表にする。

各Phaseの担当エージェントは、該当する失敗モードのゲートを**必ず実行**すること。
ゲート未実施で次Phaseに遷移することは **fail-closed原則に反する**。

| ID | 失敗パターン | 過去の事例 | トリガー条件 | 防止ゲート | 実施Phase |
|----|------------|-----------|------------|-----------|----------|
| **F1** | 外部APIセマンティクス未検証 | Stripe `charge.amount_refunded` が累積値であることを知らず、部分返金で二重計上 | 外部API（Stripe/Supabase/OAuth等）の新しいフィールド・メソッドを使う | **External API Semantics Checklist** (Phase 2 planner) — 使うAPI callを全列挙し、各々について「cumulative vs incremental」「throw vs return-error」「replace vs merge」を公式ドキュメントで確認 | Phase 2 |
| **F2** | スキーマ・関数間整合性未確認 | unique制約にカラム追加したが、既存関数の`ON CONFLICT`が古いままでマイグレーション後に全操作が失敗 | `ALTER CONSTRAINT` / `ALTER INDEX` / カラム追加・削除を含むマイグレーション | **Schema Impact Analysis** (Phase 2 planner + Phase 4 verifier) — 変更するテーブル名で`grep -r`して、`ON CONFLICT`, `upsert`, `INSERT INTO`, 関数定義を全検索し、影響関数リストを明示 | Phase 2 + Phase 4 |
| **F3** | Early Return盲点 | メイン関数に処理を追加したが、別経路の関数（bulk処理・demo変換等）が早期returnするためコードに到達しない | 既存関数に処理を追加する / 条件分岐を含む関数を編集する | **Control Flow Map** (Phase 2 planner) — 編集する関数の全entry/exit pointをリスト化し、追加コードがどの経路で実行されるかを明示 | Phase 2 |
| **F4** | API層でのField Propagation欠落 | DBにカラム追加しフロントエンドで参照したが、間のEdge Functionのselect文に含めず、フロントで常にundefinedになる | 新しいDBカラムをフロントエンドから参照する | **End-to-End Field Propagation Check** (Phase 4 verifier) — 追加フィールドごとに `DB → RPC → Edge Function → Types → Frontend` の全層を`grep`で確認 | Phase 4 |
| **F5** | エンティティ階層の不完全処理 | 処理ハンドラが primary entity だけで検索し、secondary / child entity を見落とす | 1対Nや多段階層のデータモデルを扱う | **Entity Tree Traversal Checklist** (Phase 2 planner) — 新しいデータモデルの全階層（tier, parent, children, related）を列挙し、各処理がどの階層を対象とするか明示 | Phase 2 |
| **F6** | 境界値・Defensive Programming欠落 | ゼロ除算ガードなし、null/空配列の未ハンドル | 算術演算・配列操作・外部入力処理 | **Adversarial Test Matrix** (Phase 2 planner + Phase 3 test-designer) — 数値入力に対して必ず `0, 負値, null, undefined, Infinity`を、配列に対して`[], [1], [1000個]`を、除算に対して`分母=0`をテストケースに含める | Phase 2-3 |
| **F7** | TDDスキップ（meta失敗） | Phase 3を飛ばしてPhase 4へ直行。後付けテストは「実装に合わせたテスト」になりバグ検出力が落ちる | いかなる実装タスクでも発生可能 | **Hard Red Gate** (orchestrator) — Phase 4起動の前に「最低N件のfailing testが存在すること」を確認。N=3（正常/境界/異常）。Red未確認のままPhase 4起動は**ERROR扱い** | Phase 3→4遷移 |

### 失敗モード呼び出しプロトコル

planner / verifier / orchestrator は、タスクを受け取った瞬間に以下を判定する:

```
1. このタスクで発生しうる失敗モードはどれか？（F1-F7から該当するものを列挙）
2. 各失敗モードに対応するゲートを実施計画に組み込む
3. ゲート未実施のままPhase遷移しない
```

### カタログの成長プロトコル

新しい失敗が本番で発生したら、**この表に追記する**。居合は「同じミスを2度繰り返さない」ための仕組み。
表が成長するほどエージェントは賢くなる。

追記フォーマット: `| F{n+1} | {パターン名} | {発生事例} | {トリガー条件} | {ゲート名} | {Phase} |`

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
| `.claude/agents/iai-planner.md` | Phase 2 プラン策定（テスト設計含む） |
| `.claude/agents/iai-verifier.md` | Phase 4 敵対的検証（PASS/FAIL/PARTIAL） |
| `.claude/agents/opus-code-review.md` | Claude Opus レビューループ |
| `.claude/agents/codex-code-review.md` | Codex CLI 最終ゲート |

## カスタマイズ

プロジェクトへの導入時に設定すべき項目:

1. **`.claude/rules/review.md`** — プロジェクト固有のレビューチェック項目（必須）
2. **`.claude/rules/coding-rules.md`** — コーディングルール（任意）
3. **`.claude/rules/testing.md`** — テストルール（任意）
4. **`.claude/settings.json`** — パーミッション・Hooks設定（推奨、`settings.json.example` を参照）

詳細は [CUSTOMIZATION.md](../../CUSTOMIZATION.md) を参照。
