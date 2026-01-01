# セクション検出アルゴリズム

本ドキュメントでは、tasks.mdからセクションを検出し、セクション単位でレビューを実行するための仕様を定義します。

---

## tasks.md セクションフォーマット

### 推奨構造

```markdown
## Section 1: Core Foundation

### Task 1.1: Define base types
**Creates:** `src/types/base.ts`
**Implements:** REQ-1.1

[タスク詳細...]

### Task 1.2: Implement utilities
**Creates:** `src/utils/helpers.ts`, `src/utils/helpers.test.ts`
**Modifies:** `src/config/index.ts`
**Implements:** REQ-1.2

[タスク詳細...]

## Section 2: Feature Implementation

### Task 2.1: Build main component
**Creates:** `src/components/Main.tsx`, `src/components/Main.test.tsx`
**Implements:** REQ-2.1, REQ-2.2

[タスク詳細...]
```

### フォーマット要素

| 要素 | パターン | 説明 |
|------|---------|------|
| セクション見出し | `## Section X: Name` | セクション境界を定義 |
| タスク見出し | `### Task X.Y: Title` | セクション内のタスク |
| 作成ファイル | `**Creates:** \`path\`` | タスクが作成するファイル |
| 変更ファイル | `**Modifies:** \`path\`` | タスクが変更するファイル |
| 要件トレース | `**Implements:** REQ-X.X` | 対応する要件ID |

---

## セクション解析アルゴリズム

### Step 1: セクション境界の検出

```
INPUT: tasks.md content
OUTPUT: sections[]

1. 行ごとにスキャン
2. `^## ` で始まる行をセクション境界として検出
3. 各セクションの範囲（開始行〜次のセクション or EOF）を記録
```

### Step 2: タスクの抽出

```
FOR each section:
    1. セクション範囲内で `^### Task ` パターンを検索
    2. タスクID（X.Y形式）を抽出
    3. タスクをセクションにマッピング
```

### Step 3: 期待ファイルの抽出

```
FOR each task:
    1. `**Creates:** ` の後のファイルパスを抽出
    2. `**Modifies:** ` の後のファイルパスを抽出
    3. バッククォート内のパス、カンマ区切り複数パス対応

    正規表現例:
    Creates: /\*\*Creates:\*\*\s*(.+)$/
    ファイル抽出: /`([^`]+)`/g
```

---

## セクションIDの生成

セクション見出しからセクションIDを自動生成：

```
入力: "## Section 1: Core Foundation"
出力: "section-1-core-foundation"

変換ルール:
1. "## " プレフィックスを除去
2. 小文字に変換
3. 英数字以外をハイフンに置換
4. 連続ハイフンを単一に圧縮
5. 先頭・末尾のハイフンを除去
```

---

## 完了判定アルゴリズム

### isSectionComplete(sectionId)

```
FUNCTION isSectionComplete(sectionId):
    section = getSectionById(sectionId)
    expectedFiles = section.expected_files

    FOR each file in expectedFiles:
        IF NOT fileExists(file):
            RETURN false

    RETURN true
```

### shouldTriggerReview(sectionId)

```
FUNCTION shouldTriggerReview(sectionId):
    // セクションが完了していること
    IF NOT isSectionComplete(sectionId):
        RETURN false

    // まだレビューされていないこと
    IF sectionAlreadyReviewed(sectionId):
        RETURN false

    RETURN true
```

---

## 状態管理

セクション完了状態は `.context/section-completion-state.json` で管理：

```json
{
  "feature": "my-feature",
  "parsed_at": "2025-12-30T10:00:00Z",
  "tasks_md_hash": "abc123...",
  "sections": {
    "section-1-core-foundation": {
      "name": "Section 1: Core Foundation",
      "line_range": [1, 25],
      "tasks": ["1.1", "1.2"],
      "expected_files": [
        "src/types/base.ts",
        "src/utils/helpers.ts"
      ],
      "status": "complete",
      "completed_at": "2025-12-30T12:00:00Z",
      "reviewed": true,
      "review_session_id": "codex-xyz"
    },
    "section-2-feature-impl": {
      "name": "Section 2: Feature Implementation",
      "line_range": [26, 50],
      "tasks": ["2.1"],
      "expected_files": [
        "src/components/Main.tsx",
        "src/components/Main.test.tsx"
      ],
      "status": "in_progress",
      "completed_at": null,
      "reviewed": false,
      "review_session_id": null
    }
  }
}
```

---

## 変更検知

### tasks.mdの変更検知

```
現在のハッシュ = hash(tasks.md内容)
保存済みハッシュ = state.tasks_md_hash

IF 現在のハッシュ != 保存済みハッシュ:
    // tasks.mdが変更された
    1. セクション構造を再解析
    2. 既存のレビュー状態を可能な限り維持
    3. 新しいセクションは未レビュー状態で追加
    4. 削除されたセクションは状態から除去
```

---

## エッジケース

### セクションがない場合

tasks.mdに `##` 見出しがない場合：

```
1. 全体を単一セクション "default" として扱う
2. 全タスク完了時にレビュー実行
```

### ファイルパス指定がない場合

タスクに `**Creates:**` / `**Modifies:**` がない場合：

```
1. 警告ログを出力
2. そのタスクは常に「完了」として扱う
3. セクション内の他タスクの完了で判定
```

### 複数ファイルの記述

カンマ区切りで複数ファイルを指定可能：

```markdown
**Creates:** `src/a.ts`, `src/b.ts`, `src/c.ts`
```

---

## 実装例

### Codex実行時のセクションコンテキスト

```bash
codex exec --sandbox read-only "
## セクション実装レビュー

### レビュー対象
セクション: {{SECTION_NAME}}
タスク: {{SECTION_TASK_IDS}}

### 変更ファイル
$(for file in {{SECTION_FILES}}; do echo "- $file"; done)

### セクション境界
- このセクションで完了すべき: {{IN_SCOPE_FEATURES}}
- 後続セクションで対応: {{OUT_OF_SCOPE_FEATURES}}

[プロンプト本文...]
"
```
