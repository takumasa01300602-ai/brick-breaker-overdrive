# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) and AI coding assistants when working with code in this repository.

## 言語

**ユーザーへの返答は、常に日本語で書いてください。** 途中で英語に切り替えないでください。ツール出力やエラーメッセージが英語の場合も、説明は日本語で行います（コード・コマンド・識別子はそのままで構いません）。

## 概要

`brick-breaker-overdrive` は、**ブレイクアウト・オデッセイ：魔導ブロック崩し（BREAKOUT ODYSSEY）** のプロジェクトです。
- **ジャンル**: ボール100個増殖×スマッシュブロック崩し
- **キャッチコピー**: 「1球が100球に増殖！ボスブロックを粉砕する超インフレ崩し！」
- **概要**: 魔導パドルで光球を弾き返し、タイミングよく上フリックで「貫通スマッシュ」を叩き込め！分裂カプセルで盤面のボールを最大120個まで増殖させ、画面を埋め尽くす弾幕で巨大ボスブロックのコアを削り切る、脳汁全開のインフレ系ブロック崩し。
- **ターゲット端末**: **iPhone 17 シリーズ（iPhone 17 / 17 Pro / 17 Pro Max / iPhone Air、iOS 26 の Safari）の縦画面**。
- **基盤**: ポータルサイト `taka-games`（および `sugar-stack` / `dead-bridge-overdrive` / `fortress-of-runes` / `yubisaki-panic` / `farm-farm-happy`）から **本番アカウント・アカウント機能・D1データベース（`farm-farm-happy-db`）・コミット＆自動デプロイ環境** を完全引き継ぎ済みです。
- **現在のステータス**: **ゲーム本編実装・バグ＆ゲームバランス自動検証・ポータル連携完了**。詳細なゲーム企画・仕様は `PLAN.md` および下記の「ゲーム実装仕様」に記載しています。
- **重要原則**: **ゲームを実装した後は、ユーザーに確認を丸投げせず、必ず下記の「🚨 実装後のバグ検証・ゲームバランス検証（必須ルール）」に従って Claude 自身がバグ潰しとバランス調整（自動シミュレーション検証）を完了させてから報告・コミットしてください。**

Cloudflare Workers 無料プラン + D1 で配信します。描画は Canvas 2D（または WebGL）、音は WebAudio 合成（外部画像・音声ファイル不要、アイコン PNG のみ `tools/make-icons.mjs` で生成）。開発用の依存は `wrangler` だけです。コードのコメントやユーザー向けメッセージは日本語で書いてください。

## コマンド

**ユーザーに `npm` / `npx` のコマンドを案内するときは、`npm.cmd` / `npx.cmd` と `.cmd` を付けて書いてください**（Windows の PowerShell で実行ポリシーに止められないため）。以下の一覧は略記です。

```bash
npm.cmd install
npm.cmd run dev          # wrangler dev → http://localhost:8799 （.dev.vars 設定済み）
npm.cmd run db:local     # schema.sql をローカルのD1に適用
npm.cmd run db:remote    # schema.sql を本番のD1（farm-farm-happy-db）に適用（何度流しても安全）
npm.cmd run deploy       # wrangler deploy
npm.cmd run icons        # public/icons の PNG を作り直す
npx.cmd wrangler secret put INVITE_CODE                 # 本番の招待コード（他ゲームと同じものにする）
node tools/reset-password.mjs <名前> <新パスワード>      # 出力された wrangler d1 コマンド2つを実行する（全ゲーム共通）
```

動作確認は `npm.cmd run dev` で起動してブラウザで行います（ポートは他プロジェクトと重ならないよう **8799**）。`?debug` を付けて開くと端末チェック（iPhone 17 向けの注意画面）を出しません。PC のブラウザでもマウスで操作できます。

## 構成

- `src/index.js` — Worker 本体（**実装済み**）。`/api/*` だけを処理し、それ以外は `env.ASSETS` 経由で `public/` を返します。アカウント API は `taka-games` と同一仕様です。ゲーム用 API：
  - `GET /api/records` … 自分のベスト記録（`best_score`, `best_stage`, `best_balls`, `best_combo`）
  - `POST /api/records` … 記録の保存 `{ score, stage, balls, combo }`（自己ベスト更新、3秒に1回まで＝`too_soon` 429）
  - `GET /api/ranking` … 全体ハイスコア上位20人と自分の順位
