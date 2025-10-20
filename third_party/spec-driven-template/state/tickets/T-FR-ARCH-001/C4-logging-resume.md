---
Ticket: T-FR-ARCH-001
Checkpoint: C4
Title: Logging & Resume Support
Status: Backlog
Owner: coder:TBD
Modules:
  - models/trainers/base.py
  - models/trainers/single.py
  - tools/logger.py
  - main.py
Tests:
  - tests/trainers/test_sugar_resume.py
  - tests/trainers/test_logger_output.py
Updated: 2025-10-20T00:00:00Z
---

# Scope
- 親チケット: `state/tickets/T-FR-ARCH-001/index.md`
- 上位仕様: `spec/tasks/implementation-tasks.md` の C4
- 目的: SuGaR ステージ固有のログとチェックポイント再開を安定化し、訓練途中の中断から復帰してもロス履歴を継続させる。

# Deliverables
1. Trainer の save/load にステージ状態 (`current_stage`, `stage_step`) を含める。
2. ロガーに `loss/sugar_surface`, `loss/sugar_norm`, `metrics/stage_id` を出力させる。
3. Resume 時に損失ログが連続的に記録されることを保証。
4. レポート／トレースへ再開手順・ログパスを追加。

# Implementation Checklist
## 状態保存
- `models/trainers/base.py`
  ```python
  def save_state_dict(self) -> Dict[str, Any]:
      state = super().save_state_dict()
      state["schedule"] = self.stage_schedule.to_state()
      state["step"] = self.step
      state["stage_name"] = self.current_stage
      state["stage_step"] = self.stage_step
      return state

  def load_state_dict(self, state: Dict[str, Any]) -> None:
      super().load_state_dict(state)
      schedule_state = state.get("schedule")
      if schedule_state is not None:
          self.stage_schedule = self.stage_schedule.__class__.from_state(schedule_state)
      self.step = state.get("step", 0)
      self.current_stage = state.get("stage_name", "pretrain")
      self.stage_step = state.get("stage_step", 0)
  ```
  - 旧バージョン checkpoint にも対応するため `dict.get` で後方互換を確保。

## ロギング
- `models/trainers/base.py` または `trainer.log_train_scalars`
  - `metrics/stage_id = {"pretrain": 0, "entropy": 1, "sugar": 2}[self.current_stage]`
  - `loss_dict.setdefault("loss/sugar_surface", torch.tensor(0., device=self.device))`
  - `LossDict` の初期化時点で SuGaR 損失を 0 クリアする。
- `tools/logger.py`
  - TensorBoard: `self.tb_writer.add_scalar(key, value, global_step)`
  - CSV: `writer.writerow({"step": self.step, "loss/sugar_surface": ..., ...})`
  - Append モードでオープンし、resume 時も時系列が連続するように `global_step` を明示。

## Resume フロー
- `main.py` / `train.py`
  - `resume_state.stage` が指定されている場合は `StageSchedule.from_state` を呼び出した後に `load_state_dict`。
  - CLI 引数 `--resume` 時に `tensorboard_logger` を append モードで開く。
  - 再開直後のログに `metrics/stage_id` と SuGaR損失が出力されることを `INFO` ログでも通知。

## レポート更新
- `state/reports/T-FR-ARCH-001.md`
  - `Test Results` セクションに再開コマンド例 (`python main.py +experiment=omnire_sugar resume=true ckpt=...`) を追記。
- `state/trace/T-FR-ARCH-001/`
  - 再開試験のコマンド・ログファイルパスを残す。

# Acceptance Criteria
- チェックポイント保存 → `torch.load` → `load_state_dict` → 直後のステップで `metrics/stage_id` が連続して記録される。
- `resume=True` で再開後に `loss/sugar_surface` のタイムスタンプ／ステップ番号が途切れない。
- CSV ロガーで `loss/sugar_*` 列が自動的に生成される。
- `state/reports/T-FR-ARCH-001.md` に再開手順が明記され、judge が追試可能。

# Test Plan
- `tests/trainers/test_sugar_resume.py`
  - 疑似トレーナーで 5k ステップ進め checkpoint 保存 → `load_state_dict` → 5k+1 ステップ実行 → `loss/sugar_surface` が連続値。
  - ロガー（CSV/TensorBoard）をモックし、`metrics/stage_id` が記録されるか assert。
- `tests/trainers/test_logger_output.py`
  - CSV 行数がステップ数と一致。

# Trace
- TODO: `state/trace/T-FR-ARCH-001/` を更新したら追記。
