# English AI Teacher — プロジェクト固有ルール

`/home/manabu/projects/CLAUDE.md`（共通ルール）に加えて、本プロジェクトでは以下に従う。

## 概要
- ブラウザだけで動く英会話練習アプリ。GitHub Pages で配信する静的サイト
- 公開URL: https://my-english-teacher-org.github.io/english-app/ ／ リポジトリ: `my-english-teacher-org/english-app`
- 使用API: Google Gemini API（`generativelanguage.googleapis.com`）
- APIキー: 各利用者のブラウザの `localStorage`（`lm_api_key`）に保存。サーバーには送らない
- `index.html` がメイン（単一HTMLに全て内包）。`live-app.html` が Gemini Live API 版、`live.html` は Live版の旧実装

## 構成の制約
- **サーバーサイドが存在しない。** 中継サーバーを前提とする対処（APIキーの秘匿、CORS回避のプロキシなど）は現構成では採れない。必要になった場合は構成変更の合意を取ること

## 動作確認
- `file://` では CSP とブラウザ拡張の制約で正しく動かない。ローカルHTTPサーバで開く
  `nohup python3 -m http.server 8777 --bind 127.0.0.1 > /tmp/httpd.log 2>&1 &` → http://127.0.0.1:8777/index.html
- `127.0.0.1` は secure context なので `navigator.wakeLock` や `getUserMedia` もローカルで検証できる
- 初回は「プロファイル初期設定」画面（`#scr-profsetup`）が出る。画面IDは `#scr-profsetup` / `#scr-main` で、`.screen` クラスが `display:none` を効かせている

## バグ再発防止（決まりだけ。原因・テストの全文は `docs/claude_notes/bugs.md`）
- **学習者の発話の表示は逐語。** AIの解釈で `.cap-en` を置き換えない。補正・提案は下に添える。プロンプトにルールを足すときは矛盾する旧ルールを消す（2026-08-25）
- 学習者が黙る＝難しすぎる。話題を変える前に同じ話題を易しく言い換えさせる。段階は3段階で打ち止め（2026-08-25）
- 通話中は Screen Wake Lock を取る。`visibilitychange` で取り直す。「非対応」（`'wakeLock' in navigator`）と「今は取れなかった」を区別し、取得失敗では案内を出さない（2026-08-25）
- `explainApiError()` は Gemini が返すステータスを網羅する（400 と 401 を返し分ける）。分岐を足したら既存の分岐が奪われていないか確かめる（2026-08-25）
- 特定端末だけ Gemini に届かないのはアプリ側では回避できないことがある。CORS エラーの表示を鵜呑みにせず、アプリ内の「API接続テスト」（①単純GET→②プリフライト→③POST）で切り分ける。別端末・別ネットワークで再現するか、APIキーの未設定・失効も疑う（2026-08-25）
