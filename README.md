# SDD Workflow Toolkit

Claude Code用 Spec-Driven Development (SDD) ワークフローサポートツールキット

## 概要

このツールキットは、[OpenAI Codex CLI](https://github.com/openai/codex) を使用してClaude Codeの成果物を**独立レビュー**し、SDDワークフロー全体をサポートします。

### なぜ外部レビューが重要か

- **別視点からの問題発見**: 異なるモデル（Codex/GPT）が別の角度から検証
- **確証バイアスの排除**: 生成者と異なるレビュアーによる客観的評価
- **追跡可能性**: Codex Session IDによるレビュー履歴の証跡

---

## 特徴

| 機能 | 説明 |
|------|------|
| **4フェーズCodexレビュー** | requirements → design → tasks → impl の各フェーズを自動レビュー |
| **セクション単位レビュー** | tasks.mdのセクション単位で効率的にimplをレビュー |
| **E2Eエビデンス収集** | Playwright MCPでレビュー承認後のスクリーンショットを自動収集 |
| **プリフライトインタビュー** | 要件生成前にタイムボックス付き日本語質問で要件を明確化 |
| **規模別レビュー戦略** | small/medium/largeに応じた最適なレビュー戦略を自動選択 |
| **コンテキスト節約モード** | バックグラウンド実行と状態保存で長時間セッションも快適 |

---

## クイックスタート

### 前提条件

1. **Claude Code** がインストール済み
2. **Codex CLI** がインストール済み
   ```bash
   npm install -g @openai/codex
   codex auth status  # 認証確認
   ```
3. **Kiro spec構造** (`.kiro/specs/[feature]/`)

### インストール

```bash
# リポジトリをクローン
git clone https://github.com/monoharada/sdd-workflow-toolkit.git

# スキルディレクトリにコピー
cp -r sdd-workflow-toolkit/skill ~/.claude/skills/sdd-codex-review
```

### コマンド一覧

```bash
# フェーズ別レビュー
/sdd-codex-review requirements [feature-name]
/sdd-codex-review design [feature-name]
/sdd-codex-review tasks [feature-name]
/sdd-codex-review impl [feature-name]

# セクション単位レビュー（推奨）
/sdd-codex-review impl-section [feature-name] [section-id]
/sdd-codex-review impl-pending [feature-name]

# E2Eエビデンス収集
/sdd-codex-review e2e-evidence [feature-name] [section-id]

# 自動進行モード
/sdd-codex-review auto [feature-name]
```

---

## Kiroワークフロー統合

### 推奨ワークフロー

```
/kiro:spec-requirements [feature]
    ↓
/sdd-codex-review requirements [feature]  ← OK
    ↓
/kiro:spec-design [feature]
    ↓
/sdd-codex-review design [feature]        ← GO
    ↓
/kiro:spec-tasks [feature]
    ↓
/sdd-codex-review tasks [feature]         ← APPROVED
    ↓
/kiro:spec-impl [feature] [task]
    ↓
/sdd-codex-review impl-section [feature] [section-id]  ← APPROVED
    ↓
（[E2E]タグ付きセクションの場合）
/sdd-codex-review e2e-evidence [feature] [section-id]
    ↓
次のセクションまたは完了
```

### spec.json 自動更新

レビュー完了時、`spec.json` に以下の情報が追加されます：

```json
{
  "phase": "requirements-approved",
  "approvals": { "requirements": { "approved": true } },
  "codex_reviews": {
    "requirements": {
      "session_id": "codex-xxx",
      "final_verdict": "OK",
      "review_count": 2,
      "resolved_issues": 3
    }
  }
}
```

---

## セクション単位レビュー

### セクション定義

tasks.mdの`##`見出しでセクションを定義：

```markdown
## Section 1: Core Foundation
### Task 1.1: Define base types
**Creates:** `src/types/base.ts`

### Task 1.2: Implement utilities
**Creates:** `src/utils/helpers.ts`

## Section 2: Feature Implementation [E2E]
### Task 2.1: Build main component
**Creates:** `src/components/Main.tsx`
**E2E:** 初期表示、ユーザー操作確認
```

### 完了検知ロジック

1. `**Creates:**`/`**Modifies:**` からファイル一覧を抽出
2. 全ファイルの存在を確認
3. 完了 AND 未レビュー → Codexレビュー実行

### コマンド

| コマンド | 説明 |
|----------|------|
| `impl-section [feature] [section-id]` | 特定セクションをレビュー |
| `impl-pending [feature]` | 完了済み・未レビューのセクションを順次レビュー |

---

## E2Eエビデンス収集

### トリガー条件

```
Codexレビュー APPROVED AND [E2E]タグ付きセクション
```

### 実行フロー

1. セクションのCodexレビューが承認される
2. `[E2E]`タグを検出
3. Playwright MCPでスクリーンショットを収集
4. `.context/e2e-evidence/[feature]/[section]/` に保存

### 重要

**E2E失敗はブロッキングではありません** - エビデンス目的であり、品質ゲートではありません。

---

## 規模別レビュー戦略

implフェーズでは変更規模に応じて自動的に最適な戦略を選択：

| 規模 | ファイル数 | 行数 | 戦略 |
|------|-----------|------|------|
| small | ≤3 | ≤100 | diff のみ |
| medium | 4-10 | 100-500 | arch → diff |
| large | >10 | >500 | arch → 並列diff → cross-check |

### 各戦略の説明

- **diff**: 変更差分のみをレビュー
- **arch**: アーキテクチャ整合性確認（依存関係、責務分割、セキュリティ設計）
- **並列diff**: ファイルを3-5グループに分割して並列レビュー
- **cross-check**: グループ間の横断的整合性確認

---

## 判定基準

### severity

| severity | 意味 | 影響 |
|----------|------|------|
| **blocking** | 修正必須 | 1件でも `ok: false` |
| **advisory** | 推奨・警告 | `ok: true` でも出力可 |

### フェーズ別verdict

| Phase | PASS | FAIL |
|-------|------|------|
| requirements | OK | NEEDS_REVISION |
| design | GO | NO_GO |
| tasks | APPROVED | NEEDS_REVISION |
| impl | APPROVED | NEEDS_REVISION |

---

## トラブルシューティング

### Codex CLIが見つからない

```bash
npm install -g @openai/codex
codex auth status
```

### レビューが5回超過

最大5回のレビューサイクル後も承認されない場合、手動介入が必要です。

### Session IDがない

レビュー報告にSession IDがない場合、シミュレートの可能性があります。

---

## ディレクトリ構造

```
sdd-workflow-toolkit/
├── README.md
├── CLAUDE.md
├── LICENSE
├── skill/
│   ├── SKILL.md                  # メインスキル定義
│   ├── prompts/
│   │   ├── requirements-review.md
│   │   ├── design-review.md
│   │   ├── tasks-review.md
│   │   ├── impl-review.md
│   │   ├── section-impl-review.md
│   │   ├── e2e-evidence-prompt.md
│   │   ├── arch-review.md
│   │   └── cross-check-review.md
│   ├── workflows/
│   │   ├── phase-workflows.md
│   │   ├── section-detection.md
│   │   ├── e2e-evidence.md
│   │   ├── scale-strategies.md
│   │   └── context-saving.md
│   ├── reference/
│   │   └── spec-json-format.md
│   ├── evaluations/
│   │   ├── requirements-review-eval.json
│   │   ├── design-review-eval.json
│   │   └── impl-review-eval.json
│   └── templates/
│       └── review-report.md
├── integration/
│   ├── claude-md-snippet.md
│   ├── kiro-command-snippet.md
│   └── kiro-spec-requirements-preflight-interview-snippet.md
├── docs/
│   ├── ENHANCEMENT_PLAN.md
│   └── kiro-spec-requirements-preflight-interview.md
└── examples/
    ├── spec-json-example.json
    └── spec-requirements-template.md
```

---

## Best Practices Compliance

This skill follows [Claude Agent Skills Best Practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices).

| Criterion | Status |
|-----------|--------|
| Naming Convention | ✅ `sdd-codex-review` |
| Progressive Disclosure | ✅ workflows/, reference/, prompts/ |
| Feedback Loops | ✅ Auto-fix → test → re-review |
| Error Handling | ✅ Retry, timeout, fallback |
| Evaluations | ✅ 3 scenarios in evaluations/ |

---

## ライセンス

MIT License

## 関連リンク

- [OpenAI Codex CLI](https://github.com/openai/codex)
- [Claude Code](https://claude.ai/code)
- [Kiro](https://kiro.dev/) (Spec-Driven Development)
