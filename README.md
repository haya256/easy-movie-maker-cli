# easy-movie-maker-cli

`video.md`（動画編集指示書）を書くだけで、Claude + FFmpeg が自動で動画を生成してくれる CLI ワークフローです。

## 特徴

- マークダウンで動画構成を宣言的に記述
- 画像・BGM・字幕スタイルをすべて `video.md` 一枚で管理
- Claude Code に「動画を作って」と伝えるだけで完成動画を出力

## 必要環境

- [Claude Code](https://claude.ai/code)
- FFmpeg（`brew install ffmpeg`）
- 日本語フォント（Hiragino Sans など）

## 使い方

1. `data/video.md` を編集して動画構成を記述する
2. `data/img/` に画像素材を配置する
3. `data/bgm/` に BGM ファイルを配置する
4. Claude Code で「video.md 通りに動画を作って」と伝える
5. `data/output/output.mp4` に完成動画が出力される

## video.md の書き方

`video.md.example` を参考に編集してください。

```markdown
### シーン 1
- **画像**: img/1.jpg
- **表示時間**: 10秒
- **字幕**:
  ```
  ここに字幕テキストを書く
  複数行も可
  ```
```

BGM・字幕スタイル（フォント・サイズ・色・位置）もマークダウンで指定できます。

## フォルダ構成

```
easy-movie-maker-cli/
├── CLAUDE.md           ← Claude への動作指示（FFmpegワークフロー定義）
├── video.md.example    ← 編集指示書のサンプル
└── data/               ← ユーザーデータ（.gitignore対象）
    ├── video.md        ← 動画編集指示書
    ├── img/            ← 画像素材
    ├── bgm/            ← BGMファイル
    ├── output/         ← 完成動画の出力先
    └── tmp/            ← 作業用一時ファイル（自動削除）
```

## ライセンス

MIT
