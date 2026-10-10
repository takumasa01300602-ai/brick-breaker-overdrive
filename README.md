# ブレイクアウト・オデッセイ：魔導ブロック崩し (BREAKOUT ODYSSEY)

魔導パドルで光球を弾き返し、タイミングよく上フリックで「貫通スマッシュ」を叩き込め！分裂カプセルで盤面のボールを最大120個まで増殖させ、画面を埋め尽くす弾幕で巨大ボスブロックのコアを削り切る、脳汁全開のインフレ系ブロック崩し。

- **ジャンル**: ボール100個増殖×スマッシュブロック崩し
- **対象端末**: iPhone 17 シリーズ（iOS 26 Safari 縦画面）
- **本番 URL（予定）**: https://brick-breaker-overdrive.minicity-cloud.workers.dev
- **共通基盤**: たかGames（`taka-games`）共通アカウント & D1（`farm-farm-happy-db`）連携済み
- **企画・仕様・検証基準**: [PLAN.md](./PLAN.md) / [CLAUDE.md](./CLAUDE.md)

## 開発コマンド

```bash
npm.cmd install
npm.cmd run dev          # 開発サーバー起動 (http://localhost:8799)
npm.cmd run db:local     # ローカルD1に schema.sql を適用
npm.cmd run db:remote    # 本番D1に schema.sql を適用
npm.cmd run deploy       # Cloudflare Workersにデプロイ
npm.cmd run icons        # アプリアイコンPNG再生成
```
