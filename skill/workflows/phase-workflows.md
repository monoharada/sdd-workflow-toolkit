# Phase別レビューワークフロー詳細

本ドキュメントでは、各レビューフェーズの詳細な処理フローを説明します。

---

## Phase 1: 要件レビュー (spec-requirements完了後)

### トリガー
```bash
/sdd-codex-review requirements [feature-name]
```

### 処理フロー

1. **ファイル読み込み**
   ```bash
   .kiro/specs/[feature]/requirements.md
   .kiro/specs/[feature]/spec.json
   ```

2. **Codexレビュー実行**
   ```bash
   codex exec --sandbox read-only "
   あなたはcc-sdd要件レビュアーです。以下の要件定義をレビューしてください。

   ## レビュー基準
   1. 完全性: 全機能要件がカバーされているか
   2. 明確性: 曖昧な表現がないか
   3. テスト可能性: 受け入れ基準が検証可能か
   4. 一貫性: 要件間の矛盾がないか
   5. EARS形式準拠: When/If/The system shall 形式か

   ## 要件ファイル内容
   $(cat .kiro/specs/[feature]/requirements.md)

   ## 出力形式（JSON）
   {
     \"verdict\": \"OK\" | \"NEEDS_REVISION\",
     \"issues\": [...],
     \"summary\": \"全体評価\"
   }
   "
   ```

3. **結果解析と判定**
   - `verdict === "OK"` → 承認、次フェーズへ
   - `verdict === "NEEDS_REVISION"` → 自動修正適用、再レビュー

4. **自動修正（NEEDS_REVISION時）**
   - `issues`配列を走査
   - 各`suggestion`をrequirements.mdに適用
   - 再度Codexレビューを実行（最大6回）

5. **spec.json更新（OK時）**
   ```javascript
   spec.approvals.requirements.approved = true;
   spec.phase = "requirements-approved";
   spec.updated_at = new Date().toISOString();
   ```

6. **ユーザー報告**
   - レビュー回数
   - 解決済み指摘
   - 次のステップ提案

---

## Phase 2: 設計レビュー (spec-design完了後)

### トリガー
```bash
/sdd-codex-review design [feature-name]
```

### 処理フロー

1. **ファイル読み込み**
   ```bash
   .kiro/specs/[feature]/design.md
   .kiro/specs/[feature]/requirements.md  # 参照用
   ```

2. **Codexレビュー実行**
   - 設計レビュー基準（`.kiro/settings/rules/design-review.md`準拠）
   - GO/NO-GO判定
   - Critical Issues ≦ 3件

3. **判定基準**
   - `verdict === "GO"` → 承認
   - `verdict === "NO_GO"` → 自動修正 or ユーザー介入

4. **spec.json更新（GO時）**
   ```javascript
   spec.approvals.design.approved = true;
   spec.phase = "design-approved";
   ```

---

## Phase 3: タスクレビュー (spec-tasks完了後)

### トリガー
```bash
/sdd-codex-review tasks [feature-name]
```

### レビュー基準
- 要件カバレッジ: 全要件IDがタスクにトレースされているか
- タスク粒度: 各タスクが適切なサイズか
- 依存関係: タスク間の依存が正しく定義されているか
- 並列実行可能性: (P)マークが適切に付与されているか

### spec.json更新（APPROVED時）
```javascript
spec.approvals.tasks.approved = true;
spec.phase = "ready-for-implementation";
```

---

## Phase 4: 実装レビュー (spec-impl完了後)

### トリガー
```bash
/sdd-codex-review impl [feature-name]
```

### レビュー対象
- 変更されたソースファイル
- 対象タスクのtasks.md内の定義
- design.mdのインターフェース定義

### レビュー基準
- 要件準拠: 対象タスクの要件IDが満たされているか
- 設計準拠: design.mdのインターフェース定義に従っているか
- コード品質: TypeScript strict mode準拠、テストカバレッジ

### 設計コンテキストの提供（重要）

**NEEDS_REVISION時は、タスク範囲と設計意図を明示的に伝えて再レビューする**

段階的リファクタリングなど、タスクが分割されている場合：

```bash
codex exec --sandbox read-only resume [SESSION_ID] "
## 設計コンテキストの補足

### タスク分割の設計意図
- タスクX.X: [このタスクの範囲]を実施
- タスクY.Y: [関連する後続タスク]で対応予定

### 現タスクの完了条件
1. [条件1] ✅
2. [条件2] ✅

### 指摘への対応方針
[指摘内容]は[後続タスク]で対応予定。

この設計コンテキストを踏まえて再評価をお願いします。
"
```

### 実装レビュー専用コマンド
```bash
# mainブランチとの差分をベースにレビュー
codex exec --sandbox read-only review --base main "[追加の指示]"

# 未コミットの変更をレビュー
codex exec --sandbox read-only review --uncommitted "[追加の指示]"
```

---

## 自動進行モード

### トリガー
```bash
/sdd-codex-review auto [feature-name]
```

### 処理フロー

1. spec.jsonの現在`phase`を確認
2. 現在フェーズの成果物をCodexでレビュー
3. OKなら次フェーズへ自動進行
4. 全フェーズ完了までループ

```
initialized → requirements → design → tasks → impl → completed
```

---

## レビュー判定基準

| 判定 | 条件 |
|------|------|
| ok: true (OK/GO/APPROVED) | blocking = 0件 |
| ok: false (NEEDS_REVISION/NO_GO) | blocking ≧ 1件 |

### severity定義

| severity | 意味 | 影響 |
|----------|------|------|
| blocking | 修正必須 | 1件でも`ok: false` |
| advisory | 推奨・警告 | `ok: true`でも出力可 |

### category定義

- **correctness**: 正確性、要件/設計準拠
- **security**: セキュリティ（自動的にblocking）
- **perf**: パフォーマンス
- **maintainability**: 保守性
- **testing**: テストカバレッジ
- **style**: スタイル、形式

---

## 修正ループとテスト/リンタ統合

### 修正ループフロー

`ok: false`の場合、`max_iters`回まで反復：

```
[ok: false] → [issues解析] → [修正] → [テスト/リンタ] → [再レビュー]
    ↑                                                      │
    └──────────────────────────────────────────────────────┘
```

### テスト/リンタ自動実行

修正適用後に自動実行：

```bash
npx tsc --noEmit          # TypeScript型チェック
npx eslint --fix .        # ESLint
npx prettier --write .    # Prettier
npm test                  # テスト実行
```

### 停止条件

1. **ok: true** - レビュー承認
2. **max_iters到達** - 最大5回
3. **テスト2回連続失敗** - テストが安定しない

---

## Codex呼び出しパターン

### 初回レビュー
```bash
codex exec --sandbox read-only "[prompt]"
codex exec --sandbox read-only review --base main "[指示]"
```

### 再レビュー（2回目以降）
```bash
codex exec --sandbox read-only resume --last "[修正内容]"
codex exec --sandbox read-only resume [SESSION_ID] "[修正内容]"
```

**`resume`のメリット**:
- 前回コンテキスト保持
- トークン消費削減
- 高速レスポンス
