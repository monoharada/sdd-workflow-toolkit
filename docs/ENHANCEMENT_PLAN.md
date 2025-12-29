# SDD Codex Review Plugin 強化計画

## 概要

makaneko氏の「codex-review」スキル（[参考記事](https://note.com/makaneko_ai/n/n3cefcec49e2d)）を参考に、sdd-codex-review-pluginを強化する。

## 現状分析

### 現在の機能
- 4フェーズ対応: requirements, design, tasks, impl
- 自動修正・再レビューループ（最大6回）
- `codex exec --full-auto` でCodex実行
- コンテキスト節約モード（バックグラウンド実行）

### 課題
1. **セキュリティ**: `--full-auto`はCodexに書き込み権限を与える
2. **タイムアウト**: 大規模変更時にCodexがタイムアウト
3. **severity判定**: 3段階（critical/medium/minor）は判定が複雑
4. **コンテキスト継続**: 再レビュー時のコンテキストが引き継がれない

---

## 改善項目一覧

| 優先度 | 項目 | 効果 | 対象ファイル |
|--------|------|------|-------------|
| 高 | `--sandbox read-only` | セキュリティ強化 | SKILL.md |
| 高 | severity 2段階化 | 収束速度向上 | prompts/*.md |
| 高 | 規模判定+戦略分岐 | タイムアウト回避 | SKILL.md |
| 中 | notes_for_next_review | コンテキスト継続 | prompts/*.md, SKILL.md |
| 中 | タイムアウト時分割 | 安定性向上 | SKILL.md |
| 中 | 並列レビュー | 大規模変更対応 | SKILL.md |
| 低 | cross-check | 大規模変更品質 | SKILL.md, prompts/ |
| 低 | テスト/リンタ統合 | 品質ゲート強化 | SKILL.md |

---

## 詳細仕様

### 1. `--sandbox read-only` オプション追加【高】

#### 変更内容

```bash
# 変更前
codex exec --full-auto "[prompt]"

# 変更後
codex exec --sandbox read-only "[prompt]"
```

#### 理由
- Codexに書き込み権限を与えない
- レビュー専用として明確化
- 不要な操作のリスクを排除

#### 変更箇所
- `skill/SKILL.md`: Codex実行コマンドすべて

---

### 2. severity 2段階化【高】

#### 変更内容

```
# 変更前
severity: "critical" | "medium" | "minor"
判定: critical = 0 かつ medium ≦ 2 → OK

# 変更後
severity: "blocking" | "advisory"
判定: blocking = 0 → ok: true
```

#### フィールド定義

| severity | 意味 | 影響 |
|----------|------|------|
| blocking | 修正必須 | 1件でも`ok: false` |
| advisory | 推奨・警告 | `ok: true`でも出力可、レポートに記載のみ |

#### JSON出力スキーマ

```json
{
  "ok": true,
  "phase": "requirements|design|tasks|impl",
  "summary": "レビューの要約",
  "issues": [
    {
      "severity": "blocking",
      "category": "correctness|security|perf|maintainability|testing|style",
      "location": "ファイルパス:行番号",
      "problem": "問題の説明",
      "recommendation": "修正案"
    }
  ],
  "notes_for_next_review": "次回レビューへのメモ"
}
```

#### 変更箇所
- `skill/prompts/requirements-review.md`
- `skill/prompts/design-review.md`
- `skill/prompts/tasks-review.md`
- `skill/prompts/impl-review.md`
- `skill/SKILL.md`: 判定ロジック

---

### 3. 規模判定と戦略分岐【高】

#### 適用フェーズ
- `impl` フェーズのみ（他フェーズは単一ファイルのため不要）

#### 規模判定基準

```bash
# 規模判定コマンド
git diff HEAD --stat
git diff HEAD --name-status --find-renames
```

| 規模 | ファイル数 | 変更行数 | 戦略 |
|------|-----------|---------|------|
| small | ≤3 | ≤100 | diff のみ |
| medium | 4-10 | 100-500 | arch → diff |
| large | >10 | >500 | arch → diff並列 → cross-check |

#### 戦略詳細

**small**:
- 単純なdiffレビュー
- 現在のフローと同等

**medium**:
- archフェーズ: アーキテクチャ整合性確認
- diffフェーズ: 詳細コードレビュー

**large**:
- archフェーズ: アーキテクチャ整合性確認
- diffフェーズ: 並列実行（3-5サブエージェント）
- cross-checkフェーズ: 横断的整合性確認

#### 変更箇所
- `skill/SKILL.md`: 新セクション「規模判定（implフェーズ）」追加

---

### 4. notes_for_next_review フィールド【中】

#### 目的
再レビュー時にCodexが残したメモをプロンプトに含め、コンテキストを継続する。

#### 実装
1. Codex出力JSONに`notes_for_next_review`フィールドを追加
2. 再レビュー時にこのメモをプロンプトに含める

#### プロンプト追加内容

```
前回メモ: {notes_for_next_review}
```

#### 変更箇所
- `skill/prompts/*.md`: 全プロンプトテンプレート
- `skill/SKILL.md`: 再レビューフロー

---

### 5. タイムアウト時の分割リトライ【中】

#### 変更内容

```
# 変更前
タイムアウト → 指数バックオフ（5s, 15s, 45s）で3回リトライ

# 変更後
タイムアウト → ファイル数を半分に分割して1回リトライ
再失敗 → 該当ファイル群を「未レビュー」としてレポート
```

#### フロー

```
タイムアウト発生
    ↓
ファイル数を半分に分割
    ↓
分割後のファイル群で再レビュー
    ↓
成功 → 残りのファイル群をレビュー
失敗 → 「未レビュー」としてレポートに記録、手動確認推奨
```

#### 変更箇所
- `skill/SKILL.md`: エラーハンドリングセクション

---

### 6. 並列レビュー【中】

#### 適用条件
- `impl`フェーズ
- 規模: large（>10ファイル、>500行）

#### 実装詳細

```
並列度: 3-5サブエージェント
分割単位: ディレクトリ単位
最大: 1呼び出しあたり5ファイル/300行
統合: メイン（Claude Code）で実施
```

#### フロー

```
[large規模判定]
    ↓
[archフェーズ] 単一実行
    ↓
[diffフェーズ]
    ├─ サブエージェント1: src/components/
    ├─ サブエージェント2: src/services/
    └─ サブエージェント3: src/utils/
    ↓
[結果統合] Claude Codeがマージ
    ↓
[cross-checkフェーズ] 横断的チェック
```

#### 変更箇所
- `skill/SKILL.md`: 並列レビューセクション追加

---

### 7. cross-checkフェーズ【低】

#### 目的
大規模変更時に横断的な整合性を確認する。

#### 観点
- interface整合
- error handling一貫性
- 認可漏れ
- API互換性
- テスト網羅

#### プロンプトテンプレート

```
並列レビュー結果を統合し横断レビューせよ。出力はJSON1つのみ。

これはレビューゲートとして実行されている。横断的なblocking
（例: interface不整合、認可漏れ、API互換破壊）があればok: falseとせよ。

全体stat: {stat_output}
各グループ結果: {group_jsons}
観点: interface整合、error handling一貫性、認可、API互換、テスト網羅
```

#### 変更箇所
- `skill/prompts/cross-check-review.md`: 新規作成
- `skill/SKILL.md`: cross-checkフェーズ追加

---

### 8. テスト/リンタ統合【低】

#### 目的
修正ループ内でテスト/リンタを自動実行し、品質ゲートを強化する。

#### フロー

```
修正適用
    ↓
テスト/リンタ実行
    ↓
失敗 → 修正 → 再実行（最大2回）
    ↓
2回連続失敗 → ループ停止、ユーザー介入要求
    ↓
成功 → 再レビュー依頼
```

#### 停止条件追加
- `ok: true`
- `max_iters`到達
- **テスト2回連続失敗**（新規）

#### 変更箇所
- `skill/SKILL.md`: 修正ループセクション

---

## 実装チェックリスト

### Phase 1: ドキュメント
- [x] `docs/ENHANCEMENT_PLAN.md` 作成

### Phase 2: 優先度【高】
- [x] `--sandbox read-only` オプション追加
- [x] severity 2段階化（blocking/advisory）
- [x] 規模判定セクション追加

### Phase 3: 優先度【中】
- [x] `notes_for_next_review` フィールド追加
- [x] タイムアウト時分割リトライ
- [x] 並列レビュー対応

### Phase 4: 優先度【低】
- [x] cross-checkフェーズ追加
- [x] テスト/リンタ統合

---

## 実装完了日: 2025-12-29

### 変更ファイル一覧

**SKILL.md**:
- `--sandbox read-only` オプションに変更
- severity 2段階化 (blocking/advisory)
- 規模判定セクション追加
- 並列レビューセクション追加
- 修正ループとテスト/リンタ統合セクション追加
- エラーハンドリング強化（タイムアウト時分割リトライ）

**prompts/*.md**:
- requirements-review.md: 新スキーマ、notes_for_next_review
- design-review.md: 新スキーマ、notes_for_next_review
- tasks-review.md: 新スキーマ、notes_for_next_review
- impl-review.md: 新スキーマ、notes_for_next_review
- arch-review.md: 新規作成
- cross-check-review.md: 新規作成

---

## 参考資料

- [makaneko氏の記事](https://note.com/makaneko_ai/n/n3cefcec49e2d)
- [OpenAI Codex CLI](https://github.com/openai/codex)
- [PLANS.md for multi-hour problem solving](https://cookbook.openai.com/)
