# ハンドオーバー：ステロイド誘発性高血糖 インスリン調整支援ツール

最終更新: 2026-06-12 / 作成: こうじ（yuukenn384@gmail.com）との共同設計セッション

このドキュメントは、現行の単一HTMLツール `steroid_hyperglycemia_tool.html` を起点に、Claude Code で
**改良版・BBI（強化インスリン）専用版・スライディングスケール版**などの派生アプリを作るための引き継ぎ資料です。
新しいセッションがこれ一枚で「臨床ロジック・実装・設計判断・既知の負債・拡張方針」を把握できることを目的とします。

---

## ⚠ 2026-06-13 改訂サマリ（最優先・最新。「朝の二段入力」A案）

日々調整タブ（STEP3）の**入力導線を「朝にまとめて入力」する2ブロック構成に変更**。計算ロジック（§1.3）は不変。

- **画面**：STEP3 を ①**前日ブロック**（見出し＝直近記録日。朝FBS=前日入力済みをprefill・訂正可／昼前・夕前・就寝前→朝/昼/夕ボーラス）＋ ②**当日ブロック**（当日朝FBS→眠前基礎／夜間・早朝低血糖）に分割。当日日付を変えると前日ブロックのみ追従（`syncPrevBlock()`／`prevBlockHTML()` 新設）。初日（前日記録なし）は「記録なし」表示＋ボーラス据え置き。
- **保存**：`upsertDay` を **gフィールド単位マージ**（`mergeG()` 新設）に変更。同一日付への追記で**既存の朝FBS等を消さない**のが主眼。`calcNextDay`/`saveDay` は「前日分→`prevRecord(today).date`」「当日FBS・hypo→当日付」に振り分け保存。`readDayForm` は `readTodayForm`/`readPrevForm` に分離。
- **「前日」の定義**＝直近の記録がある日（`prevRecord`、カレンダー前日に限定しない）。2〜3日に1回運用で間が空いても直近記録を前日として拾う。
- **エスカレーション判定**：当日朝FBS＋前日ブロック各値の最高で実施（朝に確認した値ベース）。
- **既知の留意点**：過去日の日中値（昼前/夕前/就寝前）の訂正は「翌日を当日に選ぶ→前日ブロックで訂正」。ログ行の「編集」は当日朝FBS・低血糖が主。ログ行から日中値も直接編集できるUIは次回候補。
- **検証**：`AI-Company/Project/インスリンツール改訂_朝入力A案/test/run.mjs`（jsdom・20ケース：既存15＋A案5。M1=マージで前日FBSが消えない回帰）。`node --check`＋`node run.mjs`で全PASS。改訂前バックアップ：同 Project の `backup/`。

## ⚠ 2026-06-12 改訂サマリ（下の本文より新しい）

本文（§0〜§7）は初版時点の記述。**2026-06-12 の改訂で以下が変わったため、矛盾する箇所はこのブロックを正とすること。**

- **配布**：iPhone でローカルHTMLは localStorage が永続化しない（`file://`の仕様）ため **GitHub Pages で公開**。
  公開URL `https://yuukenn384-code.github.io/insulin-steroid-tool/`、リポジトリ `~/insulin-steroid-tool`（公開・`index.html`）。
  iCloud `インスリンツール/steroid_hyperglycemia_tool.html` がアプリ実体（同内容）。端末間同期はされない→ export/import で移行（**残す**）。
- **① 日々調整のタイミングを修正（臨床的に重要）**：基礎は「**当日朝の空腹**（前夜の基礎の効果）」で調整する。
  **§1.3 の『前日 朝（FBS）→ 眠前の基礎』は誤りとして廃止**。ボーラスは従来どおり前日参照（朝←前日昼前 / 昼←前日夕前 / 夕←前日就寝前）。NPHは前日 夕〜就寝前で調整・当日朝空腹<120で安全減。
