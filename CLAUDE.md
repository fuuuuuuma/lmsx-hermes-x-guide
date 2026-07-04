# hermes-x-search 運用ルール

Hermes Agent（NousResearch）＋ Grok xAI OAuth を使った X（Twitter）検索・メディア生成・下書き保存の常時運用プロジェクト。

稼働構成・ファイル格納パス・hermes設定値・コマンド例は `README.md` を参照。

---

## 運用フロー（確定版）

```
① リサーチ（必須）
   └─ hermes-x でX検索 → 情報収集

② 下書き作成（任意）
   └─ 取得データをもとに投稿文を drafts/ に保存

③ メディア生成（任意）
   ├─ 画像: Grok Imagine → assets/YYYY-MM-DD/ に保存
   ├─ 音声: Grok TTS → assets/YYYY-MM-DD/ に保存
   └─ 動画: Grok Imagine → ffmpeg で音声合成 → assets/YYYY-MM-DD/ に保存

④ ユーザーが手動で実施
   └─ X投稿画面に貼り付け → 下書き保存または投稿
```

## トリガー → アクション

| ユーザー発話 | 即実行するアクション |
|---|---|
|「Xを検索して」「X検索」「Xで〜調べて」 | `hermes -z "..." -t x_search` を即実行。確認不要 |
| 「〜を10件取得して」 | X検索結果を指定件数取得し、「タイトル / 要約 / URL」形式で出力 |
| 「下書きを作って」「X用の文章を作って」 | 取得データをもとに投稿案を作成し `drafts/YYYY-MM-DD_*.md` に保存 |
| 「画像生成して」 | `hermes -z "image_generate..."` で Grok Imagine を即実行。`assets/YYYY-MM-DD/` に保存 |
| 「音声生成して」「読み上げて」 | `hermes -z "text_to_speech..."` で Grok TTS を即実行。`assets/YYYY-MM-DD/` に保存 |
| 「動画生成して」 | Grok Imagine で動画生成。`assets/YYYY-MM-DD/` に保存 |
| 「動画に音声を乗せて」 | ffmpeg で合成。尺を揃えてから `-final.mp4` として保存 |

---

## X 下書き保存（自動化不可）

**結論:** `hermes -z` からの X 下書き保存は不可。

**理由:**
- Hermes `-z` はブラウザを毎回使い捨て起動 → X ログイン状態が維持されない
- セッション再開（`--resume`）してもブラウザ状態は別管理で引き継がれない
- インタラクティブ `hermes` TUI なら可能だが Claude Code の Bash から TTY がないため起動不可
- Claude in Chrome MCP は BANリスクのため使わない

**運用:** Claude Code でテキスト・動画を生成・保存まで行い、X への貼り付けと下書き保存はユーザーが手動で行う。

| Claude Code が担当 | ユーザーが担当 |
|---|---|
| X 検索・情報収集 | X 投稿画面に貼り付け |
| 投稿文案の生成 | 動画ドラッグ&ドロップ |
| 画像・音声・動画の生成 | 「下書き」クリック |
| ファイルを `assets/` に格納 | 投稿するかどうかの判断 |

---

## 絶対にしないこと

- **computer_use・Claude in Chrome MCP・Hermes browser で X を操作しない** — いずれも BAN リスク。X の UI 操作は全てユーザー手動
- **X に直接投稿しない** — ファイル生成・下書きテキスト保存まで。X への貼り付けと投稿はユーザーが判断
- **下書き保存を自動化しようとしない** — Hermes -z はブラウザ使い捨てで X ログイン維持不可。試みても時間の無駄
- **`~/.hermes/auth.json` の内容をログや会話に出力しない** — OAuth トークンが含まれる
- **生成ファイルを `/tmp` や `~/ClaudeCode/` 直下に放置しない** — `assets/YYYY-MM-DD/` に格納
- **投稿文案を「完成」と言わない** — 投稿するかどうかの判断はユーザーに委ねる

## デバッグダンプの保持ポリシー

`~/.hermes/sessions/request_dump_*.json` は単なるデバッグログ（実行時に読み返されない）。
403切り分け（`feedback_hermes_x_credit_403`）に使うのは直近分のみなので、**30日を超えたものは削除してよい**：
`find ~/.hermes/sessions -name "request_dump_*.json" -mtime +30 -delete`（2026-07-04 に初回清掃済み）
