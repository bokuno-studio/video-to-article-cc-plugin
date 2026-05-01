# video-to-article — Claude Code Plugin

Twitter/X の動画URLを渡すだけで、記事を自動生成して **Zenn・Qiita（日本語）と dev.to・Hashnode（英訳）の4媒体に同時公開**します。

## できること

- Twitter/X の動画URLから音声を自動ダウンロード（yt-dlp）
- ローカルで文字起こし（faster-whisper・**APIキー不要・完全無料**）
- 重要シーンのスクリーンショット自動抽出（ffmpeg）
- 日本語まとめ記事 + 全文書き起こし（日本語訳）を自動執筆
- zenn-post スキルと連携して4媒体に同時投稿

## 使い方

```
「この動画まとめて」
「記事にして → <Twitter/X URL>」
「Zennだけでいい → <URL>」
```

## インストール

### 前提

- [Claude Code](https://claude.ai/code) がインストール済み
- [zenn-post プラグイン](https://github.com/bokuno-studio/zenn-post-cc-plugin) がインストール済み

### 依存ツール

```bash
brew install yt-dlp ffmpeg
pip install faster-whisper
```

### 手順

**1. このリポジトリをクローン**

```bash
git clone https://github.com/bokuno-studio/video-to-article-cc-plugin ~/.claude/plugins/video-to-article
```

**2. SKILL.md の環境情報を書き換える**

`skills/video-to-article/SKILL.md` の「環境情報」セクションを自分の環境に合わせて編集する。

**3. Claude Code のプラグインマーケットプレイスに登録**

```bash
claude plugin marketplace add ~/.claude/plugins/video-to-article
claude plugin install video-to-article
```

## 仕組み

```
Twitter/X URL
  ↓ yt-dlp（音声抽出・MP3）
  ↓ yt-dlp（動画DL・スクショ用）
  ↓ faster-whisper tiny（文字起こし・5分チャンク）
  ↓ ffmpeg（スクリーンショット抽出）
  ↓ Claude（日本語まとめ記事執筆）
  ↓ zenn-post スキル（4媒体投稿）
```

## 文字起こしについて

`faster-whisper` の `tiny` モデルをローカルで実行するため、**OpenAI API キーは不要**です。

| 動画時間 | 処理時間の目安（M1 Mac） |
|---------|----------------------|
| 〜30分   | 約2〜3分 |
| 〜60分   | 約5〜6分 |
| 〜120分  | 約10〜12分 |

## ライセンス

MIT
