# ニュートキオ 実戦記録アプリ — 開発再開用ドキュメント

このファイルは、Claude Code で開発を再開するときに読み込ませる想定のドキュメントです。
このREADMEと `newtokio-app.html`（本体）をリポジトリのトップレベルに置いた状態で、
Claude Codeに「README.mdを読んで、このプロジェクトの続きを頼む」と伝えれば、
これまでの経緯・設計判断・未解決事項を踏まえた状態で作業を再開できます。

姉妹アプリ「おだてブタ2」（`odate-buta-app.html` / `odate-buta-app.README.md`）と
同じアーキテクチャをベースにしています。共通部分の詳細（Firebase同期の考え方、
イベントソーシングの原則など）はそちらのREADMEも参照してください。

## 0. プロジェクト概要

- 対象: 羽物機「ニュートキオ」の実戦データ記録・分析アプリ
- 構成: **単一HTMLファイル（`newtokio-app.html`）のみ**。他の姉妹アプリ（家計簿・おだてブタ等）と同じ、GitHubの同一ディレクトリに置く前提
- バックエンド: Firebase Realtime Database（他の姉妹アプリと同じFirebaseプロジェクトを共用しているが、専用ルートパス配下に保存し、既存データとは分離）
- 認証: 匿名認証（`firebase.auth().signInAnonymously()`）。同一端末・同一ブラウザでの継続利用を前提
- デプロイ: GitHub Pages等を想定。アクセス制限として `?key=2029` というクエリパラメータを要求する簡易ゲートを実装済み（本格的なセキュリティではなく、偶発的アクセスを防ぐ程度のもの。1行の定数`ACCESS_KEY`なので変更は容易。おだてブタ2と同じ値になっている）

## 1. このアプリ最大の目的：タワー滞在時間と傾斜の相関分析

ニュートキオでは、大当たり後に中央のタワーで玉が左右にゆらゆら揺れて滞在する時間がある。
**「傾斜が手前になるほど、その滞在時間が伸びるはず」という仮説を検証したい**、というのが
このアプリを作った一番の動機。

- 実戦中カウンター画面に「タワー滞在計測」という手動ストップウォッチカードがある。1タップ目で計測開始、2タップ目（同じボタン）で計測終了・記録。抽選結果（当たり/はずれ）は記録しない（滞在時間と傾斜の相関だけ見られれば十分という判断）
- 計測中の状態（`state.towerTimer = {sessionId, startedAt}`）は**localStorageに永続化**しており、リロードしても計測中断にならない（Firebaseには同期しない、単一端末内の一時状態）。実戦終了時に計測中のタイマーが残っていた場合は黙って破棄する（`endSession()`内）
- 記録は`eventType:'TOWER_STAY'`のイベントとして保存され、他のイベントと違い`durationMs`という専用フィールドに実測時間（ミリ秒）を持つ
- **分析タブの「前後傾斜別」グルーピングで、傾斜帯ごとの平均タワー滞在時間を直接比較できる**ようにしてある（`aggregateSessions()`が`towerStayCount`/`towerStayTotalMs`を分子分母として合計してから平均を再計算する方式。単純平均ではない）。これがこのアプリの核心機能

## 2. おだてブタ2から流用した部分・変更した部分

流用（考え方・計算式ともおだてブタと同一）:
- 皿単価（`m.saraTanka` = 役物入賞玉数 ÷ 使用皿数）とその関連指標（皿単価あたりの期待差玉、期待収支・期待時給の考え方）
- SPルート（`SP_ENTRY`/`SP_WIN`）
- 傾斜の記録・区分（`TILT_CONFIG`、前後傾斜0.3度刻み）、台番号のワンタップ選択、ハンマー・ハンドル・ストップボタンの「癖」評価（1〜5段階）
- Firebase同期方式（localStorage即時保存→非同期push、`updatedAt`ベースのマージ、`eventId`単位でのイベント合算）
- 集計は常にイベントから再計算する方針（率や割合はFirebaseに固定保存しない）

