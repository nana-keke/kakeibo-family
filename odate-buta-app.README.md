# おだてブタ2 実戦記録アプリ — 開発再開用ドキュメント

このファイルは、Claude Code で開発を再開するときに読み込ませる想定のドキュメントです。
このREADMEと `index.html`（本体）をリポジトリのトップレベルに置いた状態で、
Claude Codeに「README.mdを読んで、このプロジェクトの続きを頼む」と伝えれば、
これまでの経緯・設計判断・未解決事項を踏まえた状態で作業を再開できます。

## 0. プロジェクト概要

- 対象: スマホの羽根物「Pポチッと一発！おだてブタ2」の実戦データ記録・分析アプリ
- 構成: **単一HTMLファイル（`index.html`）のみ**。GitHubの同一ディレクトリに姉妹アプリ（家計簿アプリ、同じくhtml1本構成）が存在するため、それに合わせている
- バックエンド: Firebase Realtime Database（既存の家計簿アプリと同じFirebaseプロジェクトを共用しているが、専用ルートパス配下に保存し、既存データとは分離）
- 認証: 匿名認証（`firebase.auth().signInAnonymously()`）。同一端末・同一ブラウザでの継続利用を前提
- デプロイ: GitHub Pages 等を想定。アクセス制限として `?key=2029` というクエリパラメータを要求する簡易ゲートを実装済み（本格的なセキュリティではなく、偶発的アクセスを防ぐ程度のもの）

## 1. ファイル構成

```
index.html   … アプリ本体（HTML + CSS + JS、外部ファイルなし）
README.md    … このファイル
```

外部読み込みは以下のみ（CDN）:
- Firebase compat SDK（app / auth / database）

## 2. Firebase設定・保存先

```js
const firebaseConfig = { ... }; // index.html冒頭に記載（家計簿アプリと共用プロジェクト）
const APP_ROOT = "odatteButaApp/v1";
```

保存パス:
```
odatteButaApp/v1/users/{firebaseAuthUid}/sessions/{sessionId}
odatteButaApp/v1/users/{firebaseAuthUid}/settings
```

同期方式:
- 操作のたびに即座に `localStorage` へ保存 → 非同期でFirebaseへ push
- push失敗時は `state.unsynced`（Set）に積んで保持し、`online`イベント発火時・12秒おきの`setInterval`で再送
- 起動時に一度だけFirebaseから `pullFromFirebase()` し、`updatedAt` を比較して**セッション単位・イベント単位（eventId基準）でマージ**（ローカル/リモートどちらのデータも失わない設計）

## 3. アクセス制限（簡易ゲート）

```js
const ACCESS_KEY = "2029"; // ← ここを変えれば鍵を変更できる
```

- URLに `?key=2029` を付けてアクセスすると `localStorage` に許可フラグが保存され、以後キー無しでも利用可能
- クライアント側のみの簡易的な足止めであり、真のセキュリティ対策ではない旨をユーザーには伝達済み

## 4. データモデル

### セッション（1実戦）

```js
{
  sessionId, date, machineNumber, startedAt, endedAt, status, // 'active' | 'ended'
  tilt: { frontBack, leftRight, note },
  counterBaseline,               // 羽根開放回数の基準値（実戦開始時のデータカウンター値）
  openingCounterSnapshots: [],   // データカウンターのスナップショット履歴（下記）
  events: {},                    // eventId をキーにしたイベントのマップ
  memo, createdAt, updatedAt,
}
```

### イベント（操作履歴）

```js
{
  eventId, sessionId, eventType, amount, detail, // detail は一部イベント種別のみ使用
  occurredAt, createdAt, updatedAt, deleted, note,
}
```

- 「ひとつ戻す」は最新の非削除イベントを論理削除するだけ（`deleted:true`）。物理削除はしない
- 集計値・率はFirebaseに固定値として保存せず、**常にイベントから再計算**する（`computeStats()`）

### データカウンタースナップショット（羽根開放回数）

```js
{ id, occurredAt, value, note, prevValue, editHistory:[], deleted, createdAt, updatedAt }
```