- **② データモデルを刷新**：`S.history`（push専用）→ **`S.log`**（日付ごと `{date,g:{fbs,preL,preD,hs},hypo,dose}`、1日1レコード=upsert）。
  日付ごとに**保存・参照・編集・削除**可。旧 `history` は読み込み時に `migrateHistoryToLog` で自動移行。
  `calcNextDay`＝**プレビュー**（血糖は保存するが dose は記録しない）、`adoptNext`＝**採用時にその日の dose を記録**。
  これにより §4 の負債「calcNextDayが呼ぶたびhistory.push（重複ログ）」は**解消済み**。`prevRecord/prev2Record` で前日・前々日参照。
- **採用前の手入力**：本日量の提案は採用前に編集可（id `n_nph/n_pe/n_basal/n_bB/n_bL/n_bD`、`nextEditFields`）。
- **増量条件のレジメン別既定**：`confirmDefaultFor(type)`＝BBI:false(1日)/NPH・basal:true(2日)。`S.rules.confirm2day` は `null`(既定従う)/`true`/`false` の3値、`effConfirm2day()` で解決。減量は常に即時。
- **UI整理**：STEP2 の提案レジメン説明は末尾の `<details>` に格納／`.alert b` を `.alert>b:first-child` に変更し本文中`<b>`の改行崩れを修正／「推奨に戻す」ボタン削除／STEP3 文言を「翌日量」→「本日量」。
- **未対応の負債**：デッドコード（`SCALES`/`correctionDose`/`scaleTier`/`scaleTable`/`cur.tier`）は**まだ残置**（スライディング版の種）。
- **検証基盤**：`AI-Company/Project/インスリンツール改訂/test/run.mjs`（jsdom・15ケース。①タイミングT1/T2/T3、②ログ、移行ほか）。`node --check`＋`node run.mjs`。
  設計・計画・改訂前バックアップ：同 `Project/インスリンツール改訂/`（specs/ plans/ backup/ work/）。

---

## 0. 一行サマリ

非ICU入院・朝1回プレドニゾロン（PSL）を主対象に、**前日の血糖プロファイルから当日の定時インスリン量を提案する
「パターン調整（proactive titration）」支援ツール**。スライディングスケールは使わない。単一HTML・依存ライブラリなし・
データは端末内（localStorage）のみ・外部送信なし。

> ⚠ これは臨床判断"支援"ツール。最終決定は指示医。院内での妥当性検証が前提。派生アプリでもこの位置づけと免責表示を必ず維持すること。

---

## 1. 臨床的背景とロジックの核

出典: VERA Health の相談資料 `ステロイド誘発性高血糖.pdf`（院内プロトコル案。透析・50kg・PSL朝1回・DPP-4i内服の具体例を含む）。
本ツールが扱うのは主に「朝1回PSL → 午後〜夕方に高血糖ピーク」のパターン。

### 1.1 初期レジメンの分岐（実装: `computeRegimen`）
- 食事 摂取不良/NPO → **基礎（持効型）＋（定時は保留）**。`type='basal'`
- 通常食 ＋ 重症域（直近血糖の最高 ≥350/HI）＋ 腎正常 → **BBI（基礎＋毎食前ボーラス）推奨**。`type='BBI'`
- それ以外（通常食）→ **NPH〔中間型〕朝（PSL同時）＋（定時）**。`type='NPH'`
- **腎不全の安全弁**: `renalLow = 透析 || eGFR≤30` の重症例は、BBIではなく **NPH低用量を推奨**（低血糖回避優先）。
  BBIは画面の「強化インスリンで見る」で手動切替可。
- 低用量開始 `lowStart = eGFR<60 || 透析 || 年齢≥70 || 摂取不良`。

