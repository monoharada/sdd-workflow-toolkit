# spec.json フォーマット仕様

レビュー結果とフェーズ進行状況を記録するspec.jsonの仕様です。

---

## 基本構造

```json
{
  "phase": "requirements-approved",
  "approvals": {
    "requirements": { "approved": true },
    "design": { "approved": false },
    "tasks": { "approved": false }
  },
  "updated_at": "2025-12-25T12:00:00.000Z",
  "codex_reviews": {
    "requirements": {
      "review_count": 2,
      "final_verdict": "OK",
      "resolved_issues": 3,
      "session_id": "codex-session-xyz",
      "timestamp": "2025-12-25T12:00:00.000Z"
    }
  }
}
```

---

## フィールド説明

### phase

現在のフェーズを示す文字列。

| 値 | 意味 |
|----|------|
| `initialized` | 初期状態 |
| `requirements-approved` | 要件レビュー承認済み |
| `design-approved` | 設計レビュー承認済み |
| `ready-for-implementation` | タスクレビュー承認済み、実装可能 |
| `impl-approved` | 実装レビュー承認済み |
| `completed` | 全フェーズ完了 |

### approvals

各フェーズの承認状況。

```json
{
  "requirements": { "approved": boolean },
  "design": { "approved": boolean },
  "tasks": { "approved": boolean }
}
```

### codex_reviews

Codexレビューの履歴。

```json
{
  "[phase]": {
    "review_count": number,        // レビュー回数
    "final_verdict": string,       // 最終判定（OK, GO, APPROVED等）
    "resolved_issues": number,     // 解決した指摘件数
    "session_id": string,          // Codex Session ID（必須）
    "timestamp": string            // ISO8601形式タイムスタンプ
  }
}
```

---

## フェーズ別更新例

### 要件レビュー承認時

```javascript
spec.approvals.requirements.approved = true;
spec.phase = "requirements-approved";
spec.updated_at = new Date().toISOString();
spec.codex_reviews.requirements = {
  review_count: 2,
  final_verdict: "OK",
  resolved_issues: 3,
  session_id: "abc123",
  timestamp: new Date().toISOString()
};
```

### 設計レビュー承認時

```javascript
spec.approvals.design.approved = true;
spec.phase = "design-approved";
```

### タスクレビュー承認時

```javascript
spec.approvals.tasks.approved = true;
spec.phase = "ready-for-implementation";
```

---

## 判定値マッピング

| フェーズ | PASS | FAIL |
|---------|------|------|
| requirements | OK | NEEDS_REVISION |
| design | GO | NO_GO |
| tasks | APPROVED | NEEDS_REVISION |
| impl | APPROVED | NEEDS_REVISION |

---

## セクション追跡（section_tracking）

### 概要

実装フェーズでセクション単位のレビューを行うための追跡情報。
tasks.mdの`##`見出しで定義されたセクション単位で進捗を管理します。

### 構造

```json
{
  "section_tracking": {
    "enabled": true,
    "total_sections": 3,
    "completed_sections": 1,
    "reviewed_sections": 1,
    "current_section": "section-2-feature-impl",
    "sections": {
      "section-1-core-foundation": {
        "name": "Section 1: Core Foundation",
        "tasks": ["1.1", "1.2"],
        "expected_files": [
          "src/types/base.ts",
          "src/utils/helpers.ts"
        ],
        "status": "complete",
        "reviewed": true,
        "review_session_id": "codex-section1-001"
      },
      "section-2-feature-impl": {
        "name": "Section 2: Feature Implementation",
        "tasks": ["2.1", "2.2"],
        "expected_files": [
          "src/components/Main.tsx",
          "src/components/Main.test.tsx"
        ],
        "status": "in_progress",
        "reviewed": false,
        "review_session_id": null
      }
    }
  }
}
```

### フィールド説明

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `enabled` | boolean | セクション追跡が有効か（デフォルト: true） |
| `total_sections` | number | 全セクション数 |
| `completed_sections` | number | 完了済みセクション数 |
| `reviewed_sections` | number | レビュー済みセクション数 |
| `current_section` | string | 現在作業中のセクションID |
| `sections` | object | セクション別詳細情報 |

### セクション別フィールド

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `name` | string | セクション表示名 |
| `tasks` | string[] | セクション内のタスクID配列 |
| `expected_files` | string[] | タスクが作成/変更するファイル一覧 |
| `status` | string | `"pending"` / `"in_progress"` / `"complete"` |
| `reviewed` | boolean | Codexレビュー済みか |
| `review_session_id` | string / null | Codex Session ID |

### 後方互換性

- `section_tracking`フィールドが存在しない場合: 従来動作（タスクごとのレビュー）
- tasks.mdに`##`見出しがない場合: 全体を単一セクションとして扱う

---

