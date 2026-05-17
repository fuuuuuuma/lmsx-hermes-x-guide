# hermes-x 完全セットアップガイド

> **対象読者:** このリポジトリを受け取ったAIエージェント、または新環境にhermes-xを導入する開発者  
> **目的:** Xの最新アルゴリズムを活かしたリサーチ → 記事・画像・音声生成までを、AIエージェントが自律的にセットアップ・運用できる状態にする  
> **最終更新:** 2026-05-17  
> **参照元:**
> - X アルゴリズム: https://github.com/xai-org/x-algorithm
> - Grok Hermes: https://x.ai/news/grok-hermes

---

## 目次

1. [hermes-xとは](#hermes-xとは)
2. [Xの新アルゴリズム（2026年）](#xの新アルゴリズム2026年)
3. [システム構成](#システム構成)
4. [セットアップ手順](#セットアップ手順)
5. [ワークフロー](#ワークフロー)
6. [ファイル構造](#ファイル構造)
7. [運用ルール](#運用ルール)
8. [よく使うコマンド集](#よく使うコマンド集)
9. [トラブルシューティング](#トラブルシューティング)

---

## hermes-xとは

**hermes-x** は、Hermes Agent（NousResearch）と Grok（xAI）のX OAuth認証を組み合わせて、X（旧Twitter）の検索・リサーチ・コンテンツ生成を自動化するシステム。

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

> ソース: https://github.com/xai-org/x-algorithm

### コアアーキテクチャ

「For You」フィードは2つのコンテンツソースを統合している：

| ソース | 内容 |
|--------|------|
| **Thunder** | フォロー中アカウントの投稿（インネットワーク） |
| **Phoenix Retrieval** | 機械学習による発見投稿（アウトオブネットワーク） |

両者は **Grokベースのトランスフォーマーモデル** によって統一的にランク付けされる。手工業的な特徴量エンジニアリングは廃止済み。ユーザーのエンゲージメント履歴から自動学習する。

### スコアリング方式

```
Final Score = Σ (weight_i × P(action_i))
```

予測対象のアクション（約15種類）：

| アクション | 重み |
|-----------|------|
| いいね | 正（高） |
| 返信 | 正（高） |
| リポスト | 正（高） |
| ブックマーク | 正（高） |
| プロフィール訪問 | 正（中） |
| 動画視聴 | 正（中） |
| クリック | 正（中） |
| ブロック・ミュート | 負 |
| 報告（スパム等） | 負（大） |

**重要:** 候補同士はランキング時に相互参照できない（ユーザーコンテキストのみ参照）。スコアの一貫性とキャッシュ可能性を担保するための設計。

### 2026年の主要変更点

- **統合推論パイプライン**: 検索→ランキングを単一エントリポイントに統合
- **Grox コンテンツ理解**: スパム検出・分類・ポリシー適用が強化
- **広告ブレンディング**: フィード内への広告注入ロジックが刷新
- **事前学習モデル**: Git LFS経由で配布される3GBアーカイブ（外部開発者向け）

### hermes-xが活かせるアルゴリズム上のポイント

```
ブックマーク・返信・リポストを同時に獲得するコンテンツ = スコア最大化
```

- 保存したくなる「Tips型」「まとめ型」コンテンツはブックマークを稼ぐ
- 返信を促す「二択質問」「意見募集」は返信スコアに直結
- 動画は視聴完了率が重要（冒頭3秒フック必須）
- ブロック・ミュート・報告を誘発するコンテンツはスコアが大きく下がる

---

## システム構成

```
ユーザー / Claude Code
    │
    ├─ hermes-x "クエリ"
    │       └─ Hermes Agent v0.14.0
    │               ├─ モデル: grok-4.3（xai-oauth）
    │               ├─ x_search ツール（SuperGrok枠、API課金なし）
    │               ├─ image_generate（Grok Imagine）
    │               ├─ text_to_speech（Grok TTS / voice: eve）
    │               └─ 動画生成モード
    │
    ├─ assets/YYYY-MM-DD/       ← 生成物を格納
    └─ drafts/YYYY-MM-DD_*.md  ← 投稿下書き
```

**依存関係:**
- Hermes Agent v0.14.0+
- Python 3.11+
- ffmpeg（音声合成・動画処理）
- X Premium サブスクリプション（SuperGrok枠のため必須）

---

## セットアップ手順

> AIエージェントはこのセクションを上から順に実行すること。

### Step 1: Hermesのインストール

```bash
pip install hermes-agent
# または
pipx install hermes-agent

# バージョン確認（v0.14.0以上であること）
hermes --version
```

### Step 2: xAI OAuth認証の設定

```bash
# xAI OAuthでログイン（ブラウザが開いてX認証を求められる）
hermes auth xai-oauth

# 認証後、トークンは ~/.hermes/auth.json に自動保存される
# このファイルは絶対にgitにコミットしない
```

**前提条件:** X PremiumまたはX Premium+サブスクリプションが必要（SuperGrok枠のため）

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

### Step 4: X検索ツールの有効化確認

```bash
# x_search ツールが有効か確認
hermes -z "テスト" -t x_search
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

### Step 6: symlinkの作成（PATHを通す）

```bash
mkdir -p ~/.local/bin
ln -sf ~/ClaudeCode/projects/常時運用/hermes-x-search/hermes-x ~/.local/bin/hermes-x

# PATHの確認
echo $PATH | grep -q ".local/bin" && echo "OK" || echo "PATHへの追加が必要"
```

`~/.zshrc` または `~/.bashrc` に以下を追加（必要な場合）:
```bash
export PATH="$HOME/.local/bin:$PATH"
source ~/.zshrc
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

# TTSテスト
hermes -z "text_to_speechで「テストです」をeve音声で /tmp/test.ogg に保存して"
```

---

## ワークフロー

### 全体フロー

```
① リサーチ（必須）
   └─ hermes-x でX検索 → 情報収集

② 下書き作成（任意）
   └─ 取得データをもとに drafts/ に保存

③ メディア生成（任意）
   ├─ 画像: Grok Imagine → assets/YYYY-MM-DD/ に保存
   ├─ 音声: Grok TTS → assets/YYYY-MM-DD/ に保存
   └─ 動画: Grok Imagine → ffmpegで音声合成 → assets/YYYY-MM-DD/ に保存

④ ユーザーが手動で実施
   └─ X投稿画面に貼り付け → 下書き保存または投稿
```

### ワークフロー① リサーチ

```bash
# 基本形（5件返却）
hermes-x "調べたいトピック"

# 件数・フォーマット指定
hermes -z "Xで「生成AI」の最新投稿を10件返して。タイトル/要約/URL形式" -t x_search

# 特定アカウントの投稿を検索
hermes -z "Xで @openai の最新ツイートを5件まとめて" -t x_search
```

### ワークフロー② X記事・投稿文の作成

**アルゴリズム対応チェックリスト（スコア最大化）:**
- [ ] 冒頭に数字または固有名詞を含む
- [ ] ブックマークを促すTips・まとめ形式にする
- [ ] 末尾に返信を促す二択質問を入れる
- [ ] 動画付きの場合: 冒頭3秒にフックを入れる
- [ ] ブロック・報告を誘発する過激な表現を避ける

**投稿文テンプレート（X向け）:**
```
[フック] 冒頭に数字 or 固有名詞で問題提起

[本文] 解決策 + 具体的な数字/ツール名

[Tips] 保存したくなる情報（ブックマーク狙い）

[CTA] 「どっちが好き？」などの二択質問（返信狙い）

[URL・ハッシュタグ] #Claude #AI #生成AI
```

### ワークフロー③ 画像生成

```bash
DATE=$(date +%Y-%m-%d)
mkdir -p ~/ClaudeCode/projects/常時運用/hermes-x-search/assets/$DATE

# 16:9画像（PC・サムネイル用）
hermes -z "image_generateツールで16:9の[テーマ]の画像を生成してパスを返して"

# 9:16画像（モバイルフィード最適化）
hermes -z "image_generateツールで9:16縦型の[テーマ]の画像を生成してパスを返して"

# 生成後 assets/ に移動
mv /path/to/generated.jpg ~/ClaudeCode/projects/常時運用/hermes-x-search/assets/$DATE/
```

### ワークフロー④ 音声生成（TTS）

```bash
DATE=$(date +%Y-%m-%d)
mkdir -p ~/ClaudeCode/projects/常時運用/hermes-x-search/assets/$DATE

hermes -z "text_to_speechツールで次のテキストをeve音声で \
  ~/ClaudeCode/projects/常時運用/hermes-x-search/assets/${DATE}/voice.ogg に保存して: \
  [読み上げテキスト]"
```

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

---

## ファイル構造

```
~/ClaudeCode/projects/常時運用/hermes-x-search/
├── README.md              # このファイル（AIエージェント向けセットアップガイド）
├── CLAUDE.md              # Claude Code向け運用ルール
├── hermes-x               # X検索スクリプト本体（実行権限付き）
├── .gitignore             # auth.json・assets・draftsを除外
├── assets/
│   └── YYYY-MM-DD/
│       ├── thumbnail.jpg      # 生成画像
│       ├── voice.ogg          # 生成音声
│       ├── video.mp4          # 生成動画（音声なし）
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

### 権限分担

| hermes-x / AIエージェントが担当 | ユーザーが担当 |
|--------------------------------|----------------|
| X検索・情報収集 | X投稿画面への貼り付け |
| 投稿文案の生成 | 動画のドラッグ&ドロップ |
| 画像・音声・動画の生成 | 「下書き」ボタンのクリック |
| `assets/` への格納 | 投稿するかの最終判断 |

### 禁止事項

| 禁止事項 | 理由 |
|----------|------|
| Xへの自動投稿 | BAN対策（利用規約違反リスク） |
| `~/.hermes/auth.json` をログ・会話に出力 | OAuthトークン漏洩リスク |
| computer_use / Claude in Chrome でXを操作 | BAN対策 |
| 下書き保存の自動化を試みる | Hermes `-z` はブラウザ使い捨てでXログイン不可 |
| 生成ファイルを `/tmp` に放置 | 中間ファイルの散乱防止 |

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
hermes -z "image_generateツールで動画モードで[プロンプト]の動画を生成してパスを返して"

# === 動画＋音声合成 ===
ffmpeg -i video.mp4 -i voice.ogg \
  -filter_complex "[0:v]setpts=PTS*(音声秒数/動画秒数)[v]" \
  -map "[v]" -map 1:a -c:v libx264 -c:a aac -shortest final.mp4 -y

# === Hermes管理 ===
hermes --version       # バージョン確認
hermes update          # アップデート
hermes auth xai-oauth  # xAI再認証
hermes config list     # 設定確認
```

---

## トラブルシューティング

### `hermes: command not found`

```bash
pip show hermes-agent || pipx list | grep hermes
pip install hermes-agent
```

### `hermes-x: command not found`

```bash
ln -sf ~/ClaudeCode/projects/常時運用/hermes-x-search/hermes-x ~/.local/bin/hermes-x
chmod +x ~/ClaudeCode/projects/常時運用/hermes-x-search/hermes-x
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc
```

### `x_search: authentication error` / `401 Unauthorized`

```bash
# xAI OAuth を再認証
hermes auth xai-oauth
ls -la ~/.hermes/auth.json  # ファイル存在確認
```

### `x_search` ツールが使えない

X Premiumサブスクリプションが有効か確認（SuperGrok枠が必要）。  
`~/.hermes/config.yaml` の `provider: xai-oauth` を確認。

### TTS音声が生成されない

```bash
grep -A3 "tts:" ~/.hermes/config.yaml
# → provider: xai, voice: eve であること
mkdir -p ~/ClaudeCode/projects/常時運用/hermes-x-search/assets/$(date +%Y-%m-%d)
```

### ffmpegエラー（動画合成時）

```bash
brew install ffmpeg  # macOS
ffprobe -v quiet -print_format json -show_format -show_streams input.mp4
```

---

## セキュリティ注意事項

- `~/.hermes/auth.json` は **絶対にgitにコミットしない**
- `.env` ファイルも同様（.gitignoreに追加済み）
- Xの自動操作（computer_use / Claude in Chrome）は利用規約違反リスクのため禁止
- SuperGrok枠の消費量に注意（X Premiumの利用制限内で運用）

---

## 参考リンク

- [X アルゴリズム 公式リポジトリ](https://github.com/xai-org/x-algorithm)
- [Grok Hermes 公式ページ](https://x.ai/news/grok-hermes)
- [xAI API ドキュメント](https://docs.x.ai/)
- [X Premium 料金プラン](https://help.twitter.com/en/using-x/x-premium)

---

*このガイドはAI収益化ラボ（河村風真）が運用するhermes-xシステムを元に作成。*  
*Xへの投稿判断は必ずユーザーが行い、AIによる自動投稿は行わない。*
