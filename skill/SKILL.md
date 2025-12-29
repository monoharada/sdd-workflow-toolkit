---
name: sdd-codex-review
description: "Executes automated code reviews using OpenAI Codex CLI for SDD workflow phases (requirements, design, tasks, impl). Triggers after /kiro:spec-* completion. Loops until APPROVED with auto-fix. Use: /sdd-codex-review [phase] [feature-name]. Codex統合自動レビュー。"
---

# SDD-Codex-Review

Automated code review skill using OpenAI Codex CLI for Spec-Driven Development workflows.

## Quick Start (English)

```bash
# Review each phase
/sdd-codex-review requirements [feature-name]
/sdd-codex-review design [feature-name]
/sdd-codex-review tasks [feature-name]
/sdd-codex-review impl [feature-name]

# Auto-progress mode (review all phases sequentially)
/sdd-codex-review auto [feature-name]
```

**Key Points:**
- Executes actual `codex exec --sandbox read-only` commands (no simulation)
- Returns JSON verdict with Codex Session ID
- Auto-fixes issues and re-reviews until APPROVED (max 5 iterations)

---

## 概要

cc-sddワークフローの各フェーズ完了時にOpenAI Codexを使用して自動レビューを実行。
OKが出るまでフィードバックループを繰り返します。

## 前提条件

- `codex` CLIがインストール済み
- プロジェクトがcc-sdd仕様に準拠（`.kiro/specs/[feature]/`）
- spec.jsonが存在

---

## シミュレート禁止

**Codex CLIを実際に実行することが必須です。**

| 禁止 | 必須 |
|------|------|
| ❌ レビュー結果を自分で生成 | ✅ `codex exec`を実行 |
| ❌ Session IDなしで報告 | ✅ Session IDを報告に含める |

**Why?** Different models catch different issues. Session ID proves actual execution.

---

## 使用方法

```bash
# 特定フェーズのレビュー
/sdd-codex-review requirements [feature-name]
/sdd-codex-review design [feature-name]
/sdd-codex-review tasks [feature-name]
/sdd-codex-review impl [feature-name]

# 自動進行モード
/sdd-codex-review auto [feature-name]
```

---

## ワークフロー概要

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│ kiro:spec-* │ ──▶ │ Codex Review │ ──▶ │ 自動修正    │
│ 完了        │     │ (JSON出力)   │     │             │
└─────────────┘     └──────────────┘     └──────┬──────┘
                                                 │
       ┌──────────────────────────────┬──────────┴──────────┐
       ▼                              ▼                      ▼
┌──────────────┐                ┌──────────┐          ┌──────────┐
│ OK: 次フェーズ│                │ 再レビュー│          │ 5回超過: │
│ spec.json更新│                │ (ループ) │          │ ユーザー │
│ ユーザー報告 │                └──────────┘          │ 介入要求 │
└──────────────┘                                      └──────────┘
```

---

## レビュー判定基準

| 判定 | 条件 |
|------|------|
| ok: true | blocking = 0件 |
| ok: false | blocking ≧ 1件 |

### フェーズ別verdict

| Phase | PASS | FAIL |
|-------|------|------|
| requirements | OK | NEEDS_REVISION |
| design | GO | NO_GO |
| tasks | APPROVED | NEEDS_REVISION |
| impl | APPROVED | NEEDS_REVISION |

### severity

- **blocking**: 修正必須（1件で`ok: false`）
- **advisory**: 推奨・警告（レポートに記載のみ）

### category

correctness, security, perf, maintainability, testing, style

---

## 詳細ドキュメント

| トピック | ファイル |
|----------|----------|
| Phase 1-4詳細・修正ループ | [workflows/phase-workflows.md](workflows/phase-workflows.md) |
| 規模別戦略（small/medium/large） | [workflows/scale-strategies.md](workflows/scale-strategies.md) |
| コンテキスト節約モード | [workflows/context-saving.md](workflows/context-saving.md) |
| spec.json仕様 | [reference/spec-json-format.md](reference/spec-json-format.md) |

---

## プロンプトテンプレート

| フェーズ | テンプレート |
|----------|-------------|
| 要件 | [prompts/requirements-review.md](prompts/requirements-review.md) |
| 設計 | [prompts/design-review.md](prompts/design-review.md) |
| タスク | [prompts/tasks-review.md](prompts/tasks-review.md) |
| 実装 | [prompts/impl-review.md](prompts/impl-review.md) |
| アーキテクチャ | [prompts/arch-review.md](prompts/arch-review.md) |
| クロスチェック | [prompts/cross-check-review.md](prompts/cross-check-review.md) |

---

## Codex呼び出しパターン

### 初回レビュー
```bash
codex exec --sandbox read-only "[prompt]"
codex exec --sandbox read-only review --base main "[指示]"
```

### 再レビュー（session継続）
```bash
codex exec --sandbox read-only resume --last "[修正内容]"
codex exec --sandbox read-only resume [SESSION_ID] "[修正内容]"
```

---

## ユーザー報告フォーマット

```markdown
## Codexレビュー完了報告

### フェーズ: [PHASE] ([FEATURE])

| 項目 | 状態 |
|------|------|
| **Codex Session ID** | `[SESSION_ID]` |
| レビュー回数 | X / 5 |
| 最終判定 | [VERDICT] |

### サマリー
[Codexからのsummary]

### 解決済み指摘事項
1. [issue1] → [suggestion1適用]

### 次のステップ
[次に実行すべきコマンド]
```

---

## エラーハンドリング

| エラー | 対応 |
|--------|------|
| タイムアウト | ファイル分割してリトライ |
| 通常エラー | 1回リトライ |
| 再失敗 | 「未レビュー」として記録 |
| max_iters到達 | ユーザー介入要求 |

### レビュー完了待ち
- 60秒ごとに最大20回ポーリング
- 20回到達後も未完了: タイムアウト扱い

---

## 実装時の注意事項

1. **ファイル存在確認**: 対象ファイルの存在を必ず確認
2. **JSON抽出**: Codex出力から```json...```ブロックを正規表現で抽出
3. **差分適用**: suggestionは可能な限り自動適用
4. **履歴保持**: レビュー結果をspec.jsonのcodex_reviewsに記録
5. **順次処理**: 1フェーズずつ処理（依存関係のため）

---

## コンテキスト節約モード

**自動的に有効**。オプション指定不要。

1. レビュー開始前に状態を`.context/`に保存
2. バックグラウンドでCodex実行
3. 結果取得後に自動復元
4. 完了後にクリーンアップ

詳細: [workflows/context-saving.md](workflows/context-saving.md)
