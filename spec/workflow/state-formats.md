# state/ リソース仕様（Draft）

## 目的
- `state/` ディレクトリ配下にある進捗・成果物リソースの構造を統一し、マルチエージェント運用でのハンドオフを円滑にする。
- `spec/workflow/agents.md` に定義された役割／責務と連携し、**planner が状態を正しく読み、planner / coder / judge が迷わず更新できる**状態を保つ。

## ディレクトリ構成
```
state/
├── state.json               # 全体の進捗スナップショット
├── tickets/                 # チケット原本（planner 管理、Markdown）
│   └── <ticket-id>/
│       ├── index.md
│       └── C*-*.md
├── trace/                   # coder の実行ログ
│   └── <ticket-id>/
│       └── <timestamp>-<short-hash>.json
└── reports/                 # チケット単位のレポート
    └── <ticket-id>.md
```

## state.json
- planner が操作する進捗の単一ソース。トップレベルで `updated_at` と `tickets` を定義する。
- `tickets` はチケット ID をキーとしたオブジェクト。

```jsonc
{
  "updated_at": "2025-01-15T07:12:34Z",
  "tickets": {
    "T-0123": {
      "title": "API レスポンス時間の改善",
      "status": "In Review",          // No Status / Backlog / Ready / In Progress / In Review / Done
      "assignees": ["coder:hana"],
      "priority": "High",             // 任意: Low / Medium / High / Critical
      "due": "2025-01-20",
      "dependencies": ["T-0098"],
      "links": {
        "requirements": ["requirements.md#performance"],
        "tickets": ["state/tickets/T-0123/index.md"],
        "reports": ["state/reports/T-0123.md"]
      },
      "milestone": "2025Q1-perf",     // 任意
      "tags": ["performance", "backend"],
      "notes": "judge から再レビュー待ち"
    }
  }
}
```

### 更新ポリシー
- planner が状態遷移（`Backlog→Ready` など）を反映した瞬間に `updated_at` を刷新し、必要に応じて `spec/workflow/branching-strategy.md` に沿った GitHub アクション（ブランチ作成・PR 指示）を実行する。
- coder / judge は `state.json` を直接触らず、更新が必要な場合は planner に依頼する。
- 依存チケットが Done になった場合は planner が `dependencies` のリストを整理し、必要に応じて `Ready` へ遷移させる。

## tickets/<ticket-id>/
- planner が発行・保守するチケットの原本。SSOT (`requirements.md`) と同期する。
- 各チケットは `state/tickets/<ticket-id>/` ディレクトリで管理し、以下の Markdown ファイルを必須とする。
  - `index.md`: チケット全体のメタデータ・背景・受入基準を記述。
  - `C*-*.md`: チェックポイント（C1, C2 ...）単位の実装手順書。`spec/tasks/implementation-tasks.md` のエントリに対応し、関数／クラス契約・テスト仕様を詳細化する。

````markdown
---
Ticket: T-0123
Title: API レスポンス時間の改善
Priority: P1
Status: Backlog            # Backlog / Ready / In Progress / In Review / Done
Assignees:
  - planner:kei
Dependencies:
  - T-0098
Tags: [performance, backend]
Updated: 2025-01-12T07:31:22Z
Links:
  Requirements:
    - spec/overview/requirements.md#performance
  Reports:
    - state/reports/T-0123.md
---

# Problem
現状 API の p95 が 800ms と遅く、要件の 500ms を満たしていない。

# Value
レイテンシ改善によりユーザー離脱率を抑え、SLA を満たす。

# Acceptance Criteria
- p95 <= 500ms (loadtest 1,000 rps)
- 主要エンドポイントのピーク時エラー率 < 0.5%

# Checkpoints
| ID | File | Purpose | Owner |
|----|------|---------|-------|
| C1 | C1-bottleneck-survey.md | ボトルネック調査 | planner:kei |
| C2 | C2-cache-implementation.md | キャッシュ導入 | coder:hana |

# History
- 2025-01-10 Backlog (planner:kei)
- 2025-01-12 Ready (planner:kei)
````

- `C*-*.md` の先頭にも Front Matter を置き、契約対象（例: `module: models/renderers/canonical_depth.py`, `public_api: CanonicalDepthRenderer.forward`) とテスト項目 (`pytest` コマンド等) を明記する。planner は `index.md` の表を更新して各チェックポイントの状態を同期させる。
- `index.md` と `C*-*.md` は Markdown であるため、長文コンテキストや参考リンクを直接記述できる。各ファイルの最後に `# Trace` セクションを設け、関連 `state/trace/<ticket-id>/` エントリを列挙することを推奨。

