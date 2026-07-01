# hermes-x-search 実装ドキュメント

行動規約（トリガー→アクション・絶対にしないこと）は `CLAUDE.md` を参照。ここには実装詳細（稼働構成・ファイル格納・設定値・コマンド例）を置く。

---

## 稼働構成

```
Claude Code
  └─ Bash: hermes -z "<プロンプト>" [-t x_search]
       └─ Hermes Agent v0.14.0
            ├─ モデル: grok-4.3（xai-oauth、X Premium OAuthで認証済み）
            ├─ X検索: x_search ツール（SuperGrok枠を消費、API課金なし）
            ├─ 画像生成: Grok Imagine（xAI OAuth経由）
            ├─ 音声生成: Grok TTS（provider: xai、voice: eve）
            └─ 動画生成: Grok Imagine 動画モード
```

**スクリプトショートカット:**
```bash
hermes-x "<クエリ>"   # X検索5件を「タイトル/要約/URL」形式で返す
~/.local/bin/hermes-x  # symlink先: このディレクトリの hermes-x スクリプト
```

---

## ファイル格納ルール

生成物は必ず以下に保存する。中間ファイルや `/tmp` への放置は禁止。

```
~/ClaudeCode/projects/常時運用/hermes-x-search/
├── assets/
│   └── YYYY-MM-DD/          # 日付ごとに格納
│       ├── *.jpg / *.png    # 生成画像
│       ├── *-video.mp4      # 生成動画（音声なし）
│       ├── *-final.mp4      # 動画＋音声合成済み（X投稿用）
│       └── *.ogg / *.mp3    # 生成音声
├── drafts/
│   └── YYYY-MM-DD_*.md      # 投稿下書き（テキスト）
├── hermes-x                 # X検索スクリプト本体
└── CLAUDE.md                # 行動規約
```

---

## hermes 設定ファイル

| ファイル | 内容 |
|---|---|
| `~/.hermes/config.yaml` | プロバイダ・モデル・ツール設定。`tts.provider: xai` 設定済み |
| `~/.hermes/auth.json` | xAI OAuth トークン（コミット・出力禁止） |

**主要設定値（変更禁止）:**
- `model.provider: xai-oauth`
- `model.default: grok-4.3`
- `tts.provider: xai`（Grok TTS 有効）
- `browser.camofox.managed_persistence: true`（セッション永続化）
- `browser.camofox.user_id: fuumanpnp`

---

## よく使うコマンド

```bash
# X検索（基本）
hermes-x "Claude Code 最新情報"

# X検索（件数・フォーマット指定）
hermes -z "Xで「OpenAI」の最新投稿を10件返して。タイトル/要約/URL形式で" -t x_search

# 画像生成
hermes -z "image_generateツールでサイバーパンク風AIエージェントの16:9画像を生成して、パスを返して"

# TTS音声生成
hermes -z "text_to_speechで「本文」を日本語音声に変換して /Users/kawamurafuushin/ClaudeCode/projects/常時運用/hermes-x-search/assets/YYYY-MM-DD/voice.ogg に保存して"

# 動画＋音声合成（ffmpeg）
ffmpeg -i <video.mp4> -i <voice.ogg> \
  -filter_complex "[0:v]setpts=PTS*(音声秒数/動画秒数)[v]" \
  -map "[v]" -map 1:a -c:v libx264 -c:a aac -shortest <final.mp4> -y

# Hermesバージョン確認・アップデート
hermes --version
hermes update
```
