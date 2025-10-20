# CODEX — Drivestudio Multi-Agent Onboarding

## Purpose
- 最短時間で新規エージェントが役割を理解し、日常業務に入れるようにする。
- 役割別に必読ドキュメントと確認手順を明示し、`spec/` のどこから読み始めれば良いかを示す。
- ブランチ運用や `state/` 更新の基礎ルールは本書のリンクから辿れるようにする。

## Quick Start (全エージェント共通)
1. `README.md` で環境構築・依存関係を確認。
2. 本 `CODEX.md` を読み、担当ロールのセクションを把握。
3. `spec/index.md` で仕様ドキュメント全体の目次を確認。
4. 役割別ルートに従い、`spec/` と `state/` の該当ファイルを読了。
5. 割り当てチケット (`state/tickets/<ticket-id>/index.md`) の `Problem` / `Acceptance Criteria` を確認し、planner へ疑問点があれば即時質問。

## Repository Landmarks
| 領域 | 目的 | 初回に目を通すべきファイル |
|------|------|-----------------------------|
| `spec/` | プロジェクトルール・手順・要件 | `spec/index.md` |
| `state/` | 進捗・チケット・レポート | `state/state.json`, `state/tickets/<ticket-id>/index.md` |
| `docs/` | 外部仕様・データセット手順 | `docs/Waymo.md` 等、チケットで指定されたもの |
| `tools/`, `configs/` | 実行スクリプトと Hydra 設定 | チケットの `Configs` 欄に従う |

## Role Playbooks — 読み進め方
### Planner
1. `spec/roles/agent-planner.md` で日次ルーチンとエスカレーション基準を把握。
2. `spec/workflow/state-formats.md` → `spec/workflow/branching-strategy.md` の順で、`state/` 更新とブランチ/PR 運用を再確認。
3. 新規チケットを扱う前に、該当 `state/tickets/<ticket-id>/index.md` と `Checkpoints` の `C*-*.md` を確認し、未整備なら追記。
4. `spec/tasks/implementation-tasks.md` で全 FR の進行状況を俯瞰し、優先順位を決める。
5. 進行中 PR / trace / reports をレビューし、必要な差戻しや researcher への依頼を判断。

### Coder
1. `spec/roles/agent-coder.md` を読み、`trace`/`reports` の更新基準とブランチ運用を把握。
2. `spec/workflow/state-formats.md` を通読し、`state/tickets/<ticket-id>/` の構造 (`index.md` + `C*-*.md`) を理解。
3. 割り当てチケットの `Checkpoints` リストを順番に開き、実装対象モジュール・テスト・期待成果物を確認。
4. 着手前に `spec/tasks/implementation-tasks.md` の該当 FR セクションを読み、関連する FR との依存を把握。
5. 実装後は `state/trace/<ticket-id>/` にコマンドログを追加し、`state/reports/<ticket-id>.md` の `What Was Done` / `Test Results` を更新。

### Judge
1. `spec/roles/agent-judge.md` でレビュー判定基準と差戻し手順を確認。
2. `spec/workflow/branching-strategy.md` のレビュー段階を読み、PR のチェックポイントを把握。
3. 対応チケットの `state/tickets/<ticket-id>/index.md` → `C*-*.md` を参照し、受入基準とテスト要件を明確化。
4. `state/reports/<ticket-id>.md` の `Test Results` / `Risks` を読み、`state/trace/<ticket-id>/` のログで再現性を確認。
5. レビューコメントは GitHub 上で残し、結論は `spec/roles/agent-judge.md` の手順に従って正式に記録。

### Researcher
1. `spec/roles/agent-researcher.md` を読み、調査依頼フローと成果物テンプレを理解。
2. `spec/workflow/state-formats.md` の `reports/` セクションを確認し、調査結果がどこへ反映されるか把握。
3. 調査依頼チケット (`state/tickets/<ticket-id>/index.md`) を確認し、`Notes` にある背景と参照リンクを精読。
4. エビデンスは `docs/` or `state/reports/<ticket-id>.md` の引用としてまとめ、planner へ引き渡す。

## Operational Checklist
- **コミュニケーション**: 疑問点・ブロッカーは 24h 以内に planner へ報告。
- **ブランチ命名**: `spec/workflow/branching-strategy.md` の `feature/<ticket-id>-<slug>` を厳守。
- **ログ**: 主要コマンドは必ず `state/trace/<ticket-id>/` に JSON で残す。
- **テスト**: チケットの `Tests` セクションに記載されたコマンドを最低限実行し、結果を `state/reports/<ticket-id>.md` に転記。

## Reference Flow (例: coder が FR-ARCH-001 C1 に着手する場合)
1. `CODEX.md` → 「Coder」節を読む。
2. `spec/tasks/implementation-tasks.md` で FR-ARCH-001 のチェックポイント一覧を把握。
3. `state/tickets/T-FR-ARCH-001/index.md` を開き、`Checkpoints` から `C1-canonical-depth-renderer.md` を確認。
4. 実装着手前に `spec/workflow/state-formats.md` の `trace` / `reports` ルールを再読。
5. 実装・テスト後に `state/trace/T-FR-ARCH-001/` と `state/reports/T-FR-ARCH-001.md` を更新し、planner にレビュー準備完了を報告。

## Questions & Escalation
- 不明点は Slack #planner もしくは `state/tickets/<ticket-id>/index.md` の `Notes` で共有。
- 本 CODEX 自体に修正が必要な場合は、`feature/<ticket-id>-codex-update` ブランチを作成し、spec と併せて PR を作成。
