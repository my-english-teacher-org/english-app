# テスト実行結果：発話の逐語表示・沈黙時の平易化・画面スリープ防止

- 実施日: 2026-08-25
- 対象: `index.html`（コミット `df6ae3a` 時点／main マージ済み）
- テスト設計: [`docs/test_design/conversation-display-test-design.md`](../test_design/conversation-display-test-design.md)
- 実行環境: Chrome（Linux x86_64）、ローカルHTTPサーバ `http://127.0.0.1:8777/` 上のページコンテキストで実行
- 結果: **42件中 42件 PASS（失敗0）**

## 結果サマリ

| 群 | 観点 | 件数 | PASS | FAIL |
|---|---|---|---|---|
| A | 発話表示（`fillUserFeedback`） | 12 | 12 | 0 |
| B | システムプロンプト | 4 | 4 | 0 |
| C | 沈黙時の段階的な平易化 | 7 | 7 | 0 |
| D | APIエラー変換 | 6 | 6 | 0 |
| E | 画面スリープ防止 | 13 | 13 | 0 |
| **計** | | **42** | **42** | **0** |

## A. 発話表示

| ID | ケース | 結果 |
|---|---|---|
| A1 | 発話が書き換わらない（`I pray tennis` のまま） | PASS |
| A2 | 直訳が出る（`私はテニスを祈ります`） | PASS |
| A3 | 提案は副表示（`💬 もしかして: I play tennis`） | PASS |
| A4 | 変更語が `.cap-diff` で強調される | PASS |
| A5 | 発音ヒントが出る | PASS |
| A6 | 表示順が 発話→直訳→提案→ヒント | PASS |
| A7 | 補正が元と同一なら提案なし・`sessionCorrections` に積まない | PASS |
| A8 | 補正が空なら提案なし | PASS |
| A9 | 訳が空なら `.cap-jp` を作らない | PASS |
| A10 | HTML混入時にタグが要素化されない（XSS） | PASS |
| A11 | 二重呼び出しで要素が重複しない | PASS |
| A12 | 対象不在で呼んでも例外を投げない | PASS |

A1 の実測値: `.cap-en` の `textContent` === `"I pray tennis"`（入力そのもの）。
A10 の実測値: `<img src=x onerror=alert(1)>` を入力しても `querySelectorAll('img,script,b,i').length === 0`、
かつ `.cap-en` は文字列としてそのまま保持。

## B. システムプロンプト

| ID | ケース | 結果 |
|---|---|---|
| B1 | `MUST be a LITERAL translation` を含む | PASS |
| B2 | 旧ルール `translate what you THINK they meant` を含まない | PASS |
| B3 | `let your own reply respond to that intended meaning` を含む | PASS |
| B4 | `or silence), drop down a level` を含む | PASS |

## C. 沈黙時の段階的な平易化

`window.fetch` をスタブ化し、送信された指示文を捕捉して検証。

| ID | ケース | 結果 |
|---|---|---|
| C1 | 3回連続の沈黙で3回とも異なる指示 | PASS |
| C2 | 1回目：`rephrased with much simpler` かつ `Do NOT change the topic` | PASS |
| C3 | 2回目：`one level simpler` かつ `Do NOT change the topic yet` | PASS |
| C4 | 3回目：`Switch to a much easier` | PASS |
| C5 | 学習者の発話（`sendMessage`）で `_stallCount` が 0 に戻る | PASS |
| C6 | 4〜6回目は3回目と同じ指示 | PASS |
| C6b | 段階は3種類のみ（際限なく増えない） | PASS |

## D. APIエラー変換

| ID | 入力 | 実際の `short` | 結果 |
|---|---|---|---|
| D1 | `Request had invalid authentication credentials` | APIキーが無効です（401） | PASS |
| D2 | `API key not valid` | APIキーが無効です | PASS |
| D3 | `HTTP 429` | API利用上限に達しました（無料Tierの制限の可能性） | PASS |
| D4 | `PERMISSION_DENIED` | アクセス権限エラー | PASS |
| D5 | `Failed to fetch` | ネットワークエラー | PASS |
| D6 | `HTTP 503` | サーバー一時エラー | PASS |

