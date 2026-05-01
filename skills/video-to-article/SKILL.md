---
name: video-to-article
description: Twitter/X の動画URLから記事を自動生成して4媒体に投稿する。「この動画まとめて」「記事にして」と言われたら使う。yt-dlpで音声DL→faster-whisperで文字起こし→ffmpegでスクショ抽出→記事執筆→zenn-postスキルで投稿。
argument-hint: [Twitter/X の動画URL]
allowed-tools: [Read, Edit, Write, Bash]
---

# video-to-article スキル

Twitter/X の動画URLを受け取り、記事を自動生成して Zenn・Qiita・dev.to・Hashnode の4媒体に投稿する。

## 依存ツール

以下がインストール済みであること。なければ自動インストールする。

```bash
which yt-dlp          # brew install yt-dlp
which ffmpeg          # brew install ffmpeg
python3 -c "from faster_whisper import WhisperModel"  # pip install faster-whisper
```

## 環境情報（各自書き換え）

- zenn-content リポジトリ: `/path/to/your/zenn-content`
- GitHub: `your-github-username/zenn-content`
- zenn-post スキルの .env: `~/.claude/plugins/zenn-post/.env`

---

## 実行フロー

### 0. 前提確認

ツールが揃っているか確認し、なければインストールする。

```bash
which yt-dlp || brew install yt-dlp
which ffmpeg || brew install ffmpeg
python3 -c "from faster_whisper import WhisperModel" 2>/dev/null || pip install faster-whisper --break-system-packages
```

### 1. 音声ダウンロード

```bash
# 音声のみ抽出（MP3・64kbps・16kHz・モノラル）
yt-dlp -x --audio-format mp3 --audio-quality 0 \
  --postprocessor-args "ffmpeg:-ar 16000 -ac 1 -ab 64k" \
  -o "/tmp/video-article-audio.%(ext)s" \
  "<URL>"
```

### 2. 動画ダウンロード（スクショ用）

```bash
yt-dlp -f "best[ext=mp4]" \
  -o "/tmp/video-article-full.%(ext)s" \
  "<URL>"
```

### 3. 文字起こし（faster-whisper・ローカル完結・APIキー不要）

```python
from faster_whisper import WhisperModel
import subprocess, json

model = WhisperModel("tiny", device="cpu", compute_type="int8")
audio = "/tmp/video-article-audio.mp3"
output = "/tmp/video-article-transcript.jsonl"

# 動画の長さを取得
result = subprocess.run(
    ["ffprobe", "-v", "quiet", "-show_entries", "format=duration",
     "-of", "default=noprint_wrappers=1:nokey=1", audio],
    capture_output=True, text=True
)
duration = float(result.stdout.strip())
chunk_size = 300  # 5分チャンク

num_chunks = int(duration // chunk_size) + 1

with open(output, "w") as out:
    for i in range(num_chunks):
        start = i * chunk_size
        ts = f"{int(start)//60}:{int(start)%60:02d}"

        subprocess.run([
            "ffmpeg", "-ss", str(start), "-t", str(chunk_size),
            "-i", audio, "/tmp/video-chunk.mp3", "-y"
        ], capture_output=True)

        segs, _ = model.transcribe("/tmp/video-chunk.mp3", beam_size=1)
        text = " ".join(s.text.strip() for s in segs)

        out.write(json.dumps({"ts": ts, "start": start, "text": text}, ensure_ascii=False) + "\n")
        print(f"[{ts}] {len(text)}文字")
```

### 4. スクリーンショット抽出

文字起こし内容を読んでから、重要シーンのタイムスタンプを判断してスクショを取る。

```bash
# 指定タイムスタンプから1枚取得
ffmpeg -ss <HH:MM:SS> -i /tmp/video-article-full.mp4 \
  -frames:v 1 /tmp/screenshot-<slug>-<番号>.jpg -y
```

目安：動画の長さに応じて3〜5枚。スライドや画面操作の重要シーンを優先する。

取得したスクショを zenn-content の `images/` にコピーして git push する。

```bash
cp /tmp/screenshot-<slug>-*.jpg "<zenn-contentのパス>/images/"
cd "<zenn-contentのパス>"
git add images/<slug>-*.jpg
git commit -m "feat: add images for <slug>"
git push origin main
```

### 5. 記事構成を組み立てる

文字起こし全文・スクショ内容をもとに以下の構成で記事を書く。

**frontmatter**

```yaml
---
title: "<タイトル（日本語）>"
emoji: "<絵文字>"
type: "tech"  # または "idea"
topics: ["topic1", "topic2", ...]  # 最大5つ・英小文字
published: false
---
```

**本文構成**

```
:::message
この記事は @<投稿者> さんによる以下の X ポストの動画（約XX分）を日本語でまとめたものです。
:::

@[tweet](<元のツイートURL>)

## はじめに
（動画・登壇者の紹介、何が学べるかを2〜3行で）

---

## <主要セクション1>
（スクショ + 解説）

## <主要セクション2>
...

## まとめ
（箇条書きまたは表で要点を整理）

---

## 📝 動画全文書き起こし（日本語訳）

（文字起こしを時系列で日本語に翻訳・整形したもの）
```

**画像パスは `/images/` 絶対パスで記述する**（`../images/` 相対パスはZennで表示されないため）

```markdown
![キャプション](/images/<slug>-01.jpg)
```

### 6. ユーザーに確認

記事の内容（タイトル・概要・本文）を見せて「このまま公開しますか？」と確認する。
**公開前に必ず確認を取ること。**

### 7. zenn-post スキルで投稿

OKが出たら **zenn-post スキルを呼び出して4媒体に投稿する**。

投稿先の構成は zenn-post スキルの仕様に従う：

| 言語 | 媒体 | 内容 |
|------|------|------|
| 日本語 | Zenn | オリジナル記事 |
| 日本語 | Qiita | Zenn 記法変換のみ |
| 英語 | dev.to | 英訳版 |
| 英語 | Hashnode | dev.to と同じ英訳（canonical は Zenn URL） |

「Zennだけ」「日本語だけ」など絞り込み指定があればそれに従う。

---

## 注意事項

- **スクショは動画ファイルから取得する**（音声ファイルからは取得不可）
- 文字起こしは英語音声でも日本語音声でも動作する（faster-whisper が自動検出）
- 動画が非常に長い場合（60分超）は tiny モデルのままで問題ないが、処理時間が増える
- Twitter/X の動画は認証が必要な場合があり、その場合は yt-dlp の cookies オプションが必要
- 記事の全文書き起こしセクションは**必ず日本語で書く**（英語音声でも日本語訳する）