### 1.2 開始量（係数 × 体重）
- TDD係数: NPH 0.15(lowStart)/0.25、basal 0.15/0.2、BBI 0.3(lowStart)/0.4 U/kg。`TDD = weight × factor`
- NPH: `nph = round(TDD×0.5)`、重症/高値なら各食前 定時を付与（残りを3等分）。
- BBI: `basal = floor(TDD×0.5)`（端数はボーラスへ）。ボーラスは**朝1回PSLで午後〜夕が上がる**ため
  **朝25% / 昼37.5% / 夕37.5%**に配分（資料/医師の例 50kg→基礎7・朝2/昼3/夕3 と一致）。

### 1.3 日々の調整 = パターン調整（実装: `calcNextDay`）★最重要
**スライディング（その場の補正）は使わない。** 毎朝、前日のAC/HS（朝・昼前・夕前・就寝前）を見て当日の定時量を動かす。

- **マッピング（医師指定・BBI）**:
  - 前日 朝（空腹/FBS） → **眠前の基礎**
  - 前日 昼前 → **朝ボーラス**
  - 前日 夕前 → **昼ボーラス**
  - 前日 就寝前 → **夕ボーラス**
- **調整スケール（`S.rules.scale`、2U刻み・編集可）**: 血糖値 → 該当インスリンの調整単位。既定:
  `<80:-4 / 80–119:-2 / 120–179:0 / 180–249:+2 / 250–349:+4 / 350–449:+6 / ≥450:+8`
  関数 `adjFromScale(g)` が単位を返す（負=減量）。
- **非対称ルール**:
  - **増量（delta>0）= 2日確認**（前日も同時間帯が増量域なら実施。1日目は据え置き）。`S.rules.confirm2day`
  - **減量（delta<0）= 即時**（1日で。低血糖回避を優先）。`hypo` チェック時は基礎 −4。
- 基礎インスリンの投与: **BBI/basal は眠前、NPH は朝（PSL同時）**。
- NPHモード: NPHは午後〜夕に効くため前日 夕前/就寝前で増減、空腹<120でNPH減、昼前でprandial増。
- **エスカレーション（量は変えず警告のみ）**: 直近最高 ≥360 mg/dL(≈20mmol/L)→DKA/HHS鑑別・医師コール・BBI/IV検討。
  ≥270(≈15mmol/L)持続→内分泌相談。

### 1.4 ステロイド減量（実装: `calcTaper`）
減量と**同日に**インスリン見直し、48時間 低血糖を厳重監視。減量比率（既定: ステロイド減少率の50%）で各量を減量。
BBIは**昼・夕ボーラスを優先的に**減量（係数1.3）、基礎・朝は控えめ（0.7）。内服化の判断は `oralGuide`（eGFR/透析でDPP-4i腎用量、SU/SGLT2i入院中休薬 等）。

---

## 2. 現行アプリの構成

- **ファイル**: `steroid_hyperglycemia_tool.html`（単一ファイル / Vanilla JS / 外部依存なし / CDNなし）。
- **タブ（SPA）**: `reg`(患者登録) / `init`(初期レジメン提案) / `daily`(日々の調整) / `taper`(減量・内服化)。
- **対象環境**: PC/iPhone Safari。`<meta apple-mobile-web-app-*>` でホーム画面追加対応。`@media (max-width:480px)` でスマホ最適化。

### 2.1 状態 `S`（グローバルではなく script スコープの `const`）
```js
S = {
  patient,   // 登録情報（下記）
  regimen,   // computeRegimen の戻り + cur + locked
  history,   // 日々ログ [{day,g,hypo,from,to,actions}]
  override,  // 'NPH'|'BBI'|'basal'|null（手動レジメン切替）
  activeKey, // 保存キー
  rules: { confirm2day:true, scale:[[80,-4],[120,-2],[180,0],[250,2],[350,4],[450,6],[9999,8]] }
}
```
- `patient = {id, weight, egfr, dialysis, age, dm, diet, steroid, stDose, stTiming, orals[], prof{fbs,preL,preD,hs}}`
- `regimen = {type, factor, tdd, nph, basal, prandialEach, bB, bL, bD, tier, sev, note[], cur{}, locked}`
- `cur = {nph, basal, prandialEach, bB, bL, bD, tier}` … 現在の定時量（titrationで更新される実体）
- `locked` … true の間は `renderInit` が `cur` を上書きしない（保存/確定済みの量を保持するための重要フラグ）

