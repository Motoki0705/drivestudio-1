---
Ticket: T-FR-ARCH-001
Checkpoint: C2
Title: SuGaR Loss Functions
Status: Backlog
Owner: coder:TBD
Modules:
  - models/losses/sugar.py
  - models/losses.py
  - models/gaussians/deformgs.py
  - models/trainers/single.py
Tests:
  - tests/models/test_sugar_losses.py
  - tests/trainers/test_single_trainer_sugar_losses.py
Updated: 2025-10-20T00:00:00Z
---

# Scope
- 親チケット: `state/tickets/T-FR-ARCH-001/index.md`
- 上位仕様: `spec/tasks/implementation-tasks.md` の C2
- 目的: SuGaR Eq.8 / Eq.10 に基づく `R_surface`, `R_norm` を実装し、`f̂(p)` の近似ロジックを確立する。

# Deliverables
1. `models/losses/sugar.py` に `estimate_f_hat`, `sugar_surface_loss`, `sugar_normal_loss` を実装。
2. `models/losses.py` の `LossRegistry` / `LossDict` に `loss/sugar_surface`, `loss/sugar_norm` を登録。
3. `models/gaussians/deformgs.py` に `compute_true_sdf` を追加し、Gaussian パラメータから理想 SDF を計算。
4. Trainer 側で SuGaR ステージ時のみ損失評価を行い、`loss_dict` に反映。

# Implementation Checklist
## 関数仕様
- `models/losses/sugar.py`
  ```python
  def estimate_f_hat(
      depth: Tensor,
      gaussian_means: Tensor,
      ray_dirs: Tensor,
      camera_centers: Tensor,
      mask: Tensor,
  ) -> Tensor: ...

  def sugar_surface_loss(
      f_hat: Tensor,
      f_true: Tensor,
      weights: Tensor,
      mask: Tensor,
      eps: float = 1e-6,
  ) -> Tensor: ...

  def sugar_normal_loss(
      grad_f: Tensor,
      gaussian_normals: Tensor,
      mask: Tensor,
      eps: float = 1e-6,
  ) -> Tensor: ...
  ```
  - `estimate_f_hat`: `camera_centers + depth * ray_dirs` で交差点を求め、`gaussian_means` との距離差から `f̂(p)` を算出。正負符号は視線方向との差で決定。
  - `sugar_surface_loss`: `weights` はガウシアン重み（`softmax` 正規化済み）を想定。マスク false 部分は除外。
  - `sugar_normal_loss`: `grad_f` を `torch.autograd.grad` で計算するユーティリティを提供し、単位法線へ正規化。

## Gaussian 拡張
- `models/gaussians/deformgs.py`
  ```python
  def compute_true_sdf(self, sample_points: Tensor) -> Tensor:
      """
      Args:
          sample_points: [B, K, H, W, 3]
      Returns:
          f_true: same shape, computed from Gaussian scale.
      """
  ```
  - SuGaR Eq.7 に基づき、`scale_xyz` から `s_g*` を導出し `sqrt(-2 log d(p))` を実装。
  - CUDA / CPU で一貫するよう `torch.clamp` で数値安定化。

## Trainer 統合
- `models/trainers/base.py`
  - `self.losses_dict` へ `loss/sugar_surface` と `loss/sugar_norm` を追加（初期値 0）。
- `models/trainers/single.py`
  - SuGaR ステージ時に `estimate_f_hat` → `compute_true_sdf` → `sugar_*_loss` のパイプラインを組み、合計損失へ加算。
  - `loss_dict["loss/sugar_surface"]` 等を `float()` キャストしてロガーに渡す。

# Acceptance Criteria
- `estimate_f_hat` が深度ゼロ・距離ゼロの画素で厳密にゼロを返す（float tolerance 1e-6）。
- `sugar_surface_loss` が Δ=0.001 を与えた合成データで L1 的に Δ に比例する（勾配が NaN にならない）。
- 正常系の backward で `torch.autograd.grad(f_hat.sum(), depth)` が定義される。
- Trainer 実行時に `loss/sugar_surface`, `loss/sugar_norm` が TensorBoard / CSV に記録される。

# Test Plan
- `tests/models/test_sugar_losses.py`
  - `estimate_f_hat` の正負符号確認（手前側: 負、奥側: 正）。
  - `gradcheck` で `sugar_surface_loss` / `sugar_normal_loss` の微分可能性を確認。
  - `mask` false の箇所が損失へ寄与しないこと。
- `tests/trainers/test_single_trainer_sugar_losses.py`
  - ダミーシーンで forward → loss 計算 → backward が成功し、損失が正の有限値。

# Trace
- TODO: `state/trace/T-FR-ARCH-001/` を更新したら追記。
