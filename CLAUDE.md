# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 概要

**miyaGolf**（旧 Miya Golf Log）— ゴルフのスコアを記録・分析する個人用 PWA。現行バージョン **v5.6**。
**全コードが `index.html` 1ファイルに収まっている**（HTML + CSS + バニラ JS、ビルド工程・依存パッケージ・npm なし）。
リポジトリ：`Yoshimimkinc/miya-golf-log`／公開URL：https://yoshimimkinc.github.io/miya-golf-log/

## 開発・実行・デプロイ

- **実行**：`index.html` をブラウザで直接開くだけで動く。ローカル確認は `start index.html`（Windows）、または `python -m http.server` などで配信してアクセス。
- **ビルド／lint／テスト**：なし。編集 = `index.html` を直接書き換える。フレームワーク・モジュール分割は導入しない方針（1ファイル静的配信を維持）。
- **デプロイ**：`main` に push すると GitHub Pages が自動デプロイ（1〜2分）。
  - ⚠ **Pages のデプロイが「Deployment failed, try again later」で失敗することが頻発する**。空コミット（`git commit --allow-empty`）で再トリガーすれば毎回復旧する。push 後はデプロイ成功まで見届けること。
- **更新の反映**：PWA はキャッシュが残るため、利用者は画面下部の「最新版に更新」(`hardUpdate()`) でキャッシュ削除＋クエリ付きリロード。ダメならタブ開き直し。
- **開発経路が2つある**：この Dropbox フォルダ（PC）と Web 版 Claude（スマホ、`claude/...` ブランチ経由）。**作業開始前に必ず `git fetch` してローカルが origin/main より遅れていないか確認する**こと（過去に65コミット差が生じた実績あり）。

## アーキテクチャ（big picture）

### 画面遷移
8つの `<section class="screen">`（`home` / `start` / `play` / `bulk` / `done` / `summary` / `stats` / `manage`）を `show(id)` で切り替える SPA 風構成。実ルーティングはなく `active` クラスの付け替えだけ。
加えてボトムシート型モーダルが3つ：`dataModal`（データ管理）／`playModal`（プレー中メニュー）／`photoModal`（スコアカード写真拡大）。

### 2つの入力モードと統一データモデル
- **ライブ入力**（`play`）：1画面・1タップ設計。スコアは数字グリッド（イーグル〜+5、スコアカード記号◎○△□■付き）、パットは0〜5、OB/1ペナ、事件簿メモ。ホール看板風バー＋攻略メモ折りたたみ＋**同ホールの過去事件簿表示**（`pastHoleNotes()`）。
- **一括入力**（`bulk`）：18ホールまとめて／途中から／個別セル編集。イベントタグ・振り返り（良かった/悪かった/次回）もここ。
- 両モードは同じ `holes[]` 配列を編集する。`round.inputMode` は常に `'hybrid'`。相互に行き来でき、データは共有される。

### 中心となるデータ構造（すべてグローバル変数）
- `holes[]`：18要素。各ホール = `{rel, putts, ob, pen, entered, note}`。
  **重要：`rel` は「パー基準の相対スコア」**。実スコア = `PARS[i] + rel`。`entered` が false のホールは未入力扱い。
- `round{}`：ラウンドのメタ情報（id / date / course / layout / partners / parTotal / pars / mantras / events / review / memo / photo / totals）。
  - **`round.pars`**：ラウンド自身が Par 構成を持つ（未登録コースの MD インポート対応）。Par の解決は `parFromRound()` を使う。
  - **`round.photo`**：スコアカード写真（縮小圧縮した dataURL）。まるごとバックアップに含まれる。MD/CSV には含まれない。
- `PARS[]`：現在編集中の18ホールのパー配列（`currentPars()` / `parFromRound()`）。
- `COURSE_MASTER`：コース定義マスタ。`{courses:{セクション名:[9つのパー]}, layouts:{表示名:[前半, 後半]}}`。コース追加はここを編集。
  登録済み：甲斐ヒルズ / 葛城 / 富嶽 / ホロン / リバー富士 / 菊川 / その他Par72。
- `HOLE_TIPS`：コース×セクション×ホールごとの攻略コメント（なければパー別の汎用文言）。
- `BASE_MANTRAS` / `EVENT_ITEMS`：マントラ初期値・イベントタグの選択肢。