### 更新ポリシー
- 要件変更は planner が本ファイルと `requirements.md` を同期させる（差分は `trace` へ記録）。
- `status_history` は planner が state 遷移を行った後に追記する（監査目的）。

## trace/<ticket-id>/<timestamp>-<short-hash>.json
- coder が実装・検証時に残す変更ログ。judge が妥当性判断を行う際の根拠になる。

```jsonc
{
  "id": "T-0123",
  "entry": "2025-01-14T05:22:11Z",
  "actor": "coder:hana",
  "summary": "キャッシュ層を導入し、ロードテストを実施",
  "commands": [
    "git checkout -b T-0123-cache",
    "docker compose up -d redis",
    "pytest tests/perf/test_api.py::test_p95_under_threshold"
  ],
  "artifacts": [
    "diff: git show HEAD",
    "metrics: logs/loadtest-20250114.csv"
  ],
  "environment": {
    "os": "Ubuntu 22.04",
    "python": "3.11.5",
    "docker": "24.0.5"
  },
  "commit": {
    "hash": "ab12cd3",
    "branch": "T-0123-cache"
  },
  "notes": "キャッシュ初回ヒット率 85%。追加調整の余地あり。"
}
```

### 更新ポリシー
- coder は主要な作業単位ごとにファイルを追加し、`timestamp` は ISO8601 UTC を使用する。
- `commands` は実行順に記録し、失敗コマンドも明示する（後続調査に役立つ）。
- 追加の検証や再実行を行った場合は、その実施者が `actor` を `coder:<name>` などで明示し、テスト結果を `summary` や `notes` に追記する。

## reports/<ticket-id>.md
- coder が成果と検証結果を要約し、planner / judge へのインプットとする。
- Markdown 形式で、以下のテンプレートを基本とする。

```markdown
---
Ticket: T-0123
Status: In Review          # state.json と揃える
Updated: 2025-01-15T07:05:11Z
Summary: キャッシュ導入で API p95 を 480ms に短縮
Confidence: Medium         # Low / Medium / High
---

## What Was Done
- Redis キャッシュレイヤを追加し、読み込みエンドポイントを対応
- ロードテスト (1,000 rps) で p95 480ms を確認

## What Wasn't Done
- 書き込み系エンドポイントの最適化は未着手

## Risks
- キャッシュミス時のフォールバックが未検証

## Next Actions
1. 書き込みエンドポイントのキャッシュ検討（planner 相談）
2. キャッシュ監視アラートの整備（coder:taro）

## Test Results
- `pytest tests/perf/test_api.py::test_p95_under_threshold` を coder:hana が実行し合格
- カナリアリリース手順は未整備：別途チケット化推奨
```

### 更新ポリシー
- coder が最新状況とテスト結果を反映する。
- judge の差戻し時は planner が `Summary` と `Next Actions` を更新し、差戻し理由を明記するか、更新を coder に依頼する。
- `reports/<ticket-id>.md` は常に最新状態を上書きし、履歴は `trace` で管理する。

> **Assets:** 票単位で図表や追加資料が発生する場合は `state/reports/<ticket-id>/figures/` や `tables/` などのサブディレクトリに保存し、該当 MD から参照する。

## 運用メモ
- `state.json` は planner の判断が唯一の真実、tickets / trace / reports は各ロールの観点から補完する。
- planner → coder → judge のフローで記録が揃っているか planner が `reports` を基に確認し、状態遷移と GitHub アクション（`spec/workflow/branching-strategy.md`）を判断する。
- 外部参照（例: metrics, dashboards）は `links` や `artifacts` に URL / パスを記録し、ハルシネーション抑止と再検証の容易化を図る。
- 設計ドキュメントは粒度に応じて 2 層構成にする：`spec/tasks/implementation-tasks.md` は P1/P0 単位の全体像を提供し、各チェックポイント C1/C2/... の詳細は `state/tickets/<ticket-id>/C*-*.md` にて関数・クラス契約レベルで定義する。planner は `state/tickets/<ticket-id>/index.md` の `Checkpoints` 表を更新し、coder / judge は該当 Markdown を実装・レビューの唯一の参照源とする。