- 実戦中の羽根開放回数 = 最新スナップショット値 − `counterBaseline`
- 新しい値が前回より小さい場合は `confirm()` で警告してから記録（自動補正はしない）

## 5. イベント種別カタログ（`EVENT_DEFS`）

基本・ラウンド・SP・リトライ・2回目開放などは1ボタン=1イベントのシンプルな構成（`eventType`がそのままキー）。

| eventType | 内容 |
|---|---|
| PLATE | ＋1皿（125玉） |
| YAKUMONO | 役物入賞＋1玉 |
| NORMAL_WIN / SP_ENTRY / SP_WIN | ノーマル・SP系 |
| RETRY_ENTRY / RETRY_WIN | リトライ系 |
| JACKPOT2 / SECOND_HIT / SECOND_HIT_V / SECOND_HIT_RETRY | 2回目開放系（`JACKPOT2`はこのファネルの起点として`m.secondHitRate`の分母に使われるため、UI上もこのカードに配置） |
| ROUND3 / ROUND5 / ROUND10 | ラウンド系 |

### はたき関連（複数世代の仕様変更を経ているので特に注意）

はたきの入力仕様は会話の中で3回設計変更されており、**過去データを一切削除・自動変換せず、
新しい集計ロジックが旧世代のイベントも正しく合算できるように後方互換を維持**している。

- **第1世代**（最初期の仕様）: `HATAKI`（detail無し、独立の「はたき＋1」ボタン） / `HATAKI_N4`〜`HATAKI_N8`（独立ボタン、当時ははたき総数に自動加算されない仕様だった）
- **第2世代**（1タップ統合方式）: `HATAKI` の1種類に `detail:'unknown'|4|5|6|7|8` を持たせ、1タップ=1イベントで「回数不明」または「n回目」を記録。当時はこの1タップが総数にも内訳にも同時に反映される設計だった
- **第3世代（＝現行仕様）**: 「はたき総数」と「確認対象球の内訳」を**完全に独立した別集計**に分離
  - `HATAKI`（detail無し）＝ **はたき総数＋1**（単独ボタン、他のどこからも自動加算されない）
  - `HATAKI_CONFIRM`（detail無し）＝ **確認対象球＋1**（単独ボタン。見届けた球に対して必ず押す）
  - `HATAKI_CONFIRM`（detail:4〜8）＝ 確認対象球が実際にはたかれていた場合に**追加で**押す「n回目」タグ（＝はたかれた球は2タップになる）
  - `HATAKI_CONFIRM`（detail:'none'）は旧世代（前々回の仕様）の「はたかなかった」ボタンの名残。現在はボタンとしては廃止されているが、過去データ互換のため集計時は「確認対象球＋1」と同じ意味で合算される

現在の集計ロジック（`computeStats()` 内）:

```js
r.hatakiTotal            = sumHatakiTotal(session);        // 独立集計：はたき総数
r.hatakiConfirmedBalls   = sumHatakiConfirmedBalls(session);// 独立集計：確認対象球数（分母）
r.hatakiN4..N8           = sumHatakiN(session, n);          // 3世代分すべて合算
r.hatakiNTotal           = n4+n5+n6+n7+n8;
r.hatakiNotSweptRaw       = r.hatakiConfirmedBalls - r.hatakiNTotal; // 逆算（マイナスもありうる）
r.hatakiNotSwept          = Math.max(0, r.hatakiNotSweptRaw);
```

`r.hatakiNotSweptRaw < 0`（＝「確認対象球＋1」の押し忘れ疑い）は警告として表示する（`countValidWarnings()`）。

警告カード（`.warn-box`）は共通ヘルパー`warnBoxHtml(s)`に集約し、実戦中画面・集計画面・実戦詳細画面の3箇所で共通表示。`floating`クラスにより`position:fixed`で画面上部（トップバー直下）に固定表示され、スクロールしても常に見える（元々あった箇所の余白は占有しなくなるため、直下のコンテンツに一時的に重なる。使用感を見てから、余白を空ける対応を追加するか検討中）。

## 6. 主要な指標（`computeStats()` の `m` オブジェクト）