変更した部分:
- ラウンド出玉はおだてブタの値より各+10玉（`APP_SETTINGS.payout3R/5R/10R = 250/460/1020`）
- リトライルートは無し（`RETRY_ENTRY`/`RETRY_WIN`は実装していない）
- 2回目開放ファネル（`JACKPOT2`/`SECOND_HIT`系）・はたき（`HATAKI`系）は無し。タワー滞在計測はこれらとは別の独立した新機能
- 台番号は`1011`〜`1018`の固定リスト（`MACHINE_NUMBERS`）

新規追加:
- タワー滞在計測（上記1章）
- ノーマルルートの当たり方パターン（下記3章）

## 3. ノーマル当選パターン（`NORMAL_WIN_PATTERNS`）

ノーマルルートの当たり方には名前付きパターンがある。単一の`eventType:'NORMAL_WIN'`に
`detail`タグでパターンを区別する方式（おだてブタの「はたき確認球のn回目タグ」と同じ、
1つのeventType + detailタグ配列という拡張パターンを踏襲）。

現在分かっている4パターン + それ以外:

| detail | ラベル |
|---|---|
| `dote_goe` | 土手越え |
| `pole_dance` | ポールダンス |
| `turntable` | ターンテーブル |
| `straight_in` | ストレートイン |
| `other` | それ以外 |

**新しいパターン名が判明したら、`NORMAL_WIN_PATTERNS`配列に1件追加するだけでよい**。
既存データ（`detail:'other'`で記録済みのもの）は変換・削除しない（おだてブタと同じ
「既存データを削除・自動変換しない」原則）。集計タブ・分析タブのパターン別内訳は
`sumNormalWinPattern(session, detail)`で都度再計算しているため、パターン追加時に
過去データを触る必要はない。

## 4. データモデル

セッション・イベントの基本構造はおだてブタ2と同じ
（`{sessionId, date, machineNumber, tilt, counterBaseline, openingCounterSnapshots, events, hammerHabit, isDemo, ...}` /
`{eventId, sessionId, eventType, amount, detail, occurredAt, deleted, ...}`）。

`TOWER_STAY`イベントのみ、`durationMs`（実測滞在時間・ミリ秒）という専用フィールドを持つ
（他のイベント種別では使わない、一部イベント種別のみ使うフィールドとしての拡張）。

保存パス:
```
newTokioApp/v1/users/{firebaseAuthUid}/sessions/{sessionId}
newTokioApp/v1/users/{firebaseAuthUid}/settings
```

## 5. 未解決・要確認の事項

- ノーマル当選パターンは現状4種類+それ以外のみ。実戦を重ねる中で新しいパターン名が
  出てきたら、`NORMAL_WIN_PATTERNS`への追加と、実戦中画面のパターンボタン配置
  （現状2列グリッド+それ以外を1行）の見直しが必要になる可能性がある
- タワー滞在計測は「傾斜と滞在時間の相関」を見る目的のみで、抽選結果（当たり/はずれ）
  は記録していない。今後、結果も合わせて分析したくなった場合は、別途イベントに
  結果フィールドを追加するか、判断が必要
- 台番号リスト（`1011`〜`1018`）・ラウンド出玉（+10玉という調整）は、ユーザーの記憶に
  基づく暫定値。実機で確認しながら`APP_SETTINGS`/`MACHINE_NUMBERS`を調整する前提

## 6. 今後の作業を始めるときのおすすめの進め方

1. `newtokio-app.html`を開き、まず`NORMAL_WIN_PATTERNS` → `computeStats()` → `renderCounterScreen()`の順に目を通す
2. タワー滞在計測まわりを触るときは、`state.towerTimer`のlocalStorage永続化・`startTowerStay()`/`stopTowerStay()`・分析タブでの傾斜別集計（`aggregateSessions()`の`towerStayCount`/`towerStayTotalMs`）を壊さないようにする
3. 変更後は`newtokio-app.html`内の`<script>`部分を抽出し、`node --check`で構文チェックする