### 2.2 永続化（localStorage）
- キー: `steroidGcTool_v1`
- 形: `{ patients: { [key]: {patient, regimen, history, override, rules, updatedAt} }, activeKey }`
- 主要関数: `autosave()`（patient/regimen/confirm/calc/adopt/scale編集 のたびに呼ぶ）、`loadStore/writeStore`、
  `renderPatientBar`、`onSelectPatient`、`newPatient`、`deleteActivePatient`、`fillForm`、`exportJSON`、`importJSON`。
- 端末間移行はエクスポート(JSON)→iCloud→インポート。**自動クラウド同期は不採用**（医療情報を外部サーバへ出さないため）。

### 2.3 主要関数マップ
| 関数 | 役割 |
|---|---|
| `severity(p)`/`lowStart(p)`/`maxG(p)` | 重症度・低用量開始の判定 |
| `computeRegimen(p, override)` | 初期レジメンと開始量の計算 |
| `renderInit()` | STEP2 表示。`locked` なら現在量を保持、未確定なら提案を表示。開始量の編集欄を出す |
| `editFields(r)` / `confirmRegimen()` | 実際の開始量の編集 → 確定（`locked=true`） |
| `setOverride(t)` | レジメン手動切替（`S.regimen=null` で再計算） |
| `renderDaily()` | STEP3 表示。現在量・前日血糖入力・`ruleSettingsPanel()` |
| `adjFromScale(g)` / `ruleSettingsPanel()` / `adjScaleTable()` | 調整スケール（編集可） |
| `calcNextDay()` | ★前日プロファイル→当日量。`applyMap` でマッピング適用 |
| `adoptNext()` | 提案量を現在量に採用（保存） |
| `switchBBI()` | NPH/basal → BBI へ移行・再構成 |
| `renderTaper()` / `calcTaper()` / `oralGuide(p)` | 減量・内服化 |
| 永続化系 | 2.2 参照 |

---

## 3. これまでの設計判断と根拠（派生時に踏襲すべき思想）

セッションでの合意点。なぜそうなっているかを残す。

1. **スライディングは使わない／別アプリにする** … 反応的な補正単独は後手に回るため、前日パターンで定時量を動かす方式に。
   → 旧・補正スケール（`SCALES`/`correctionDose`/`scaleTable`）は**現在デッドコード**として残置（スライディング版の種に流用可）。
2. **増量は2日確認、減量は即時**（非対称）… ノイズで増やしすぎない一方、低血糖は待たない。透析/腎不全で特に重要。
3. **基礎は眠前（BBI/basal）／NPHは朝（PSL同時）** … 医師の実運用に合わせた。
4. **マッピング（前日朝→眠前基礎 / 昼前→朝 / 夕前→昼 / 就寝前→夕）** … 医師指定。BBIの中核ロジック。
5. **調整は2U刻みの単一スケール** … 1U刻みは細かすぎ。増量も減量も同じ表（負値=減量）で一元管理。
6. **腎不全（透析/eGFR≤30）の重症はBBIを自動推奨にせずNPH低用量を維持** … 低血糖回避優先。BBIは手動切替で。
7. **開始量は提案→医師が上書き** … `editFields`/`confirmRegimen`。提案は初期値に過ぎない。
8. **データは端末内のみ** … 患者個人情報のため。氏名でなくID/イニシャル推奨。自動同期は院内承認サーバが前提の別案件。

---

## 4. 既知の制約・技術的負債（要整理）

- **デッドコード**: `SCALES`, `correctionDose`, `scaleTier`, `scaleTable`, `tier`（`cur.tier`に残るが未使用）。
  スライディング版を作るなら流用、作らないなら削除推奨。