## セクション単位のimplレビュー

### codex_reviews.impl 拡張構造

セクション単位でレビューする場合、`impl`フィールドを拡張：

```json
{
  "codex_reviews": {
    "impl": {
      "mode": "section",
      "sections": {
        "section-1-core-foundation": {
          "review_count": 1,
          "final_verdict": "APPROVED",
          "resolved_issues": 2,
          "session_id": "codex-impl-section1-001",
          "timestamp": "2025-12-30T12:00:00.000Z"
        },
        "section-2-feature-impl": {
          "review_count": 0,
          "final_verdict": null,
          "session_id": null,
          "timestamp": null
        }
      },
      "all_sections_approved": false,
      "final_verdict": null,
      "timestamp": null
    }
  }
}
```

### フィールド説明

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `mode` | string | `"section"` / `"full"`（デフォルト: "full"） |
| `sections` | object | セクション別レビュー結果 |
| `all_sections_approved` | boolean | 全セクションがAPPROVEDか |
| `final_verdict` | string / null | 全体の最終判定 |

---

## 完全な例（セクション追跡あり）

```json
{
  "name": "user-authentication",
  "phase": "ready-for-implementation",
  "approvals": {
    "requirements": { "approved": true },
    "design": { "approved": true },
    "tasks": { "approved": true }
  },
  "updated_at": "2025-12-30T15:30:00.000Z",
  "section_tracking": {
    "enabled": true,
    "total_sections": 2,
    "completed_sections": 1,
    "reviewed_sections": 1,
    "current_section": "section-2-feature-impl",
    "sections": {
      "section-1-core-foundation": {
        "name": "Section 1: Core Foundation",
        "tasks": ["1.1", "1.2"],
        "expected_files": [
          "src/types/base.ts",
          "src/utils/helpers.ts"
        ],
        "status": "complete",
        "reviewed": true,
        "review_session_id": "codex-section1-001"
      },
      "section-2-feature-impl": {
        "name": "Section 2: Feature Implementation",
        "tasks": ["2.1", "2.2"],
        "expected_files": [
          "src/components/Main.tsx",
          "src/components/Main.test.tsx"
        ],
        "status": "in_progress",
        "reviewed": false,
        "review_session_id": null
      }
    }
  },
  "codex_reviews": {
    "requirements": {
      "review_count": 1,
      "final_verdict": "OK",
      "resolved_issues": 0,
      "session_id": "codex-req-001",
      "timestamp": "2025-12-25T10:00:00.000Z"
    },
    "design": {
      "review_count": 2,
      "final_verdict": "GO",
      "resolved_issues": 2,
      "session_id": "codex-design-002",
      "timestamp": "2025-12-25T12:00:00.000Z"
    },
    "tasks": {
      "review_count": 1,
      "final_verdict": "APPROVED",
      "resolved_issues": 0,
      "session_id": "codex-tasks-003",
      "timestamp": "2025-12-25T14:00:00.000Z"
    },
    "impl": {
      "mode": "section",
      "sections": {
        "section-1-core-foundation": {
          "review_count": 1,
          "final_verdict": "APPROVED",
          "resolved_issues": 2,
          "session_id": "codex-impl-section1-001",
          "timestamp": "2025-12-30T12:00:00.000Z"
        },
        "section-2-feature-impl": {
          "review_count": 0,
          "final_verdict": null,
          "session_id": null,
          "timestamp": null
        }
      },
      "all_sections_approved": false,
      "final_verdict": null,
      "timestamp": null
    }
  }
}
```

---

## 完全な例（従来方式・後方互換）

```json
{
  "name": "user-authentication",
  "phase": "impl-approved",
  "approvals": {
    "requirements": { "approved": true },
    "design": { "approved": true },
    "tasks": { "approved": true }
  },
  "updated_at": "2025-12-25T15:30:00.000Z",
  "codex_reviews": {
    "requirements": {
      "review_count": 1,
      "final_verdict": "OK",
      "resolved_issues": 0,
      "session_id": "codex-req-001",
      "timestamp": "2025-12-25T10:00:00.000Z"
    },
    "design": {
      "review_count": 2,
      "final_verdict": "GO",
      "resolved_issues": 2,
      "session_id": "codex-design-002",
      "timestamp": "2025-12-25T12:00:00.000Z"
    },
    "tasks": {
      "review_count": 1,
      "final_verdict": "APPROVED",
      "resolved_issues": 0,
      "session_id": "codex-tasks-003",
      "timestamp": "2025-12-25T14:00:00.000Z"
    },
    "impl": {
      "review_count": 3,
      "final_verdict": "APPROVED",
      "resolved_issues": 5,
      "session_id": "codex-impl-004",
      "timestamp": "2025-12-25T15:30:00.000Z"
    }
  }
}
```
