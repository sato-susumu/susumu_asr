# `monitor_demo.gif` の作り方

README 冒頭に表示している ASR モニターのデモ GIF（[`monitor_demo.gif`](monitor_demo.gif)）の作成手順をまとめる。

## 概要

`asr_monitor_node` のデバッグウィンドウを `ffmpeg` の `x11grab` で画面録画し、`ffmpeg` のパレット最適化で GIF 化する。マイク発話の代わりに、複数発話を含むテスト用 WAV（`test/audio/multi_utterance_42s.wav`）を入力に使うことで、再現性高く録画できる。

> 録画には X11 セッション（`XDG_SESSION_TYPE=x11`）と AmiVoice の API キー（`.env` の `AMIVOICE_APP_KEY`）が必要。Wayland の場合は `wf-recorder` 等の別ツールに置き換える。

## 前提

- ビルド済みであること（colcon はコピー型インストールのため、ソース編集後は要ビルド）

```bash
cd ~/ros2_ws
colcon build --packages-select susumu_asr
source install/setup.bash
```

- `ffmpeg`、`wmctrl`、`xwininfo` がインストール済みであること

## 1. デバッグウィンドウを録画する

WAV 版デバッグ launch（`amivoice_debug_wav.launch.py`。マイク版 `amivoice_debug.launch.py` と同じモニター GUI・同じ AmiVoice 認識）を起動し、`ASR Monitor` ウィンドウを録画する。

以下のスクリプトは「launch 起動 → ウィンドウ出現待ち → 固定位置へ移動・最前面化 → `ffmpeg` で 45 秒録画 → 後片付け」を一括で行う。

```bash
#!/bin/bash
export DISPLAY=:0
cd ~/ros2_ws
source /opt/ros/humble/setup.bash
source install/setup.bash

OUT=/tmp/asr_demo.mp4
rm -f "$OUT"

# launch をバックグラウンド起動（WAV を頭から処理）
ros2 launch susumu_asr amivoice_debug_wav.launch.py > /tmp/asr_demo_launch.log 2>&1 &
LAUNCH_PID=$!

# ウィンドウ出現待ち（wmctrl で hex ID を取得）
WID=""
for i in $(seq 1 80); do
  WID=$(wmctrl -l | grep "ASR Monitor" | awk '{print $1}' | head -1)
  [ -n "$WID" ] && break
  sleep 0.3
done
[ -z "$WID" ] && { echo "NO WINDOW"; kill -9 $LAUNCH_PID; exit 1; }

# 固定位置へ移動・最前面化
wmctrl -i -r "$WID" -e 0,40,60,1200,600
wmctrl -i -a "$WID"
sleep 0.5

# 座標取得
INFO=$(xwininfo -id "$WID")
X=$(echo "$INFO" | awk '/Absolute upper-left X/{print $4}')
Y=$(echo "$INFO" | awk '/Absolute upper-left Y/{print $4}')
WIDTH=$(echo "$INFO" | awk '/Width:/{print $2}')
HEIGHT=$(echo "$INFO" | awk '/Height:/{print $2}')
W=$((WIDTH - WIDTH%2)); H=$((HEIGHT - HEIGHT%2))  # 偶数化

# 録画（45 秒：WAV 42 秒 + 余裕）
ffmpeg -y -f x11grab -framerate 20 -video_size ${W}x${H} -i :0.0+${X},${Y} \
  -t 45 -c:v libx264 -preset ultrafast -pix_fmt yuv420p "$OUT"

# 後片付け
for p in $(ps aux | grep -E "asr_monitor_node|susumu_asr_node|ros2 launch susumu" | grep -v grep | awk '{print $2}'); do
  kill -9 $p 2>/dev/null
done
```

録画結果は `/tmp/asr_demo.mp4`（1200×600）に出力される。

## 2. GIF に変換する

`ffmpeg` のパレット生成方式（`palettegen` → `paletteuse`）で高画質・低サイズに変換する。波形が毎フレーム動くため GIF は圧縮が効きにくく、**fps・幅・色数・尺**でサイズを調整する。

現在の `monitor_demo.gif` の設定：

- **尺**: 元動画の 5〜32 秒（27 秒）。冒頭 5 秒の無音待機をカットし、全発話が収まる範囲
- **サイズ**: 幅 900px（高さは比率維持で 450px）
- **fps**: 8
- **色数**: 96

```bash
cd /tmp

# パレット生成（-ss 5 で先頭 5 秒カット、-t 27 で 27 秒分）
ffmpeg -y -ss 5 -t 27 -i asr_demo.mp4 \
  -vf "fps=8,scale=900:-1:flags=lanczos,palettegen=max_colors=96:stats_mode=diff" \
  palette.png

# パレット適用して GIF 化
ffmpeg -y -ss 5 -t 27 -i asr_demo.mp4 -i palette.png \
  -lavfi "fps=8,scale=900:-1:flags=lanczos[x];[x][1:v]paletteuse=dither=bayer:bayer_scale=5:diff_mode=rectangle" \
  asr_demo.gif

# docs へ配置
cp asr_demo.gif ~/ros2_ws/src/susumu_asr/docs/monitor_demo.gif
```

### サイズ調整の目安

| パラメータ | 効果 |
|---|---|
| `scale=900:-1` の `900` | 幅。大きくすると鮮明だがサイズ増 |
| `fps=8` | フレームレート。下げるとサイズ減（カクつく） |
| `palettegen=max_colors=96` | 色数。減らすとサイズ減（階調が荒くなる） |
| `-ss` / `-t` | 切り出し開始秒 / 長さ。短くするとサイズ減 |

## 確認

GIF から 1 フレーム抜き出して目視チェックできる。

```bash
ffmpeg -y -ss 10 -i ~/ros2_ws/src/susumu_asr/docs/monitor_demo.gif -frames:v 1 /tmp/check.png
ffprobe -v error -show_entries stream=width,height -of csv=p=0 ~/ros2_ws/src/susumu_asr/docs/monitor_demo.gif
```