- `calcNextDay` は**呼ぶたびに `history.push`** する（「計算」＝記録）。複数回「計算」で重複ログが増える。
  → 派生では「計算＝プレビュー」「採用＝記録」に分離するとよい。
- NPHモードの日々調整は近似（単一NPH＋一律prandial）。BBIに比べロジックが粗い。
- `localStorage` は Safari の履歴消去や長期未使用で消える可能性 → 定期エクスポート前提。
- 単一HTMLのため、テストは `node --check` ＋ `jsdom` での手動シナリオ（下記）に依存。
- ステロイドは朝1回PSL前提。分割PSL・デキサメタゾン（長時間作用）は未対応（`stTiming`/`steroid` はパラメータ化済みで拡張余地あり）。

---

## 5. 派生アプリの方針

### 5.1 BBI（強化インスリン）専用版
- `type==='BBI'` 経路だけを残し、NPH/basal分岐・`switchBBI`・NPHの近似ロジックを削除して単純化。
- 中核はそのまま流用: マッピング（§1.3）＋ `adjFromScale` ＋ 2日確認/即時減量 ＋ 眠前basal。
- 追加余地: 高値域（+6/+8）に**1日あたり増量上限**（例: 各ボーラス +4/日 や 前日比+50%まで）を設けると安全。
- 既存の永続化・スマホ対応・`locked`機構はそのまま使える。

### 5.2 スライディングスケール版（別アプリ）
- 旧デッドコードが種: `SCALES`(Low/Med/High by 推定TDD<40/40–80/>80)、`correctionDose(g,tier)`、`scaleTable(tier)`。
- 資料の Low スケール（透析・低用量例）: `<150:0 /150–199:1 /200–249:2 /250–299:3 /300–349:4 /350–399:5 /400–449:6 /≥450:7+医師コール`、補正開始閾値 140–150。
- パターン調整版とは**思想が別**（反応的 vs 先回り）。両者を1アプリに混在させない方が安全（医師の明確な要望）。

### 5.3 共通改良候補
- ステロイド種類・時刻のパラメータ化（分割PSL/デキサメタゾン）。
- 血糖トレンドのグラフ表示（履歴の可視化）。
- 目標域(100–180)に対する達成率・低血糖イベントの集計。
- 単位系切替（mg/dL ↔ mmol/L）。
- 真の複数端末同期が必要なら、院内承認バックエンド＋認証＋暗号化＋匿名化を前提に別設計（PHIの扱いに注意）。

---

## 6. 拡張の進め方（Claude Code向けメモ）

- まず現行HTMLを単一ファイルのまま動かして挙動を確認。`renderInit`/`calcNextDay`/`autosave` の3つが心臓部。
- ロジック変更時の検証手順（このセッションで使った方法）:
  1. `node --check` で構文チェック（`<script>`内を抽出して実行）。
  2. `jsdom` で DOM 込みのシナリオテスト（登録→確定→2日調整→再起動で `cur` 復元、エクスポート整合）。
     - localStorage は `Object.defineProperty(window,'localStorage',{value:shim})` で差し替え、2回 boot して跨ぎ検証。
     - 関数は `window.<fn>`（トップレベル function 宣言）で呼べるが、`const S` は window に出ない→保存JSON/DOMで検証。
- 計算ロジックは UI から分離して純関数化（`computeRegimen`/`adjFromScale`/`calcNextDay`相当）すると、テスト・移植が楽。
- 免責・安全表示（重症高血糖の警告、医師確認前提）は派生でも必ず維持。

---

## 7. 出典・免責

- 臨床ロジックの出典: VERA Health 相談資料 `ステロイド誘発性高血糖.pdf`（院内プロトコル案）。
- 数値（係数・スケール・閾値）の一部は資料に明記がなく、医師との合意で既定値化したもの（画面で編集可）。施設標準での検証が必要。
- 本ツールおよび派生物は**臨床判断支援**であり、診療の最終決定は指示医が行う。医療情報・個人情報の取り扱いは院内規程に従うこと。