一番重要な指標は **皿単価**（`m.saraTanka` = 役物入賞玉数 ÷ 125玉使用回数）。実戦中画面の最上部に最大表示。

その他、率・割合はすべて「分子／分母」を併記する方針（`pct()` / `rate()` ヘルパー）。分母0は「－」。

はたき関連の指標のうち、以下は**確認対象球数**（`r.hatakiConfirmedBalls`）を分母にしている：
`m.hatakiNotSweptRate`, `m.hatakiNConfirmRate`, `m.hatakiN4Ratio`〜`hatakiN8Ratio`。

`m.hatakiRate`（はたき率）と `m.normalWinRate`（ノーマル当選率）は独立集計の**はたき総数**（`r.hatakiTotal`）を使う。

`m.avgHatakiCount`（平均はたき回数）は `hatakiNTotal` のみを分母にし、はたかなかった球は含めない。

## 7. 画面構成

タブ: 実戦 / 集計 / カレンダー / 分析 / 設定

- 実戦タブ内は状態に応じて `renderStartScreen` → `renderCounterScreen` → `renderHistoryScreen` / `renderEndScreen` を出し分け（`state.view.battle`）
- `renderStartScreen`の台番号は、設置台が固定のため手入力(旧: テキスト入力+datalist+過去入力履歴チップ)を廃止し、`MACHINE_NUMBERS`（`1001/1002/1003/1005/1006/1007/1008/1010`）からのワンタップ選択制に変更。選択値は`<input type="hidden" id="inMachine">`に格納し、左右傾斜クイック選択と同じ`selected`ハイライトのパターンを踏襲。未選択のままでも開始可能（任意項目のまま）
- 実戦中カウンター画面(`renderCounterScreen`)の並び順は上から：①皿単価(ヒーロー表示) → ②減算モードトグル → ③記録ボタン群（使用・基本 → はたき → ノーマル・SP → リトライ → 2回目開放 → ラウンド → 羽根開放回数更新（スタート）、**実際のタップ頻度順**） → ④直前の操作／警告 → ⑤統計グリッド → ⑥台番号・傾斜等の情報カード → ⑦ひとつ戻す・履歴・集計・実戦終了ボタン。記録ボタン群を皿単価の直下に配置し、スクロールなしでタップできるようにしている
- 羽根開放回数のスナップショット記録時、数値入力後に出ていた「メモ（任意）」の追加プロンプトは廃止（`note`は常に空文字で記録）
- `2チャッカー＋1`（`JACKPOT2`）は元は「使用・基本」カードにあったが、`2回目開放`ファネルの起点であるため`2回目開放`カードの先頭に移動済み（eventType・集計ロジックは変更なし、UI上の配置のみ）
- 各ボタンは色分け＋現在カウント数バッジ表示。押すと画面全体がその色でフラッシュする（`flashScreen()`）演出付き
- `役物入賞＋1玉`（`YAKUMONO`）と`＋1皿（125玉）`（`PLATE`）は最頻出ボタンのため、`EVENT_DEFS`に`strong:true`を付与し、他ボタンより強い画面フラッシュ（不透明度0.62・0.5秒、通常は0.42・0.35秒）にしている。UI上も「使用・基本」カード内で横並び＋専用スタイル（`.act-btn.hifreq`、枠太め・やや大きめ）で表示
- ボタン押下時は`navigator.vibrate()`によるバイブ演出あり（`strong`系ボタンは45ms、それ以外の実戦記録ボタンは18ms）。**iOS Safariは仕様上Vibration API非対応のためiPhoneでは振動しない**（Android Chrome等でのみ動作）
- `はたき総数＋1`（`HATAKI`）と`確認対象球＋1`（`HATAKI_CONFIRM`）も横並び配置（元は縦積みで各`big`表示だったが、実戦中のスクロール量削減のため統合）
- 実戦中のスクロール量を減らすため、`使用・基本`／`はたき`／`ノーマル・SP`／`リトライ`／`2回目開放`／`ラウンド`の6カードには`.cat-block.dense`（余白・ボタン間ギャップを縮小）を適用。ボタン自体の高さは変更していない（誤タップ防止のため）。低頻度の`羽根開放回数`カードは対象外
- 左右傾斜は左0.4〜右0.4を0.1度刻みの9ボタンでワンタップ選択できる（`LR_QUICK_VALUES`）。範囲外は手入力欄で対応（入力範囲そのものは制限しない）
- **減算モード**（`state.subtractMode`）: 実戦記録ボタン群の上にあるトグルをONにすると、以後のボタン押下は`amount:1`ではなく`amount:-1`のイベントとして記録される（誤タップの訂正用）。`sumEvents()`は元々`amount`をそのまま合算する設計なので、イベント本体・集計ロジックの変更は不要だった。ON中は画面フラッシュが`--danger`色になる。誤って常時ONのまま放置される事故を防ぐため、**リロードで自動的にOFFに戻る**（永続化しない・セッションデータの一部ではない）

