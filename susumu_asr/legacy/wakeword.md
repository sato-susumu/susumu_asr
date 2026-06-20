# ウェイクワード検出（廃止予定 / legacy）

> このドキュメントは廃止予定のウェイクワード検出機能に関する記述を `AGENTS.md` から移したものです。
> 現行の標準構成は `passthrough`（ウェイクワードなし常時認識）です。

## 主要クラス（ウェイクワード検出）

| クラス | ファイル | 役割 |
|--------|----------|------|
| `LivekitWakewordPlugin` | `wakeword_livekit.py` | livekit-wakeword（ONNX）でウェイクワード検出 |
| `OpenWakewordPlugin` | `wakeword_openwakeword.py` | OpenWakeWord（tflite）でウェイクワード検出 |

## プラグイン組み合わせ（ウェイクワード検出）

| vad_plugin | wakeword_plugin | asr_plugin | 用途 |
|---|---|---|---|
| `silero_vad` | `livekit_wakeword` | `google_cloud` / `whisper` / `amivoice` | livekit-wakewordでウェイクワード検出 |
| `silero_vad` | `openwakeword` | `google_cloud` / `whisper` / `amivoice` | OpenWakeWordでウェイクワード検出 |

## livekit-wakeword のインストール

`livekit-wakeword` は `Requires-Python: >=3.11` と宣言されているが、推論に使う部分は Python 3.10 でも動作する（pure Python wheel）。ROS2 Humble（Python 3.10）へのインストールは以下で行う：

```bash
pip install livekit-wakeword --ignore-requires-python
```

`setup.py` の `install_requires` には含めない（通常の `pip install` でバージョン制約エラーになるため）。

## ウェイクワードモデル

`models/` ディレクトリに ONNX 形式で配置。デフォルトは `models/hey_mycroft_v0.1.onnx`。
利用可能モデル: `alexa`, `hey_jarvis`, `hey_mycroft`, `hey_rhasspy`, `timer`, `weather`。
モデルが存在しない場合は起動時に openWakeWord の GitHub リリース（v0.5.1）から自動ダウンロードされる。
livekit-wakeword と openWakeWord は同じ embedding モデル（Google Speech Embedding）を使うため、openWakeWord 形式の ONNX モデルをそのまま livekit-wakeword で使用できる。