### 永続化（localStorage のみ・バックエンドなし）
- `miyaGolfRounds`：保存済みラウンドの配列（履歴）。
- `miyaGolfDraft`：入力途中の下書き。`saveDraft()` が操作のたびに自動保存、`restoreDraft()` で復元。
- `miyaGolfMantras`：ユーザー編集したマントラ一覧（改行区切り）。
- `miyaGolfLastBackup`：最終バックアップ日時。未バックアップ検知の警告バナー（`bkWarn`）に使う。
- `miyaGolfAvgWindow`：平均スタッツの集計範囲（`5` / `10` / `25` / `year`）。ホームのタイルとスタッツ画面が連動。

### 後方互換の要：`ensureH()` と `normalizeRound()`
- `ensureH()`：古い／不完全なホールオブジェクトの欠損フィールドを補完。**localStorage・インポートから読んだ hole は必ず `ensureH()` を通す**（データ構造を変える時はここを更新）。
- `normalizeRound()`：バックアップ／MD インポートで取り込んだラウンドを補正。旧データ（練習/コンペ区分あり等）もここで互換吸収。

### スタッツ
`roundStats()`（1ラウンド）と集計系（ホーム3タイル＋`stats` 画面）。パーオン率・ボギーオン率・平均パット・3パット率・寄せワン率・炎上率（トリプル+以上）・Par3/4/5別平均・推移グラフ・コース別／ホール別平均・バーディ履歴・年間比較など。

### IN→OUT レイアウトのホール番号
`isInOutLayout()` が `layout` に `'IN→OUT'` を含むか判定し、`displayHoleNo()` が表示番号をずらす（後半を1〜9、前半を10〜18と表示）。`displaySideName()` がセクション名を返す。番号・セクション表示まわりを触る時に注意。

### 出力・インポート
- **Markdown**（`md()`）：YAML frontmatter（`type: golf_round` ＋ gir_pct・avg_putt・スコア内訳などDataview用数値）＋ JLPGA形式スコアカード（9ホール横並び×2表、記号付き）＋事件簿・振り返り。Obsidian 取り込み前提。ファイル名は `yyyymmdd_コース名_グロス.md`（`mdFileName()`）。
- **MDインポート**（`importMd()` / `parseRoundMd()`）：複数ファイル可。新形式（9ホール横並び表）・旧形式（1行=1ホール表）の両方を読める。→ 携帯⇄PCのObsidian往復編集が可能。
- **まるごとバックアップ／復元**（`exportBackup()` / `importBackup()`）：写真含む完全バックアップJSON。iOS共有シート対応（`navigator.share`）。復元は全置換 or IDマージを選択。
- **CSV書き出し**（`exportCsv()`）：分析用・1行=1ホール・Excel対応BOM付き。

## 編集時の注意

- **バージョン番号は2か所**：画面表示の `.version` div（`miyaGolf vX.X`）と `exportBackup()` 内の `version:` 文字列。上げる時は両方そろえる。
  ⚠ 現状 `exportBackup()` 側が `'2.5'` のまま更新漏れしている（2026-07-07時点）。次にバージョンを上げるとき一緒に直すこと。
- **デザインは `DESIGN_SYSTEM.md` に従う**：カラートークン（`--main` / `--deep` / `--goldsoft` 等）、スコア色の絶対ルール（**アンダー赤 `--under`／オーバー青 `--over`**）、ピル型ボタン・枠線なしカード・絵文字なし、などのトンマナ。
- モバイル前提（`max-width:480px`、ダブルタップズーム抑止）。CSS は `<style>` 内にすべて記述（1行が非常に長いので Edit 時は一意な部分文字列で置換する）。
- 関数はグローバルスコープ、`onclick` 属性から直接呼ぶ素朴な構成。
- 練習/コンペ区分は v4.0 で全廃済み。旧データ・旧MDのインポート互換だけ `normalizeRound()` に残っている。

## リポジトリ内ドキュメント

| ファイル | 内容 |
|---|---|
| `index.html` | アプリ本体（CSS/JSインライン） |
| `DESIGN_SYSTEM.md` | トンマナ・トークン・実装ルール |
| `WORKLOG.md` | v2.6→v5.6 の開発ログ |
| `DEVELOPMENT_SUMMARY.md` | 初期からの設計方針まとめ |
| `icon.svg` / `site.webmanifest` | mgモノグラムアイコン・PWA設定 |

## 未着手の改善候補（保留中）

- **Bグループ**：FWキープ率、ライブのペース表示、ベスト更新演出、天気チップ、同伴者スコア並記
- **Cグループ**：ヤーデージ表示、ティー選択、年間目標、ダークモード
- さらにシームレスな同期が欲しくなったら「GitHub同期」構想あり
