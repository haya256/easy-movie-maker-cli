# 動画自動生成ワークフロー

このプロジェクトでは `data/video.md`（動画編集指示書）を読み取り、FFmpegを使って動画を自動生成する。

---

## 基本的な使い方

ユーザーが「video.md通りに動画を作って」と言ったら、以下の手順で実行すること。

### 手順

1. `data/video.md` を読み込む
2. シーン構成・字幕・BGM・スタイル設定をパースする
3. 各シーンの字幕テキストを一時的な `.srt` または `.ass` ファイルとして生成する
4. FFmpegコマンドを組み立てて実行する
5. `data/output/` に完成動画を出力する

---

## data/video.md のフォーマット仕様

### シーン構成

各シーンは以下の形式で記述される：

````markdown
### シーン N
- **画像**: data/img/ファイル名.png
- **表示時間**: X秒
- **字幕**:
  ```
  ここに字幕テキストを直接記述
  複数行も可
  ```
````

### BGM

| ファイル | 音量（0.0〜1.0） | フェードイン | フェードアウト |
|--------|--------------|-----------|-------------|

### 字幕スタイル

フォント・サイズ・色・縁取り・位置が指定される。
未指定の場合はデフォルト値（後述）を使うこと。

---

## FFmpeg実装ガイド

### 基本構成（画像→動画化→結合→BGM合成→字幕焼き込み）

```bash
# ステップ1: 各シーンを動画化（字幕なし）
# アスペクト比を保ったまま1920x1080にフィット。余白は黒帯でパディング
ffmpeg -loop 1 -i data/img/1.png -t 10 \
  -vf "scale=1920:1080:force_original_aspect_ratio=decrease,pad=1920:1080:(ow-iw)/2:(oh-ih)/2:black" \
  -r 30 -pix_fmt yuv420p scene1.mp4

# ステップ2: 動画を結合
ffmpeg -f concat -safe 0 -i concat_list.txt -c copy combined.mp4

# ステップ3: BGMを合成
# 画像素材から生成した動画には音声トラックがないため、[0:a]は使わず[bgm]を直接mapする
ffmpeg -i combined.mp4 -i data/bgm/b.mp3 \
  -filter_complex "[1:a]volume=0.3,afade=t=in:d=1,afade=t=out:st=37:d=3[bgm]" \
  -map 0:v -map "[bgm]" -c:v copy -shortest with_bgm.mp4

# ステップ4: 字幕を焼き込む
# assフィルタはファイルパスを絶対パスで指定すること（相対パスだとfopen失敗する場合がある）
ffmpeg -i /絶対パス/tmp/with_bgm.mp4 -vf "ass=/絶対パス/tmp/subtitles.ass" -c:a copy /絶対パス/output/output.mp4
```

### 字幕ファイル（ASS形式）の生成

字幕は各シーンの `字幕:` ブロックから読み取り、タイムコードを計算して `.ass` ファイルを生成する。

```
[Script Info]
ScriptType: v4.00+
PlayResX: 1920
PlayResY: 1080

[V4+ Styles]
Format: Name, Fontname, Fontsize, PrimaryColour, OutlineColour, BorderStyle, Outline, Shadow, Alignment, MarginL, MarginR, MarginV, Encoding
Style: Default,Noto Sans CJK JP,48,&H00FFFFFF,&H00000000,1,3,0,2,10,10,180,1

[Events]
Format: Layer, Start, End, Style, Text
Dialogue: 0,0:00:00.00,0:00:10.00,Default,最初のシーンのテキスト
Dialogue: 0,0:00:10.00,0:00:20.00,Default,2つ目のシーン
```

- フォント名は `fc-list | grep -i "noto sans cjk jp"` で確認した `Noto Sans CJK JP` を使うこと（`NotoSansCJK-Regular` ではない）
- 複数行字幕は `\N` で改行する
- MarginVはビデオ下端からのピクセル数。`y=950` の場合 `MarginV=1080-950=130`

---

## デフォルト値

| 設定 | デフォルト |
|-----|---------|
| 解像度 | 1920x1080 |
| フレームレート | 30fps |
| フォント | NotoSansCJK-Regular |
| フォントサイズ | 48 |
| 文字色 | white（&H00FFFFFF） |
| 縁取り色 | black（&H00000000） |
| 縁取り幅 | 3 |
| 字幕位置 | 下部中央 |
| BGM音量 | 0.3 |

---

## フォルダ構成

```
project/
├── CLAUDE.md              ← このファイル
├── video.md.example       ← 編集指示書のサンプル（参照用）
├── data/                  ← ユーザーデータ（.gitignore対象）
│   ├── video.md           ← 編集指示書（ユーザーが編集する）
│   ├── img/               ← 画像素材
│   ├── bgm/               ← BGMファイル
│   ├── output/            ← 完成動画の出力先
│   └── tmp/               ← 作業用一時ファイル（処理後に削除してよい）
```

---

## 注意事項

- `tmp/` は処理後に削除すること
- FFmpegが未インストールの場合は `sudo apt install ffmpeg` を先に実行すること
- 日本語フォントが見つからない場合は `fc-list | grep -i "noto sans cjk jp"` で確認し、存在するフォント名に置き換えること
- `data/output/` ディレクトリが存在しない場合は自動作成すること
- 画像のアスペクト比は必ず保持すること。`scale=幅:高さ` で直接リサイズすると縦長画像が横に引き伸ばされる
- assフィルタに渡すファイルパスは必ず絶対パスを使うこと。相対パスだとfopen失敗する場合がある
- assファイルの作成にはWrite toolではなくbashのheredocを使うこと（Write toolで作成したファイルが見つからないケースがある）
- BGMのamixフィルタで `[0:a]` を使わないこと。画像から生成した動画には音声トラックがないためエラーになる