D2〜D6 は D1 の分岐追加による回帰確認。既存の判定を奪っていないことを確認した。

## E. 画面スリープ防止

`navigator.wakeLock` をスタブ化して検証。`document.visibilityState` も必要に応じて差し替えた。

| ID | ケース | 結果 |
|---|---|---|
| E1 | `acquireWakeLock` / `releaseWakeLock` が定義済み | PASS |
| E2 | Chrome が `wakeLock` に対応している | PASS |
| E3 | 取得すると `_wakeLock` が設定される | PASS |
| E4 | 保持中の再取得は `request` を重ねて呼ばない（冪等） | PASS |
| E5 | 解放で `_wakeLock` が null になる | PASS |
| E6 | 解放済みの再解放で例外を投げない | PASS |
| E7 | 通話中に可視へ復帰すると自動で取り直す | PASS |
| E8 | 通話中でなければ取り直さない | PASS |
| E9 | 非対応端末で例外を投げず false を返す | PASS |
| E10 | 取得失敗時に false を返す | PASS |
| E10b | 取得失敗で例外を投げない | PASS |
| E10c | 取得失敗では案内トーストを出さない | PASS |
| E10d | 取得失敗しても「非対応」と判定しない | PASS |

### 実機ブラウザでの補足観測

自動操作中はタブが `hidden` のため、実際の `navigator.wakeLock.request('screen')` は
仕様どおり以下で拒否された。

```
NotAllowedError: Failed to execute 'request' on 'WakeLock':
The requesting page is not visible
```

APIは利用可能（`'wakeLock' in navigator === true`、`isSecureContext === true`）であり、
拒否理由は可視性の前提条件のみ。この観測から E10 系のテストケースを追加し、
「非対応」と「一時的な失敗」を区別する実装に修正した。

## 未実施（実機での手動確認が必要）

| ID | 項目 | 状態 |
|---|---|---|
| M1 | 通話中、無操作で画面が消えないこと（5分以上） | **未実施** |
| M2 | 他アプリへ切替→復帰後も画面が消えないこと | **未実施** |
| M3 | 通話終了後は通常どおり画面が消えること | **未実施** |
| M4 | 実際の会話で発話が書き換わらないこと | **未実施** |
| M5 | 黙っていると教師が易しく言い換えること | **未実施** |

M1〜M5 は Android 実機での確認が必要。実際のAPI応答と端末のスリープ挙動に依存するため、
本環境では検証できていない。

## 別PCでのCORSエラーについて（未解決）

本テストとは別に、ある1台のPCで Gemini API への通信が遮断される事象が報告されている。
アプリ内の「API接続テスト」の結果は以下。

```
① Googleへの到達(単純GET): ✕ 失敗 — Failed to fetch
② プリフライトあり(ヘッダー付きGET): ✕ 失敗 — Failed to fetch
③ 生成API(POST): ✕ 失敗 — Failed to fetch
```

①（プリフライトを伴わない単純GET）から失敗しているため、**アプリ側のコードでは回避できない**。
そのPCと Google の間で `generativelanguage.googleapis.com` が遮断されている。
`github.io` からのページ読み込みは成功しているため、ネットワーク全体ではなくホスト単位の遮断である。

- 検証済み: 別のPCの Chrome では①〜③すべて到達成功（CORS応答も正常）
- 検証済み: `github.io` オリジンからのプリフライトに Google は `200` と
  `access-control-allow-origin` を返す
- 想定原因: 拡張機能・セキュリティソフトのTLS検査・DNSフィルタ・プロキシ
- 対応: 端末側の切り分けが必要（アプリ側の修正対象外）
