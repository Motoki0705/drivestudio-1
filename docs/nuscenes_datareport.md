
# NuScenes データ構造レポート

このレポートは、NuScenesデータセットの`v1.0-mini`バージョンに含まれるJSONファイルの構造と、それらがどのように連携して自動運転のシナリオを表現しているかを解説します。

## 1. 全体構造

NuScenesのデータは、複数のJSONファイルによって階層的かつ正規化された形で管理されています。これにより、各データ（例：車両の姿勢、センサーのキャリブレーション情報）を重複なく効率的に格納しています。主要なJSONファイルとその関係性は以下の通りです。

```
scene.json
    |-- sample.json (シーンを構成するタイムスタンプ)
        |-- sample_data.json (各タイムスタンプでのセンサーデータ)
        |   |-- ego_pose.json (車両の自己位置)
        |   `-- calibrated_sensor.json (センサーのキャリブレーション情報)
        |       `-- sensor.json (センサー自体の情報)
        `-- sample_annotation.json (各タイムスタンプでのアノテーション)
            |-- instance.json (追跡される個々の物体)
            |   `-- category.json (物体のカテゴリ)
            `-- attribute.json (物体の状態)
```

## 2. 主要JSONファイルの詳細

### `scene.json`
- **役割**: データセット内の「シーン」を定義します。
- **主なフィールド**: 
    - `token`: シーンの一意なID。
    - `name`: シーン名（例: `scene-0061`）。
    - `description`: シーンの内容を説明するテキスト。
    - `first_sample_token`, `last_sample_token`: このシーンの最初と最後の`sample`を指すトークン。

### `sample.json`
- **役割**: シーン内の特定の「タイムスタンプ（瞬間）」を表現します。キーフレーム（keyframe）とも呼ばれます。
- **主なフィールド**:
    - `token`: サンプルの一意なID。
    - `timestamp`: タイムスタンプ（マイクロ秒単位）。
    - `scene_token`: このサンプルが属する`scene`のトークン。
    - `prev`, `next`: このサンプルの前後の`sample`を指すトークン（時系列順に辿るために使用）。

### `sample_data.json`
- **役割**: 各`sample`（瞬間）で、どのセンサーがどのデータを記録したかを定義します。
- **主なフィールド**:
    - `token`: データの一意なID。
    - `sample_token`: このデータが属する`sample`のトークン。
    - `ego_pose_token`: この瞬間の車両の自己位置（Ego Pose）を指すトークン。
    - `calibrated_sensor_token`: データ取得に使用されたセンサーのキャリブレーション情報を指すトークン。
    - `filename`: 実際のデータファイル（画像、LIDAR点群など）へのパス。
    - `is_key_frame`: このデータがキーフレームのものか否かを示すフラグ。

### `ego_pose.json`
- **役割**: 車両（Ego Vehicle）の自己位置と姿勢を記録します。
- **主なフィールド**:
    - `token`: Ego Poseの一意なID。
    - `translation`: グローバル座標系における車両の位置 `[x, y, z]`。
    - `rotation`: グローバル座標系における車両の姿勢（クォータニオン `[w, x, y, z]`）。

### `calibrated_sensor.json`
- **役割**: 各センサーのキャリブレーション情報を定義します。
- **主なフィールド**:
    - `token`: キャリブレーション情報の一意なID。
    - `sensor_token`: 対応する`sensor`のトークン。
    - `translation`: 車両（Ego）座標系から見たセンサーの位置 `[x, y, z]`（**外部パラメータ**）。
    - `rotation`: 車両座標系から見たセンサーの姿勢（クォータニオン `[w, x, y, z]`）（**外部パラメータ**）。
    - `camera_intrinsic`: カメラの内部パラメータ（3x3の行列）。LIDARやRADARの場合は空。

---

## 3. アノテーションの構造

アノテーションは、`sample_annotation.json`、`instance.json`、`category.json`の3つのファイルが連携して表現されます。

### `sample_annotation.json`
- **役割**: `sample`（瞬間）ごとに、検出された物体の3Dバウンディングボックスを定義します。
- **主なフィールド**:
    - `token`: アノテーションの一意なID。
    - `sample_token`: このアノテーションが付けられた`sample`のトークン。
    - `instance_token`: このアノテーションがどの`instance`（物体）に属するかを指すトークン。
    - `translation`: グローバル座標系におけるバウンディングボックスの中心位置 `[x, y, z]`。
    - `size`: バウンディングボックスのサイズ `[width, length, height]`。
    - `rotation`: グローバル座標系におけるバウンディングボックスの姿勢（クォータニオン `[w, x, y, z]`）。

### `instance.json`
- **役割**: シーンを通じて追跡される個々の物体（インスタンス）を定義します。例えば、ある歩行者がシーンの最初から最後まで追跡された場合、それは単一のインスタンスとなります。
- **主なフィールド**:
    - `token`: インスタンスの一意なID。
    - `category_token`: このインスタンスが属する`category`のトークン。
    - `nbr_annotations`: このインスタンスがシーン全体で何回アノテーションされたか。

### `category.json`
- **役割**: アノテーションのカテゴリ（物体の種類）を定義します。
- **主なフィールド**:
    - `token`: カテゴリの一意なID。
    - `name`: カテゴリ名（例: `human.pedestrian.adult`, `vehicle.car`）。
    - `description`: カテゴリの詳細な説明。

## 4. 座標系とデータフローのまとめ

アノテーション付き動画を生成する際、以下の流れでデータが処理されます。これは、このデータセットを扱う上での典型的なデータフローです。

1.  **シーンの選択**: `scene.json`から目的のシーンを特定します。
2.  **サンプルの取得**: シーンの`first_sample_token`から始まり、`sample.json`の`next`を辿って、シーンを構成する全サンプルのリストを取得します。
3.  **画像と姿勢情報の取得**: 各サンプルについて、`sample_data.json`を参照し、`CAM_FRONT`（正面カメラ）の画像ファイルパス、`ego_pose_token`、`calibrated_sensor_token`を取得します。
4.  **アノテーションの取得**: 同じく各サンプルについて、`sample_annotation.json`から対応するアノテーションを全て取得します。
5.  **座標変換と描画**:
    a.  **World -> Ego**: アノテーションのグローバル座標を、`ego_pose.json`から取得した車両の姿勢（Ego Pose）の逆行列を使って、車両中心の座標系に変換します。
    b.  **Ego -> Camera**: 次に、車両中心の座標を、`calibrated_sensor.json`から取得したカメラの外部パラメータ（Extrinsics）の逆行列を使って、カメラ中心の座標系に変換します。
    c.  **Camera -> Image**: 最後に、カメラ中心の3D座標を、同じく`calibrated_sensor.json`の内部パラメータ（Intrinsics）を使って、2Dの画像平面に投影します。
    d.  `instance.json`と`category.json`を辿ってカテゴリ名を取得し、投影したバウンディングボックスと共に画像に描画します。
6.  **動画生成**: 描画済みの画像を時系列順に動画ファイルとして結合します。

以上がNuScenesデータセットの主要な構造と、アノテーションデータの仕組みです。
