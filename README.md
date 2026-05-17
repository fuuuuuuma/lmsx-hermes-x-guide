# LMSX（hermes-x）完全セットアップガイド

> **対象読者:** このリポジトリを受け取ったAIエージェント、または新環境にLMSXを導入する開発者  
> **目的:** Xの最新アルゴリズムを活かしたリサーチ → 記事・画像・音声生成までを、AIエージェントが自律的にセットアップ・運用できる状態にする  
> **最終更新:** 2026-05-17

---

## 目次

1. [LMSXとは](#lmsxとは)
2. [Xの新アルゴリズム（2026年）](#xの新アルゴリズム2026年)
3. [システム構成](#システム構成)
4. [セットアップ手順](#セットアップ手順)
5. [ワークフロー](#ワークフロー)
6. [ファイル構造](#ファイル構造)
7. [運用ルール](#運用ルール)
8. [よく使うコマンド集](#よく使うコマンド集)
9. [トラブルシューティング](#トラブルシューティング)

---

## LMSXとは

**LMSX（Language Model × X Search）** は、Hermes Agent（NousResearch）と Grok（xAI）のX OAuth認証を組み合わせて、X（旧Twitter）の検索・リサーチ・コンテンツ生成を自動化するシステム。

**できること:**
- X上のリアルタイム情報をAIが検索・要約
- X投稿文（ツイート・スレッド）の自動生成
- Grok Imagineで画像生成（16:9 / 縦型対応）
- Grok TTSで日本語音声生成
- 画像＋音声をffmpegで合成してX投稿用動画を作成

**できないこと（設計上の制約）:**
- Xへの自動投稿（BAN対策で意図的に無効化）
- Xの下書き自動保存（Hermesのブラウザ使い捨て構造上不可）

---

## Xの新アルゴリズム（2026年）

### コアランキングロジック

Xは2025〜2026年にかけてアルゴリズムを大幅刷新。以下が確認されている主要変更点。

| 指標 | 重み | 説明 |
|------|------|------|
| **エンゲージメント率** | 最重要 | いいね・RT・返信・ブックマーク（特にブックマーク高評価） |
| **完読率（動画）** | 高 | 動画は70%以上の完了率がリーチに直結 |
| **投稿後60分の初速** | 高 | 投稿直後の反応速度がランク決定に強く影響 |
| **X Premium バッジ** | 中 | 認証バッジ付きアカウントの返信が優先表示 |
| **Grok / AIラベル** | 新設 | AI生成コンテンツのラベリング開始（透明性要件） |

### 2026年の勝ちパターン（AI収益化ラボ実測値）

```
AIが主語 × 具体的ツール名 × 数字入りタイトル = 再生数10倍差
```

- 「AIが〜してくれた」形式がエンゲージメント率トップ
- ツール紹介型（「〇〇とは」形式）は負ける傾向
- 冒頭3秒フック必須：問題提起 → 解決策の順
- 動画は完了率70%超が発見タブ流入のしきい値

### SuperGrok枠の活用

X PremiumサブスクリプションにはSuperGrokの利用枠が含まれており、これを使ったX検索はAPI課金が発生しない。LMSXはこの枠を活用する設計。

---

## システム構成

```
ユーザー / Claude Code
    │
    ├─ hermes-x "クエリ"
    │       └─ Hermes Agent v0.14.0
    │               ├─ モデル: grok-4.3（xai-oauth）
    │               ├─ x_search ツール（SuperGrok枠）
    │               ├─ image_generate（Grok Imagine）
    │               ├─ text_to_speech（Grok TTS / voice: eve）
    │               └─ 動画生成モード
    │
    ├─ assets/YYYY-MM-DD/   ← 生成物を格納
    └─ drafts/YYYY-MM-DD_*.md  ← 投稿下書き
```

**依存関係:**
- Hermes Agent v0.14.0+
- Python 3.11+
- ffmpeg（音声合成・動画処理）
- X Premium サブスクリプション（SuperGrok枠のため）

---

## セットアップ手順

> AIエージェントはこのセクションを上から順に実行すること。

### Step 1: Hermesのインストール

```bash
# Hermesの公式インストール方法（要確認: https://github.com/NousResearch/hermes-agent）
pip install hermes-agent
# または
pipx install hermes-agent

# バージョン確認
hermes --version
# → Hermes Agent v0.14.0 (2026.5.16) 以上であること
```

### Step 2: xAI OAuth認証の設定

```bash
# xAI OAuthでログイン（ブラウザが開いてX認証を求められる）
hermes auth xai-oauth

# 認証後、トークンは ~/.hermes/auth.json に保存される
# このファイルは絶対にgitにコミットしない
```

**前提条件:** X PremiumまたはX Premium+のサブスクリプションが必要（SuperGrok枠のため）

### Step 3: Hermes設定ファイルの作成

```bash
mkdir -p ~/.hermes
```

`~/.hermes/config.yaml` を以下の内容で作成:

```yaml
model:
  default: grok-4.3
  base_url: https://api.x.ai/v1
  provider: xai-oauth

agent:
  max_turns: 60
  gateway_timeout: 1800
  tool_use_enforcement: auto
  reasoning_effort: medium

tts:
  provider: xai
  voice: eve

browser:
  camofox:
    managed_persistence: true
    user_id: <YOUR_USER_ID>   # 任意のID文字列
```

### Step 4: X検索ツールの有効化

```bash
# toolsets の設定確認
hermes config list

# x_search ツールが有効か確認
hermes -z "test" -t x_search
# → エラーなく実行されれば成功
```

### Step 5: このリポジトリのクローンと配置

```bash
git clone https://github.com/fuuuuuuma/lmsx-hermes-x-guide.git \
  ~/ClaudeCode/projects/常時運用/hermes-x-search

cd ~/ClaudeCode/projects/常時運用/hermes-x-search

# hermes-x スクリプトに実行権限を付与
chmod +x hermes-x
```

### Step 6: symlink の作成（パスを通す）

```bash
mkdir -p ~/.local/bin
ln -sf ~/ClaudeCode/projects/常時運用/hermes-x-search/hermes-x ~/.local/bin/hermes-x

# PATHに ~/.local/bin が含まれているか確認
echo $PATH | grep -q ".local/bin" && echo "OK" || echo "~/.local/bin をPATHに追加してください"
```

`~/.zshrc` または `~/.bashrc` に以下を追加（必要な場合）:
```bash
export PATH="$HOME/.local/bin:$PATH"
```

### Step 7: ffmpegのインストール（動画生成用）

```bash
# macOS
brew install ffmpeg

# Ubuntu/Debian
sudo apt-get install ffmpeg

# 確認
ffmpeg -version
```

### Step 8: 動作確認

```bash
# X検索テスト
hermes-x "Claude Code 最新情報"
# → 5件のタイトル/要約/URL が返ってくれば成功

# 画像生成テスト
hermes -z "image_generateツールで青空の画像を生成してパスを返して"

# TTS テスト
hermes -z "text_to_speechで「テストです」を音声に変換して /tmp/test.ogg に保存して"
```

---

## ワークフロー

### ワークフロー① リサーチ（基本）

```bash
# 基本形（5件返却）
hermes-x "調べたいトピック"

# 件数・フォーマット指定
hermes -z "Xで「生成AI」の最新投稿を10件返して。タイトル/要約/URL形式" -t x_search

# 特定アカウントの投稿を検索
hermes -z "Xで @openai の最新ツイートを5件まとめて" -t x_search
```

**出力形式:**
```
1. タイトル / 1文要約 / https://x.com/...
2. タイトル / 1文要約 / https://x.com/...
...
```

### ワークフロー② X記事・投稿文の作成

```bash
# Step 1: リサーチ
hermes-x "Claude Code 2026年の新機能"

# Step 2: 投稿文生成（Claude Code 内で実行）
# 取得した情報をもとに drafts/ に保存

# Step 3: ファイル保存
# drafts/YYYY-MM-DD_topic.md に保存
```

**投稿文テンプレート（X向け）:**
```
[フック] 冒頭3秒で問題提起

[本文] 解決策 + 具体的な数字/ツール名

[CTA] 詳細はnoteで → [URL]

[ハッシュタグ] #Claude #AI #生成AI
```

**Xアルゴリズム対応チェックリスト:**
- [ ] 冒頭に数字または固有名詞を含む
- [ ] 「AIが主語」の構文を使う（例: 「AIが〇〇した」）
- [ ] ブックマークを促す内容（保存したくなるTips）
- [ ] 動画付きの場合: 冒頭3秒にフックを入れる

### ワークフロー③ 画像生成

```bash
# 基本的な画像生成
hermes -z "image_generateツールで次の画像を生成してパスを返して: [プロンプト]"

# X用サムネイル（16:9）
hermes -z "image_generateツールで16:9のサムネイル画像を生成してください。テーマ: [内容]。テキストは不要。パスを返して"

# 生成後、assets/ に移動
DATE=$(date +%Y-%m-%d)
mkdir -p ~/ClaudeCode/projects/常時運用/hermes-x-search/assets/$DATE
mv /tmp/generated_image.jpg ~/ClaudeCode/projects/常時運用/hermes-x-search/assets/$DATE/
```

**画像プロンプトのコツ（X向け）:**
- 縦型（9:16）= モバイルフィード最適化
- 横型（16:9）= PC表示・サムネイル用
- テキストは画像に含めない（X側でOCRされて干渉する場合あり）

### ワークフロー④ 音声生成（TTS）

```bash
DATE=$(date +%Y-%m-%d)
mkdir -p ~/ClaudeCode/projects/常時運用/hermes-x-search/assets/$DATE

hermes -z "text_to_speechツールで次のテキストを音声に変換して \
  ~/ClaudeCode/projects/常時運用/hermes-x-search/assets/${DATE}/voice.ogg に保存して: \
  [読み上げテキスト]"
```

**設定値:**
- provider: `xai`
- voice: `eve`（日本語対応、自然な発音）
- 出力形式: OGG（Vorbis）

### ワークフロー⑤ 動画＋音声合成

```bash
DATE=$(date +%Y-%m-%d)
ASSETS=~/ClaudeCode/projects/常時運用/hermes-x-search/assets/$DATE

# 音声の長さを取得
AUDIO_DURATION=$(ffprobe -v error -show_entries format=duration \
  -of default=noprint_wrappers=1:nokey=1 "${ASSETS}/voice.ogg")

# 動画の長さを取得
VIDEO_DURATION=$(ffprobe -v error -show_entries format=duration \
  -of default=noprint_wrappers=1:nokey=1 "${ASSETS}/video.mp4")

# 合成（音声の長さに動画を合わせる）
ffmpeg -i "${ASSETS}/video.mp4" -i "${ASSETS}/voice.ogg" \
  -filter_complex "[0:v]setpts=PTS*(${AUDIO_DURATION}/${VIDEO_DURATION})[v]" \
  -map "[v]" -map 1:a \
  -c:v libx264 -c:a aac -shortest \
  "${ASSETS}/output-final.mp4" -y

echo "完成: ${ASSETS}/output-final.mp4"
```

### ワークフロー⑥ 完全自動パイプライン（全工程）

```bash
#!/usr/bin/env bash
# lmsx-pipeline.sh: リサーチ → 投稿文 → 画像 → 音声 → 動画 の完全パイプライン

TOPIC="$1"  # 例: "Claude Code 最新機能"
DATE=$(date +%Y-%m-%d)
ASSETS=~/ClaudeCode/projects/常時運用/hermes-x-search/assets/$DATE
DRAFTS=~/ClaudeCode/projects/常時運用/hermes-x-search/drafts

mkdir -p "$ASSETS" "$DRAFTS"

echo "=== Step 1: X検索リサーチ ==="
hermes-x "$TOPIC" | tee "$DRAFTS/${DATE}_research.txt"

echo "=== Step 2: 投稿文を drafts/ に保存 ==="
# Claude Code でこの部分を人間が確認・編集する

echo "=== Step 3: 画像生成 ==="
hermes -z "image_generateツールで「${TOPIC}」をテーマにした16:9のXサムネイル画像を生成してパスを返して" \
  | grep -oE '/[^ ]+\.(jpg|png|webp)' | head -1 | xargs -I{} mv {} "$ASSETS/thumbnail.jpg"

echo "=== Step 4: 音声生成 ==="
# (投稿文が確定した後に実行)

echo "Done. 確認: $ASSETS"
```

---

## ファイル構造

```
~/ClaudeCode/projects/常時運用/hermes-x-search/
├── README.md              # このファイル（AIエージェント向けセットアップガイド）
├── CLAUDE.md              # Claude Code 向け運用ルール
├── hermes-x               # X検索スクリプト本体（実行権限付き）
├── .env                   # 環境変数（gitignore必須・認証情報なし）
├── .gitignore             # auth.json等を除外
├── assets/
│   └── YYYY-MM-DD/
│       ├── thumbnail.jpg  # 生成画像（X投稿用）
│       ├── voice.ogg      # 生成音声
│       ├── video.mp4      # 生成動画（音声なし）
│       └── output-final.mp4  # 動画＋音声合成済み（X投稿用）
└── drafts/
    └── YYYY-MM-DD_topic.md   # X投稿下書き
```

**重要なパス:**
| パス | 用途 |
|------|------|
| `~/.hermes/config.yaml` | Hermes設定（モデル・プロバイダ・TTS） |
| `~/.hermes/auth.json` | xAI OAuthトークン（**絶対にgitにコミットしない**） |
| `~/.local/bin/hermes-x` | symlink（PATHから直接呼べる） |

---

## 運用ルール

### やること

- リサーチ結果は必ず `drafts/` に保存してからコンテンツ化する
- 生成ファイルは必ず `assets/YYYY-MM-DD/` に格納する（`/tmp` 放置禁止）
- Xへの投稿判断はユーザーが行う

### やってはいけないこと

| 禁止事項 | 理由 |
|----------|------|
| Xへの自動投稿 | BAN対策（利用規約違反リスク） |
| `~/.hermes/auth.json` をログ・会話に出力 | OAuthトークン漏洩リスク |
| computer_use / Claude in Chrome でXを操作 | BAN対策 |
| 下書き保存の自動化を試みる | Hermes `-z` はブラウザ使い捨てでXログイン不可 |
| 生成ファイルを `/tmp` に放置 | 中間ファイルの散乱防止 |

### 投稿フローの権限分担

| Claude Code / AIエージェントが担当 | ユーザーが担当 |
|-----------------------------------|----------------|
| X検索・情報収集 | X投稿画面への貼り付け |
| 投稿文案の生成 | 動画のドラッグ&ドロップ |
| 画像・音声・動画の生成 | 「下書き」ボタンのクリック |
| `assets/` への格納 | 投稿するかの最終判断 |

---

## よく使うコマンド集

```bash
# === X検索 ===
hermes-x "検索したいトピック"
hermes -z "Xで「キーワード」の投稿を10件返して。タイトル/要約/URL形式" -t x_search

# === 画像生成 ===
hermes -z "image_generateツールで[プロンプト]を生成してパスを返して"

# === TTS音声生成 ===
hermes -z "text_to_speechで「テキスト」をeve音声で /path/to/output.ogg に保存して"

# === 動画生成 ===
hermes -z "image_generateツールで動画モードを使って[プロンプト]の動画を生成してパスを返して"

# === 動画＋音声合成 ===
ffmpeg -i video.mp4 -i voice.ogg \
  -filter_complex "[0:v]setpts=PTS*(音声秒数/動画秒数)[v]" \
  -map "[v]" -map 1:a -c:v libx264 -c:a aac -shortest final.mp4 -y

# === Hermes管理 ===
hermes --version      # バージョン確認
hermes update         # アップデート
hermes auth xai-oauth # xAI再認証
hermes config list    # 設定確認
```

---

## トラブルシューティング

### `hermes: command not found`

```bash
# hermesが入っているか確認
pip show hermes-agent || pipx list | grep hermes

# PATHを確認
echo $PATH

# 再インストール
pip install hermes-agent
```

### `hermes-x: command not found`

```bash
# symlinkを再作成
ln -sf ~/ClaudeCode/projects/常時運用/hermes-x-search/hermes-x ~/.local/bin/hermes-x
chmod +x ~/ClaudeCode/projects/常時運用/hermes-x-search/hermes-x

# PATHを確認
echo $PATH | grep -q ".local/bin" || echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### `x_search: authentication error` / `401 Unauthorized`

```bash
# xAI OAuth を再認証
hermes auth xai-oauth

# auth.jsonが存在するか確認
ls -la ~/.hermes/auth.json
```

### `x_search` ツールが使えない

```bash
# X Premium サブスクリプションの確認（SuperGrok枠が必要）
# config.yamlのproviderが xai-oauth になっているか確認
grep -A3 "model:" ~/.hermes/config.yaml
```

### 音声ファイルが生成されない / TTSエラー

```bash
# config.yaml の tts 設定を確認
grep -A3 "tts:" ~/.hermes/config.yaml
# → provider: xai, voice: eve になっていること

# 保存先ディレクトリが存在するか確認
mkdir -p ~/ClaudeCode/projects/常時運用/hermes-x-search/assets/$(date +%Y-%m-%d)
```

### ffmpegエラー（動画合成時）

```bash
# ffmpegのインストール確認
ffmpeg -version

# macOS
brew install ffmpeg

# 入力ファイルの形式確認
ffprobe -v quiet -print_format json -show_format -show_streams input.mp4
```

---

## セキュリティ注意事項

- `~/.hermes/auth.json` は **絶対にgitにコミットしない**（OAuthトークンが含まれる）
- `.env` ファイルも同様（gitignoreに追加済み）
- Xの自動操作（computer_use / Claude in Chrome）は **利用規約違反リスク** のため使用禁止
- SuperGrok枠の消費量に注意（X Premiumの利用制限内で運用）

---

## 参考リンク

- [Hermes Agent 公式ドキュメント](https://github.com/NousResearch/hermes-agent) ← 要確認
- [xAI API ドキュメント](https://docs.x.ai/)
- [Grok API リファレンス](https://docs.x.ai/api)
- [X Premium 料金プラン](https://help.twitter.com/en/using-x/x-premium)

---

*このガイドはAI収益化ラボ（河村風真）が運用するhermes-x-searchシステムを元に作成。*  
*Xへの投稿判断は必ずユーザーが行い、AIによる自動投稿は行わない。*
