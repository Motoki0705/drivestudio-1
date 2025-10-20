---
Ticket: T-FR-ARCH-001
Title: SuGaR 正則化ステージ導入
Priority: P1
Status: Backlog
Assignees:
  - planner:codex
Owners:
  - planner:codex
Dependencies: []
Created: 2025-10-17
Updated: 2025-10-20T00:00:00Z
Links:
  Requirements:
    - spec/overview/requirements.md#fr-arch-001
  Spec:
    - spec/tasks/implementation-tasks.md#p1-fr-arch-001-—-sugar-正則化ステージ
  Reports:
    - state/reports/T-FR-ARCH-001.md
---

# Problem
OmniRe の動的カノニカル Gaussian は表面から浮きがちで、Waymo / nuScenes シーンではメッシュ抽出が不安定になるうえ、低不透明度ガウシアンの吹き溜まりでレンダリングが破綻する。SuGaR が提案する SDF ベースの整列正則化 (Eq.8, Eq.10) を欠いているため、カノニカル空間でのサーフェス拘束が効いていない。

# Value
学習ステージに SuGaR 正則化を組み込み、動的ノードのガウシアンをカノニカルサーフェスへ吸着させることで、FR-MESH-002 / FR-ARCH-003 の前提となる一貫したレベルセット生成とメッシュ拘束初期化を実現する。これにより動的カテゴリでのメッシュ再構成と再最適化が安定化し、要件 `spec/overview/requirements.md#fr-arch-001` の成果物 (loss ログ / ckpt) を出力できる。

# Acceptance Criteria
- Waymo サンプルシーンで MultiTrainer スケジュール (7k pretrain → 2k opacity entropy → 6k SuGaR) が NaN/Inf 無しで 15k step 完走し、SuGaR ステージ開始/終了が Hydra ログに記録される。
- Canonical depth renderer が各動的インスタンス・各ビューごとの Gaussian depth map を生成し、`loss/sugar_surface`, `loss/sugar_norm` が TensorBoard / CSV に出力される (再開時も継続ログ)。
- `tests/models/test_sugar_losses.py` が f_hat(p) の数値テストを通過し、`R_surface`, `R_norm` の実装が SuGaR Eq.8 / Eq.10 に準拠する。
- 学習再開 (`resume=True`) で SuGaR ステージの進捗と opacity entropy のスケジュールが再現され、loss が連続して減少するログが確認できる。

# Checkpoints
| ID | File | Scope | Owner | Status |
|----|------|-------|-------|--------|
| C1 | [C1-canonical-depth-renderer.md](./C1-canonical-depth-renderer.md) | カノニカル深度レンダラ拡張 | coder:TBD | Pending |
| C2 | [C2-sugar-losses.md](./C2-sugar-losses.md) | f̂(p) 推定と SuGaR 損失実装 | coder:TBD | Pending |
| C3 | [C3-stage-schedule.md](./C3-stage-schedule.md) | ステージ遷移 & opacity entropy | coder:TBD | Pending |
| C4 | [C4-logging-resume.md](./C4-logging-resume.md) | ログ配線と学習再開 | coder:TBD | Pending |
| C5 | [C5-test-harness.md](./C5-test-harness.md) | SuGaR テスト整備 | coder:TBD | Pending |

# History
- 2025-10-17 Backlog (planner:codex)

# Notes
- 旧 `state/tickets/T-FR-ARCH-001.yaml` の内容を本 Markdown に統合。
- Issue #9（feature/T-FR-ARCH-001 ブランチ親コミット誤り）を参照し、ブランチ作成時は `spec/workflow/branching-strategy.md` の再発防止策に従う。

# Trace
- TODO: `state/trace/T-FR-ARCH-001/` のエントリを追記。