- `public/index.html` — アカウント画面（ログイン・登録・パスワード変更・削除）、ランキング画面、iPhone 17 端末チェックゲート、WebAudio 解錠が組み込み済みです。ここにゲーム本体（または `public/game/*.js` モジュール）を実装します。
- `public/sw.js` と `manifest.webmanifest` — PWA。
- `schema.sql` — 冪等（`IF NOT EXISTS`）。共通テーブルと各ゲームのテーブル定義を含み、このゲーム用は `bb_records`（`users(id)` を `ON DELETE CASCADE` で参照）。
- `PLAN.md` — 本作の詳細企画書・システム仕様書・バグ＆バランス検証仕様。

### 認証モデル（サーバー・クライアント・他プロジェクトの実装が一致していないと動きません）
- パスワードのハッシュはブラウザ側で計算します（Workers 無料プランの CPU 時間が 10ms のため）。`PBKDF2-SHA256(NFKC(password), salt='minicity-v1:'+username.toLowerCase(), 600000回, 256bit)` の16進文字列を「キー」として送り、サーバーは `SHA-256(salt + key)` を保存します。
- 次の値は **全リポジトリで揃えておく必要があります**：`public/index.html` の `deriveKey`/`NAME_OK`/`KEY_SALT_PREFIX`、`src/index.js` の `NAME_RE`/`checkName`、`tools/reset-password.mjs`。変えると既存のアカウントで入れなくなります。

## ゲーム実装仕様（別セッションでの本編実装ガイド）

### 1. コアメカニクスと独自性
- **「最後の1個を待つストレス」を解消するボスコア＆スマッシュシステム**:
  - 従来のブロック崩しの弱点だった「残り1〜2個のブロックになかなか当たらない」問題を完全解決！
  - ① 各ステージ上部に **「BOSS CORE（巨大コアブロック）」** が存在し、コアのHPを0にした瞬間、盤面に残っている全ブロックが連鎖大爆発して即座にステージクリア（残ブロック数に応じた大ボーナス）！
  - ② パドルにボールが当たる瞬間に指を少し上に弾く（またはパドルが光るタイミングでタップする）と **「⚡SMASH SHOT」** となり、狙った方向へ貫通レーザー弾のように飛んでいく！
- **ボール最大120個増殖の圧倒的カタルシス**:
  - ブロックから落ちるカプセル：
    - `×2 MULTIBALL`: いま画面にあるすべてのボールがその場で2つに分裂（上限120個）！
    - `+3 SHOT`: パドルから3個のボールを追加射出
    - `🔥 FIREBALL`: 6秒間、全ボールがブロックで跳ね返らず貫通破壊
    - `⚡ LASER PADDLE`: パドル両端から自動レーザー連射
    - `🛡️ GUARD WALL`: 画面最下部に10秒間ボールを落とさない反射シールドを展開


## 🚨 実装後のバグ検証・ゲームバランス検証（必須ルール）

**ゲーム本編を実装したら、「コードを書いて終わり」にせず、ユーザーに完了報告・コミットする前に必ず Claude 自身が以下のバグ検証とゲームバランス検証を実行し、問題をすべて修正してから報告してください。**

### 1. 検証用のデバッグ・自動テストフック（実装時に必ず組み込むこと）
ブラウザのバックグラウンド状態やヘッドレス環境でもAI自身が1コマずつ検証・高速シミュレーションできるよう、`window.GAME` に以下のインターフェースを公開してください（`?debug` または `localhost` で有効化）：
- `window.GAME.step(frames = 1, input = {})` : `requestAnimationFrame` に頼らず、指定フレーム数（1フレーム＝`1/60`秒）だけ決定論的にゲームロジックを進めて描画を更新する。
- `window.GAME.state()` : 現在のゲーム状態（`mode`, `score`, `stage/wave/floor`, `hp/life`, エンティティ数, エラー有無）をプレーンオブジェクトで返す。
- `window.GAME.sim(runs = 30, botProfile = 'standard')` : 描画を省略した高速ヘッドレス通しプレイを `runs` 回実行し、平均スコア・平均生存時間・クリア率／到達段階・死因内訳を集計して返す。

### 2. バグ検証チェックリスト（全項目クリア必須）
1. **構文・実行時エラー 0件の確認**:
   - 開発サーバー（`http://localhost:8799/?debug`）またはブラウザで起動し、`console.error` / `pageerror`（未定義変数、null参照、Canvas / WebAudio 例外）が **1件も出ないこと**。
2. **全画面遷移・ライフサイクルの通し検証**:
   - `タイトル画面 → ゲーム開始 → 通常プレイ → スキル/ギミック発動 → 一時停止（ポーズ） → 再開 → ゲームオーバー/クリア → リザルト → もう一度遊ぶ（リトライ） → タイトルへ戻る → ランキングモーダル表示` の全ルートを実際に動かすこと。
   - **リトライを連続5回以上繰り返しても、`requestAnimationFrame` の二重起動（倍速化バグ）やイベントリスナーの重複登録、配列の初期化漏れが起きないこと**。
