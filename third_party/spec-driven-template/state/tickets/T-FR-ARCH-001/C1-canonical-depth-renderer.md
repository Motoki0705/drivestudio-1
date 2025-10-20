---
Ticket: T-FR-ARCH-001
Checkpoint: C1
Title: Canonical Depth Renderer
Status: Backlog
Owner: coder:TBD
Modules:
  - models/renderers/canonical_depth.py
  - models/trainers/single.py
  - models/gaussians/deformgs.py
Configs:
  - configs/omnire_sugar.yaml::render.canonical_depth
Tests:
  - tests/renderers/test_canonical_depth_renderer.py
  - tests/trainers/test_single_trainer_canonical_depth.py
Updated: 2025-10-20T00:00:00Z
---

# Scope
- 親チケット: `state/tickets/T-FR-ARCH-001/index.md`
- 上位仕様: `spec/tasks/implementation-tasks.md` の C1
- 目的: 動的ガウシアンごとにカノニカル深度マップを生成し、SuGaR 損失 `f(p)` の入力を供給する。

# Deliverables
1. `models/renderers/canonical_depth.py` に `CanonicalDepthRenderer` クラスを新設。
2. `SingleTrainer` で SuGaR ステージ時のみ深度レンダリングを行い、`outputs["canonical_depth"]` / `outputs["canonical_mask"]` を追加。
3. Hydra 設定 (`configs/omnire_sugar.yaml`) に `render.canonical_depth` セクションを追加。
4. 実行ログを `state/trace/T-FR-ARCH-001/` に追加し、レンダリングの計測結果を共有。

# Implementation Checklist
## 新規モジュール
- `models/renderers/canonical_depth.py`
  ```python
  class CanonicalDepthRenderer(nn.Module):
      def __init__(
          self,
          near: float,
          far: float,
          height: int,
          width: int,
          camera_subset: Optional[List[int]] = None,
      ) -> None: ...

      def forward(
          self,
          gaussians: GaussianScene,
          cameras: CanonicalCameraBatch,
          instance_ids: Tensor,
      ) -> Dict[str, Tensor]:
          """
          Returns:
              depth: torch.FloatTensor [B, K, H, W]
              mask:  torch.BoolTensor  [B, K, H, W]
          """
  ```
  - `GaussianScene` / `CanonicalCameraBatch` は既存 render pipeline から import。存在しない場合は同等の dataclass を `models/renderers/__init__.py` にまとめる。
  - `forward` は `camera_subset` が指定された場合のみ該当ビューを処理。未指定時は入力バッチ全件。
  - 深度は正規化（`near` → 0, `far` → 1）し、未ヒット画素は 0 でマスク false。
  - GPU/CPU 双方で動作するよう `.to(device)` を透過的に扱う。

## 既存クラスの更新
- `models/trainers/single.py`
  - `SingleTrainer.__init__` で `self.canonical_depth_renderer` を初期化（Hydra 設定参照）。
  - `forward` / `render_gaussians` 後に、SuGaR ステージの場合のみ `canonical_depth_renderer` を呼び出し、`outputs` に深度・マスクを追加。
  - 既存 RGB パスへ影響しないよう `requires_canonical_depth(stage_name)` ガード関数を導入。
- `models/gaussians/deformgs.py`
  - `GaussianModel` に `get_canonical_params()` を追加し、中心・スケールを返す。
    ```python
    def get_canonical_params(self) -> Tuple[Tensor, Tensor]:
        """Returns (center_xyz, scale_radius) on device."""
    ```
  - 深度レンダラーが `instance_ids` から正しい中心・スケールを取得できるよう accessor を公開。

## Hydra 設定
- `configs/omnire_sugar.yaml`
  ```yaml
  render:
    canonical_depth:
      enable: true
      near: 0.1
      far: 80.0
      height: 256
      width: 256
      camera_subset: [0, 2]   # 任意
  ```
  - `enable` が false の場合はレンダラーを初期化しない。
  - 設定値は `SingleTrainer` で `OmegaConf` から取り出し、`CanonicalDepthRenderer` へ渡す。

# Acceptance Criteria
- Waymo サンプルバッチ（2 動的インスタンス）で `outputs["canonical_depth"].shape == (B, 2, H, W)` を確認し、ゼロ値以外に対して `canonical_mask` が true。
- 深度値が `near <= depth <= far` に収まり、`far` 付近への飽和が発生しない。
- RGB パスのみを呼び出した場合、従来の wall time から +5% 以内で完了する（計測ログを trace に添付）。
- `state/trace/T-FR-ARCH-001/` に実行コマンド（pytest 含む）を残す。

# Test Plan
- `tests/renderers/test_canonical_depth_renderer.py`
  - ダミーガウシアン（中心[0,0,5], 半径0.5）で深度が `[0.5, 0.55]` 付近となること。
  - `camera_subset=None` / `[0]` の切り替え試験。
  - CPU 実行時 fallback をカバーするため `torch.device("cpu")` での forward をテスト。
- `tests/trainers/test_single_trainer_canonical_depth.py`
  - SuGaR ステージ時のみ `outputs` に深度が追加されること。
  - `enable=false` の場合は `canonical_depth` キーが存在しないこと。

# Trace
- TODO: `state/trace/T-FR-ARCH-001/` を更新したら追記。
