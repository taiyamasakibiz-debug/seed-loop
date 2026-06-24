# SEED-LOOP

汎用の「**起票 → 足切り → 審査 → 人間承認**」ループの最小構成テンプレート。

Seed（課題・テーマ）を入れると、AI が案を**起票**し、独立した審査 AI が**厳しく採点**し、
全基準を超えた案だけを**人間が最終承認**する——という自己改善ループを、Claude Code 上で動かす。
`config/` を差し替えるだけで用途が変わる（**エンジンは無改修**）。

> Status: 🌱 最小構成（minimal core）

## 何に使えるか
`config/` の中身次第で用途が変わる。例：
- 新規事業・ビジネスアイデアの創出（このテンプレの出自）
- 既存業務・組織の**改善提案**（現場課題 → 改善案 → 審査 → 承認 → 実行）
- コンテンツ企画・施策立案 など「案を出して厳しく選ぶ」あらゆるプロセス

## コアの流れ
```
seed（課題・テーマ）
  → ideator（起票：案のドラフトを作る）
  → gatekeeper（足切り：config/gates を二値 PASS/FAIL）
  → critic × N（採点：config/axes を各軸 0–100、全軸 80 以上の AND で合格）
       └─ 不合格なら ideator に差し戻して修正（最大3回＋改善停滞検知）
  → ideas/pending（承認待ち）→【人間が承認/却下】
       ├─ 承認 → ideas/approved
       └─ 却下・不合格 → ideas/boneyard（理由付きで記録）
```

## 設計：エンジンとインスタンスの分離
- **エンジン（`.claude/`）** = 汎用・ドメイン非依存。`ideator` / `gatekeeper` / `critic` と `run-loop`。
- **インスタンス（`config/`）** = 目的固有。差し替え可能。
- 各エージェントのインスタンス固有部分は config から読む：`ideator`→`config/ideator/`（起案ブリーフ）、`gatekeeper`→`config/gates/`、`critic`→`config/axes/`。
- `critic` は `config/axes/` のファイル数だけ、`gatekeeper` は `config/gates/` の数だけ起動する。
  → **評価軸・足切りの増減も、案の作り方の変更も config のみ。エンジンは書き換えない。**

## ディレクトリ構成と各ファイルの役割
| パス | 役割 |
|---|---|
| `CLAUDE.md` | Claude Code の常時コンテキスト（規約・各所の役割） |
| `LOOP.md` | 管制パネル（合格条件・修正上限・Human Gate・予算・安全方針） |
| `STATE.md` | 横断メモリ（最終実行・承認待ち・消費済み seed・没の要約） |
| `.claude/commands/run-loop.md` | **オーケストレータ**。`/run-loop [seed]` で1サイクルを指揮する指示書 |
| `.claude/agents/ideator.md` | **起票者(Maker)**。`config/ideator/`・前提プロファイル・seed から案を起草。`model:` で使用モデル指定 |
| `.claude/agents/gatekeeper.md` | **足切り**。`config/gates/` を二値 PASS/FAIL 判定（採点はしない） |
| `.claude/agents/critic.md` | **採点者**。`config/axes/` の1軸を反証ファーストで 0–100 点（独立コンテキスト） |
| `.claude/skills/loop-setup/` | **セットアップウィザード**。`/loop-setup` で `config/` を対話構築 |
| `config/founder-profile.md` | **前提プロファイル**。全判定が従う「隠れた前提」（利用目的別に作る） |
| `config/ideator/*.md` | **起案ブリーフ**。ideator がこの用途で案をどう構成・発想するか（出力フォーマット／起案スタンス） |
| `config/gates/*.md` | **足切り基準**（二値）。1ファイル = 1ゲート |
| `config/axes/*.md` | **採点軸**。1ファイル = 1軸（問い／レンズ／**点数バンド**／出力形式） |
| `config/seeds/*` | **起票の入力（種）**。関心領域・課題など。空でも引数指定で起動可 |
| `ideas/pending/` | 承認待ちトレイ。frontmatter の `status` を人間が書き換える |
| `ideas/approved/` | 承認済み |
| `ideas/boneyard/` | 没（`killed_by` と没理由を記録。seed 昇格の判断に使う） |

## 使い方
1. このリポジトリをクローン（Claude Code が使える環境）。
2. `/loop-setup` を実行 → 利用目的をヒアリングし、`config/`（前提プロファイル・gates・axes）を対話で構築。各エージェントのモデルも設定。
3. `/run-loop <seed>` で1サイクル。合格すれば `ideas/pending/` に承認待ちが積まれる。
4. `ideas/pending/*.md` の frontmatter `status` を `approved` / `rejected` に書き換えて承認（却下時はコメントを書くと次の修正へ）。

## 安全方針
- Web からのプロンプト自動改変はしない。
- 外部リポジトリの自律クローン＆実行はしない（人間が検証・承認したものだけ取り込む）。
- 実行レイヤーへの反映は、必ず**人間承認の後**。

## クレジット / ライセンス
- ループ設計の骨格は [cobusgreyling/loop-engineering](https://github.com/cobusgreyling/loop-engineering)（MIT）の作法を参考にしています。
- [MIT License](./LICENSE)
