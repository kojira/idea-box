# Video Event Detector — コンセプト設計ドキュメント

## 概要

動画ストリーム（ライブ配信・録画）内の「意味のあるイベント」をリアルタイムで検知する汎用フレームワーク。

矩形領域（ROI: Region of Interest）とラベルの組み合わせで状態変化を定義し、変化が起きた時だけ処理を発火させる。

---

## コアコンセプト

### 「ラベル付き矩形の状態変化」がイベント

```
検出器(フレーム) → [
  { label: "slide-title",   bbox: [x, y, w, h], confidence: 0.95 },
  { label: "page-number",   bbox: [x, y, w, h], confidence: 0.98 },
  { label: "code-block",    bbox: [x, y, w, h], confidence: 0.87 },
]

比較(前フレーム, 現フレーム):
  - 新しいラベルが出現した   → APPEAR イベント
  - ラベルが消滅した         → DISAPPEAR イベント
  - bboxの位置が大きく変わった → MOVE イベント
  - ラベルは同じだが内容が変化した → CHANGE イベント
```

イベントが起きた時だけ重い処理（VLM解釈、STT、通知）を発火させる。

---

## アーキテクチャ

```
┌─────────────────────────────────────────────┐
│           Video Source                       │
│  (YouTube Live / Twitch / ローカル動画)      │
└──────────────┬──────────────────────────────┘
               │ フレーム
               ▼
┌─────────────────────────────────────────────┐
│           Object Detector                    │
│  - 軽量CNNモデル（YOLOv8 tiny 相当）         │
│  - 矩形 + ラベル + 信頼度を出力              │
│  - ターゲット: 10-30fps でリアルタイム動作   │
└──────────────┬──────────────────────────────┘
               │ 検出結果
               ▼
┌─────────────────────────────────────────────┐
│           State Tracker                      │
│  - フレーム間の状態変化を追跡                │
│  - ノイズ除去（n フレーム連続で確定）        │
│  - イベントキューに積む                      │
└──────────────┬──────────────────────────────┘
               │ イベント
               ▼
┌─────────────────────────────────────────────┐
│           Event Handler (プラガブル)          │
│  - VLM解釈（スライド内容を要約）             │
│  - STT（音声→テキスト変換）                 │
│  - 通知（Webhook / Discord / 任意）          │
│  - 録画トリガー                              │
└─────────────────────────────────────────────┘
```

---

## Object Detector の設計

### 入力
- 動画フレーム（RGB画像）
- ROI設定（オプション）: スクリーン全体 or 指定矩形内のみ処理

### 出力
```json
{
  "frame_id": 1234,
  "timestamp": 1710000000.5,
  "detections": [
    { "label": "slide-title", "bbox": [10, 20, 500, 60], "confidence": 0.95 },
    { "label": "page-number", "bbox": [750, 550, 50, 30], "confidence": 0.98 }
  ]
}
```

### モデルアーキテクチャ案
- **ベース**: YOLOv8-nano / MobileNetV3 + SSD ヘッド
- **入力サイズ**: 640×640（YOLOv8標準）
- **推論速度目標**: CPU で 10fps、GPU で 30fps 以上

### 学習データ
- 勉強会・技術発表動画（YouTube等）からフレーム抽出
- アノテーション対象ラベル:
  - `slide-title` — スライドのタイトルテキスト領域
  - `slide-body` — 本文コンテンツ領域
  - `page-number` — ページ番号
  - `code-block` — コードサンプル
  - `screen-area` — スクリーン/プロジェクター全体
  - `speaker` — 発表者（カメラ映像の場合）
- ラベリングツール: Label Studio / Roboflow

---

## State Tracker の設計

### 状態変化の定義

```python
class StateChange(Enum):
    APPEAR    = "appear"     # 新ラベルが出現
    DISAPPEAR = "disappear"  # ラベルが消滅
    MOVE      = "move"       # bbox が閾値以上移動
    CHANGE    = "change"     # 内容が変化（OCR差分など）
    STABLE    = "stable"     # 変化なし
```

### ノイズ除去
- `confidence_threshold`: 0.7 以上のみ採用
- `stability_frames`: n フレーム連続で同じ状態が続いたら確定（デフォルト: 3フレーム）
- アニメーション途中フレームは `STABLE` として無視

### スライド切り替えの判定例
```
条件: "page-number" ラベルの bbox 内容が変化した
  → CHANGE イベント発火
  → Event Handler: 現フレームをキャプチャして VLM に投げる
```

---

## Event Handler（プラガブル設計）

イベントタイプとハンドラーを設定ファイルで紐付け：

```yaml
handlers:
  - trigger:
      label: "slide-title"
      event: "CHANGE"
    actions:
      - type: "vlm_describe"
        model: "qwen-vl"
        prompt: "このスライドの内容を日本語で簡潔に要約してください"
      - type: "webhook"
        url: "https://discord.com/api/webhooks/..."
        
  - trigger:
      label: "screen-area"
      event: "APPEAR"
    actions:
      - type: "stt_start"
        model: "moonshine"
```

---

## 応用例

| ユースケース | 検知対象ラベル | 発火アクション |
|---|---|---|
| 勉強会ライブ配信 | slide-title, page-number | VLM要約 + STT文字起こし |
| ダッシュボード監視 | metric-value, alert-badge | 閾値超えで通知 |
| UI回帰テスト | button, modal, error-message | レイアウト崩れ検知 |
| 録画自動チャプター | slide-title | タイムスタンプ付きチャプター生成 |
| オンライン講義補助 | code-block | コードをクリップボードに取得 |

---

## 技術スタック案

| コンポーネント | 候補 |
|---|---|
| Object Detection | YOLOv8-nano (Ultralytics) / ONNX Runtime |
| 動画入力 | OpenCV / yt-dlp (YouTube) / ffmpeg |
| STT | Moonshine STT (既存サーバー活用) |
| VLM | qwen3-vl-30b (らぼみのエンドポイント活用) |
| 設定 | TOML / YAML |
| 実装言語 | Python (推論) or Rust (高速化後) |

---

## MVP スコープ（最小実装）

1. **フレームキャプチャ**: YouTube Live URL → ffmpeg でフレーム取得
2. **差分検知（pHash）**: まず軽量実装でスライド変化を検知
3. **VLM連携**: 変化検知時にスクリーンショットをVLMに投げて要約
4. **STT並走**: moonshine-stt にオーディオストリームを流す
5. **出力**: Discordウェブフック or テキストファイル

モデル学習は MVP 後のフェーズ2で実施。

---

## フェーズ2: カスタムモデル学習

- YouTube勉強会動画 100本分のデータ収集・アノテーション
- YOLOv8-nano でファインチューニング
- アニメーション判定の改善（「本当のスライド切り替え」vs「フェード中」）
- ONNX エクスポートで CPU/GPU 両対応
