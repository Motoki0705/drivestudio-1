---
Ticket: T-FR-ARCH-001
Checkpoint: C3
Title: Stage Scheduler & Opacity Entropy
Status: Backlog
Owner: coder:TBD
Modules:
  - models/trainers/base.py
  - models/trainers/single.py
  - models/trainers/scene_graph.py
Configs:
  - configs/omnire_sugar.yaml::optim.stage_schedule
  - configs/omnire_sugar.yaml::render.opacity_entropy
Tests:
  - tests/trainers/test_stage_schedule.py
  - tests/trainers/test_opacity_entropy_scheduler.py
  - tests/trainers/test_single_trainer_stage_resume.py
Updated: 2025-10-20T00:00:00Z
---

# Scope
- 親チケット: `state/tickets/T-FR-ARCH-001/index.md`
- 上位仕様: `spec/tasks/implementation-tasks.md` の C3
- 目的: 15k ステップの学習を 7k / 2k / 6k に分割し、SuGaR ステージ移行と opacity entropy の減衰を制御する。

# Deliverables
1. `models/trainers/base.py` に `StageSchedule` クラスと `OpacityEntropyScheduler` を追加。
2. `SingleTrainer` / `MultiTrainer` でステージ開始・終了イベントを管理し、SuGaR ステージでのみ深度 / 損失計算を有効化する。
3. `configs/omnire_sugar.yaml` にステージスケジュールと entropy パラメータを定義。
4. Resume 時にステージ進捗を復元できるよう状態管理を実装。

# Implementation Checklist
## スケジューラ設計
- `models/trainers/base.py`
  ```python
  @dataclass
  class StageSpec:
      name: str
      steps: int
      hooks: Dict[str, bool] = field(default_factory=dict)

  class StageSchedule:
      def __init__(self, stages: List[StageSpec]) -> None: ...
      def step(self, global_step: int) -> Tuple[str, int]: ...
      def to_state(self) -> Dict[str, Any]: ...
      @classmethod
      def from_state(cls, state: Dict[str, Any]) -> "StageSchedule": ...
  ```
  - `step` は 0-indexed。例: global step 0-6999 → `("pretrain", 0-6999)`。
  - `to_state` は resume 用に `{"stage_index": int, "offset": int}` を返す。

- `OpacityEntropyScheduler`
  ```python
  class OpacityEntropyScheduler:
      def __init__(self, table: Dict[str, float]) -> None: ...
      def weight(self, stage_name: str) -> float: ...
  ```
  - `table` には `"pretrain": 0.0`, `"entropy": 0.3`, `"sugar": 0.1` などを設定。

## Trainer 更新
- `models/trainers/base.py`
  - `self.stage_schedule` と `self.opacity_entropy_scheduler` を初期化。
  - `save_state_dict` / `load_state_dict` に `stage_schedule.to_state()` を含める。
- `models/trainers/single.py`
  - `self.current_stage, self.stage_step = self.stage_schedule.step(self.step)`
  - SuGaR ステージ（`current_stage == "sugar"`）でのみ深度レンダリング・損失計算を起動。
  - `self.render_cfg.opacity_entropy_weight = self.opacity_entropy_scheduler.weight(self.current_stage)`
- `models/trainers/scene_graph.py`（MultiTrainer が存在する場合）
  - 子トレーナーへ `current_stage` / `stage_step` をブロードキャスト。

## Hydra 設定
- `configs/omnire_sugar.yaml`
  ```yaml
  optim:
    stage_schedule:
      - { name: pretrain, steps: 7000 }
      - { name: entropy,  steps: 2000 }
      - { name: sugar,    steps: 6000 }
  render:
    opacity_entropy:
      table:
        pretrain: 0.0
        entropy: 0.3
        sugar: 0.1
  resume_state:
    stage:
      stage_index: 0
      offset: 0
  ```
  - `resume_state.stage` は checkpoint 読み込み時に上書き可能なデフォルト値。

# Acceptance Criteria
- 15,000 ステップで `stage_schedule.step` が `[pretrain]*7000 → [entropy]*2000 → [sugar]*6000` を返す。
- `resume=True` で 8,000 ステップ checkpoint をロードした場合、`current_stage == "entropy"`, `stage_step == 1000` を再現。
- `render_cfg.opacity_entropy_weight` がステージ遷移時に即座に更新され、SuGaR ステージで 0.1 に変化する。
- `state/trace/T-FR-ARCH-001/` にステージ遷移ログ（INFO 出力 + コマンド）が記録される。

# Test Plan
- `tests/trainers/test_stage_schedule.py`
  - `StageSchedule.step` の境界値（6999/7000, 8999/9000, 14999）を検証。
  - `to_state` / `from_state` の round-trip。
- `tests/trainers/test_opacity_entropy_scheduler.py`
  - 未定義ステージで `KeyError` を発生させる。
- `tests/trainers/test_single_trainer_stage_resume.py`
  - 疑似 trainer を 8,000 ステップ進め → state 保存 → 再読み込み → ステージ一致を確認。

# Trace
- TODO: `state/trace/T-FR-ARCH-001/` を更新したら追記。
