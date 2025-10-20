---
Ticket: T-FR-ARCH-001
Checkpoint: C5
Title: Test Harness for SuGaR
Status: Backlog
Owner: coder:TBD
Modules:
  - tests/conftest.py
  - tests/renderers/test_canonical_depth_renderer.py
  - tests/models/test_sugar_losses.py
  - tests/trainers/test_stage_schedule.py
  - tests/trainers/test_sugar_resume.py
  - .github/workflows/*
Tests:
  - pytest -k sugar --maxfail=1
Updated: 2025-10-20T00:00:00Z
---

# Scope
- 親チケット: `state/tickets/T-FR-ARCH-001/index.md`
- 上位仕様: `spec/tasks/implementation-tasks.md` の C5
- 目的: SuGaR 関連のレンダラ／損失／スケジューラを自動テストでカバーし、CI での検証を可能にする。

# Deliverables
1. `tests/` 配下に SuGaR 専用の軽量テスト群を追加し、CPU/CI 上で 2 分以内に完走する。
2. `tests/conftest.py` に `dummy_sugar_scene` fixture を定義。
3. `pyproject.toml` / CI workflow を更新し、`pytest -k sugar --maxfail=1` を実行するタスクを追加。

# Implementation Checklist
## Fixtures
- `tests/conftest.py`
  ```python
  @pytest.fixture(scope="module")
  def dummy_sugar_scene():
      torch.manual_seed(0)
      means = torch.tensor([[0.0, 0.0, 4.5], [0.5, 0.1, 5.0]])
      scales = torch.full((2, 3), 0.2)
      normals = torch.tensor([[0.0, 0.0, -1.0], [0.0, 0.0, -1.0]])
      rays = torch.tensor([...])  # 16x16 grid
      depth = torch.linspace(0.5, 0.6, steps=256).reshape(1, 1, 16, 16)
      return SimpleNamespace(
          means=means,
          scales=scales,
          normals=normals,
          rays=rays,
          depth=depth,
      )
  ```
  - CUDA が無い環境でも実行できるよう `device="cpu"` 固定。

## テストスイート
- `tests/renderers/test_canonical_depth_renderer.py`
  - C1 のレンダラーをカバー。
- `tests/models/test_sugar_losses.py`
  - C2 の損失を数値検証。
- `tests/trainers/test_stage_schedule.py`
  - C3 のスケジューラを検証。
- `tests/trainers/test_sugar_resume.py`
  - C4 の再開挙動を検証。

## PyProject / CI
- `pyproject.toml`
  ```toml
  [tool.pytest.ini_options]
  addopts = "-k 'not slow'"
  markers = [
      "sugar: Tests for SuGaR integration",
  ]
  ```
  - 既存 `addopts` がある場合は `-k sugar` と競合しないよう調整。
- `.github/workflows/ci.yml`
  - `pytest -k sugar --maxfail=1 --durations=20` を追加し、結果を `artifact` として保存。

# Acceptance Criteria
- `pytest -k sugar --maxfail=1` がローカル（CPU）で 120 秒以内に完走。
- CI で SuGaR テストジョブが追加され、成功時にグリーン。
- テスト結果ログ（`pytest --junitxml` など）が `state/trace/T-FR-ARCH-001/` に格納される。

# Trace
- TODO: `state/trace/T-FR-ARCH-001/` を更新したら追記。