## 8. 直近で見つけた不具合と修正

- 「確認対象球＋1」ボタンに全幅表示用の `big` クラスを付け忘れており、2列グリッドの中で半分の幅しか表示されず右側が空欄になっていた → 修正済み（`evButtonHtml('HATAKI_CONFIRM', r, 'big')`）
- 「リトライ穴＋1」（`RETRY_HOLE`）と「リトライ突入＋1」（`RETRY_ENTRY`）が実質同じ意味で重複していたため、`RETRY_ENTRY`（リトライ突入＋1）に一本化。あわせて「リトライ突入率」指標（穴÷突入）と超過警告も削除。**まだ本運用前でデータが無かったため、通常の「既存データは削除せず合算」の方針とは異なり、`RETRY_HOLE` 側のデータ・集計コードは合算せず完全に削除した**（§10の原則の例外。今後同様の重複が見つかった場合、既にデータが蓄積されていれば合算方式を取ること）

## 9. 未解決・要確認の事項（ユーザーへの積み残し確認）

- 「n回目はたき確認率」という指標名について、ユーザーから「AIとのラリーで生まれた歪んだ名称では」との指摘あり。名称変更・表示位置変更・非表示化のいずれかを検討中だが、まだ結論は出ていない
- 第3世代（2タップ方式）の「確認対象球＋1」＋「n回目」運用は導入直後で、実機での使用感がまだフィードバックされていない。UIの押しやすさ・押し忘れ検知の警告文言なども実運用しながら調整が必要になる可能性が高い

## 10. 開発時の注意点（既存の設計原則）

- **既存データを削除・自動変換しない**。仕様変更のたびに新しいeventTypeやdetailを追加し、集計関数側で旧世代データも合算できるようにする、という方針で一貫している
- 率・割合・単価はすべて「分子／分母」併記、分母0は「－」
- 内部計算は丸めた表示値を再利用しない（表示時のみ丸める）
- 固定設定値（125玉単価・閉店時刻・ラウンド別出玉など）は `APP_SETTINGS` に一元管理
- 傾斜の区分（0.3度刻み）の境界値は `TILT_CONFIG` に一元管理
- Firebaseには率や割合を極力保存せず、イベントから再計算する方針（集計スナップショットのキャッシュは許容するが、元イベントから再構築できることが条件）

## 11. 今後の作業を始めるときのおすすめの進め方

1. `index.html` を開き、まず `EVENT_DEFS` → `computeStats()` → `renderCounterScreen()` の順に目を通す（データモデルと表示がひとつながりで把握できる）
2. はたき関連を触るときは必ず本READMEの §5・§6 を読み、3世代分の後方互換ロジック（`sumHatakiTotal` / `sumHatakiConfirmedBalls` / `sumHatakiN`）を壊さないようにする
3. UIレイアウトを変更する際は、`.btn-row` がデフォルト2列グリッドである点に注意する（単独ボタンを置く場合は `big` クラス必須。§8の不具合と同種のミスに注意）
4. 変更後は必ず `node --check` 相当の構文チェック（本体は素のJSなので、`<script>` 部分を抽出してNode.jsで構文検証できる）を行う
