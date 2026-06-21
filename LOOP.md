# LOOP — 管制パネル

ループの設定・ポリシーを一望する管制表（loop-engineering 流の manifest）。
思想は [README.md](./README.md)、規約・フォーマットは [CLAUDE.md](./CLAUDE.md)、状態は [STATE.md](./STATE.md)。

## アクティブなループ
- **name**: seed-loop
- **起動**: `/run-loop [seed]`
- **用途**: `config/` で定義（例：ビジネスアイデア創出／既存業務の改善提案）。設定は `/loop-setup` で構築。
- **運用形態**: 単一セッションでの手動起動が基本。将来はスケジュール起動での自動運用も可。

## ゲート設定
- **合格**: 全採点軸（`config/axes/`）が **80点以上（AND）**
- **足切り**: `config/gates/` のいずれかが FAIL なら即廃棄
- **修正**: **最大3回** ＋ 改善停滞検知（合計点が前回から伸びなければ早期打ち切り）
- **打ち切り**: `ideas/boneyard/` に理由付きで記録して廃棄（没理由は seed 昇格判断に使う）

## Human Gate（ハードストップ）
- 内部基準クリア → `ideas/pending/` に積む（**承認キュー＝非同期**）
- 人間が frontmatter `status` を `approved` / `rejected` に書き換える（Obsidian でレビュー、Working Copy でスマホ同期）
- `rejected` + コメント → **最優先**で修正ループに差し戻し
- `approved` → `ideas/approved/` へ（承認後の実行・具体化は用途による）

## 予算・リスク
- **リスクレベル**: L1〜L2 相当（生成＋検証は自動／実行・プロト化は人間承認後）
- **1サイクルの起動数**: gatekeeper ×1 ＋ critic ×（`config/axes/` の軸数）＋ 修正ごとに再採点。コスト意識を持つ。
- **安全方針**: Web からのプロンプト自動改変はしない／自律クローン＆実行はしない（[CLAUDE.md](./CLAUDE.md) 参照）

## コネクタ
- 現状 MCP 不要。将来、承認トレイを Notion に昇格する余地あり。

## 拡張の作法（エンジン非依存）
- 採点軸の増減 → `config/axes/*.md` を追加/削除するだけ（critic がその数だけ起動）。
- 足切りの増減 → `config/gates/*.md` を追加/削除するだけ（gatekeeper がその数を評価）。