3. **エッジケース・入力耐性検証**:
   - フレーム落ち時（`dt` が大きく跳ねたとき）に物理演算や移動が破綻しないよう、必ず `dt = Math.min(rawDt, 1/30)` でクランプすること。
   - 画面端へのドラッグ、高速連打、2本指同時タッチ、ポーズボタン連打を行っても座標やスコアが `NaN` / `Infinity` にならないこと。
4. **iPhone 17 シリーズのビューポート・セーフエリア検証**:
   - `402×874`（iPhone 17 / 17 Pro）および `440×956`（iPhone 17 Pro Max）のサイズで、Dynamic Island（上部セーフエリア）やホームバー（下部セーフエリア）にボタン・HUDが重ならず、1画面に美しく収まっていること。
5. **クラウド記録・ランキング API 連携検証**:
   - ゲーム終了時に送信する `POST /api/records` のペイロード `{ score, stage, balls, combo }` がすべて整数（`Math.floor`）かつ `src/index.js` の上限範囲内に収まり、`200 OK` で保存されること。

### 3. ゲームバランス自動検証（感覚で数値を決めないこと）
実装後、必ず `window.GAME.sim()` または検証スクリプトで **3タイプのプレイヤーAI（①初心者/ランダム `novice`、②標準プレイヤー `standard`、③熟練プレイヤー `expert`）** を各30回以上走らせ、以下の基準を満たすまでパラメータ（速度・HP・出現率・コンボ倍率・スキル効果）を調整してください：
- **序盤30秒の爽快感**: `novice` でも開始15秒以内に理不尽な即死をせず、基本アクションの気持ちよさ（合体・撃破・コンボ）を体験できること。
- **適切なプレイ時間と難易度曲線**: `standard` で1プレイあたり **2〜5分程度** 楽しめ、中盤の山場（中ボス・フィーバー・進化）に到達できること。逆に簡単すぎて永久に終わらない「無限稼ぎループ」にならないこと。
- **スキル・選択肢の有意性**: 特定のスキルや選択肢だけが極端な一強・死に要素になっていないこと。

- **固有バグ検証**:
  - ボールの反射角がほぼ水平（$0^\circ$ や $180^\circ$ 付近）になり、左右の壁だけを延々と往復して降りてこなくなる現象を防ぐ「最小垂直速度（$|v_y| \ge v_{\min}$）補正」が入っているか。
  - ボールが100個以上に分裂した状態でボスコアを破壊し、全ブロックが一斉誘爆した際にフレーム落ちやクラッシュが起きないか。
- **固有ゲームバランス検証（自動シミュレーション 50試行）**:
  - 序盤から気持ちよく「×2分裂」「貫通」カプセルが出現し、1ステージあたり 45〜90秒で爽快にボスコア撃破まで到達できるブロック耐久値とドロップ率になっているか。


## iPhone 17 向けの注意
- 画面：iPhone 17 / 17 Pro は 402×874pt、Air は 420×912pt、17 Pro Max は 440×956pt。各画面はセーフエリア（`env(safe-area-inset-top)` 等）の内側に収めます。
- 音：iOS はタッチの中でしか音を鳴らし始められないため、`Snd.unlock()` を pointerup/touchend/click で呼び出しています。
- 触覚：必要に応じ iOS 18+ の `<input type="checkbox" switch>` トグルによるハプティクス演出を組み込みます。

## Git とデプロイ
- GitHub：`https://github.com/takumasa01300602-ai/brick-breaker-overdrive`（非公開、ブランチ `main`）
- 本番：`https://brick-breaker-overdrive.minicity-cloud.workers.dev` （Cloudflare Workers の `brick-breaker-overdrive`）
- 修正したら、そのつど日本語のメッセージでコミットします（`git add -A` → `git commit`）。
- `.git/hooks/post-commit` がコミット後に `git push` と `npm.cmd run deploy` を自動実行します（中身は `tools/post-commit.sh` と同じ）。push もデプロイもしたくないときは `SKIP_DEPLOY=1` を付けてコミットします。
- 初回本番デプロイ前には `npm.cmd run db:remote`（または `npx.cmd wrangler d1 execute farm-farm-happy-db --remote --command "..."`）で `bb_records` テーブルを作成し、`npx.cmd wrangler secret put INVITE_CODE` で共通の招待コードを設定してください。
