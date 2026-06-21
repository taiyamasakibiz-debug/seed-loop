---
description: ビジネスアイデア創出ループを1周回す（承認処理→起案→足切り→採点→修正→承認待ち）。引数で seed を指定可。
---

あなたはビジネスアイデア創出ループのオーケストレータ。**1サイクル**を回す。
起案・足切り・採点は**サブエージェントに任せ、自分で代行しない**（独立コンテキストを保つため）。

引数: $ARGUMENTS （seed のテーマ。空なら `config/seeds/` から1つ選ぶ）

## 設定（LOOP.md と一致させること）
- 合格: **全採点軸が 80点以上（AND）**
- 足切り: `config/gates/` のいずれかが FAIL なら即廃棄
- 修正: **最大3回** ＋ 改善停滞検知（合計点が前回から伸びなければ早期打ち切り）
- 打ち切り企画: `ideas/boneyard/` に**理由付きで記録**して廃棄

## 手順

### 0. 承認処理（まず最初に）
`ideas/pending/` を走査し、人間が frontmatter の `status` を変更した項目を処理する：
- `approved` → `ideas/approved/` へ移動。プロト段階では「承認済み。次は spec-kit(B) でプロトタイプ化」と案内して、その企画の処理を終える。
- `rejected` → frontmatter/本文の却下コメントを**最優先インプット**として修正ループ（ステップ5）へ。コメントが「没でよい」主旨なら `ideas/boneyard/`（`killed_by: human`）へ。
- 変更が無ければ次へ。

### 1. seed 選定
- `$ARGUMENTS` があればそれを seed にする。
- 無ければ `config/seeds/` から、STATE.md の「消費済み seed」に無いものを1つ選ぶ。
- 種が一つも無ければ、ユーザーに seed を求めて**停止**する。

#### 二重起案ガード（無人運用・再発火対策）
seed を確定したら、起案に入る**前に** `STATE.md` の「消費済み seed」を必ず照合する：
- 既に消費済みの seed なら、**新サイクルを開始しない**。
  - ScheduleWakeup の名残・再発火など「継続中」を示唆する引数で呼ばれた場合も、対象サイクルが既に完了（pending/boneyard/approved 確定）していれば**何もせず停止**する。
  - 対話運用なら「消費済み」である旨を伝えて別 seed を促す。無人運用なら静かに終了する（同じ種を二度起案しない）。
- まだ消費していない seed のときだけ、ステップ2へ進む。

### 2. 起案
`ideator` サブエージェントに「seed」と「founder-profile に従え」を渡し、企画ドラフトを生成させる。

### 3. 足切り
`gatekeeper` サブエージェントにドラフトを渡し、全ゲートを判定させる。
- 総合 FAIL → `ideas/boneyard/` に `killed_by: gate:<name>` と理由を記録 → このサイクル終了（次は別 seed を促す）。
- 総合 PASS → 次へ。

### 4. ループ状態ファイルの作成
足切り PASS 後、この企画の作業状態を `.loop/runs/<YYYY-MM-DD-slug>.json` に作成する（スキーマは末尾）。
`round=1`、`status="scoring"`、`gates`、空の `axes` 履歴で初期化する。
> このファイルが「採点履歴」と「反証の解決状況」を保持する。critic はこれを読むことで、
> 毎回フルのやり取りを抱えずに前回の文脈を引き継げる（**コンテキスト節約**・セッションをまたいでも再開可・Codex等でも同一に動く）。

### 5. 採点（各軸ごとに独立起動）
`config/axes/` の軸ファイルの**数だけ** `critic` を起動する。各 critic に渡すもの：
- 担当する軸ファイルのパス
- 現在のドラフト
- **ループ状態ファイルのパス**

**round > 1（修正後の再採点）では、critic に次を厳守させる：**
- 状態ファイルから「自分の担当軸の前回スコア」と「前回自分が挙げた反証」を読む。
- 各反証について「今回のドラフトで解決されたか」を判定する。
- **解決済みの反証を理由に減点し直さない。** 点を下げてよいのは「新たに見つかった本物の欠陥」または「前回の手当てが生んだ新たな矛盾」のときだけ。
- 原則スコアは**単調に上がる**方向。下げる場合は明確な新規根拠を述べる。

**フォールバック**：critic が点数つきフォーマットを返さなければ、その軸だけ**一度だけ再起動**。それでも駄目ならユーザーに報告して停止。

採点後、状態ファイルを更新：各軸の `history` に `{round, score}` を追加、各軸の `resolved` / `open`（反証）を更新、`totals` に合計点を追加。

### 6. ゲート判定 / 修正ループ
- **全軸 ≥80** → `ideas/pending/` に企画を書き出し（下記フォーマット）、STATE.md を更新し、
  **人間の承認を待って停止（ハードストップ）**。ユーザーに「承認待ち1件」を伝える。
- **いずれか <80** → 各 critic の「潰すべき反証」を集約して `ideator` に渡し作り直させ、
  状態ファイルの `round` を +1 して **ステップ5へ戻る**。
  - 上限：**修正は最大3回**。
  - **停滞検知**：`totals` が前回から改善しない場合は早期打ち切り。
  - 打ち切り時 → `ideas/boneyard/` に `killed_by: max-revisions | stagnation` と
    全 round のスコア履歴・反証を記録して廃棄。

### 7. STATE 更新
最終実行時刻、消費した seed、結果（pending / boneyard / approved）を `STATE.md` に反映する。

### 8. ループ状態ファイルの後始末
完了した企画の `.loop/runs/*.json` の要点（スコア履歴・反証）は pending/boneyard の本文に転記済みなので、
作業ファイルは削除してよい（`.loop/` は `.gitignore` 済みの transient 領域）。

## ファイルフォーマット

### 承認待ち: `ideas/pending/YYYY-MM-DD-<slug>.md`
```
---
status: pending          # pending | approved | rejected
created: YYYY-MM-DD
seed: <seed>
revisions: <n>
gates:
  asset-light: PASS
scores:
  moat: NN
  finance: NN
  scale: NN
---
# <タイトル>

（ideator の企画本文）

## 検証サマリ
- moat: NN — 要点
- finance: NN — 要点
- scale: NN — 要点
```
> 承認方法（人間）：この frontmatter の `status` を `approved` か `rejected` に書き換える。
> `rejected` の場合は本文末尾に**修正コメントを必ず書く**（次サイクルの最優先インプットになる）。

### 没: `ideas/boneyard/YYYY-MM-DD-<slug>.md`
```
---
status: rejected
created: YYYY-MM-DD
seed: <seed>
killed_by: gate:asset-light | axis:<name> | stagnation | max-revisions | human
revisions: <n>
scores: { moat: NN, finance: NN, scale: NN }   # 採点まで進んだ場合
---
# <タイトル>

（企画の要旨）

## 没理由
- どの軸/ゲートで、何点で、なぜ落ちたかを**具体的に**（後で seed 昇格を判断するため）。
```

### ループ状態ファイル: `.loop/runs/<YYYY-MM-DD-slug>.json`
修正ループ中の作業メモリ。critic はこれを読んで前回の文脈を引き継ぐ（コンテキスト節約・再開可能・ツール非依存）。
```json
{
  "id": "2026-06-21-saiyo-koho",
  "seed": "PR × スタートアップ で 採用広報が回らない 課題",
  "created": "2026-06-21",
  "round": 2,
  "status": "scoring",            // scoring | revising | pending | boneyard | approved
  "gates": { "asset-light": "PASS" },
  "axes": {
    "moat": {
      "history": [ {"round": 1, "score": 52}, {"round": 2, "score": 61} ],
      "open":     ["（未解決の反証）", "..."],
      "resolved": ["（解決済みとみなした反証）", "..."]
    },
    "finance": { "history": [...], "open": [...], "resolved": [...] },
    "scale":   { "history": [...], "open": [...], "resolved": [...] }
  },
  "totals": [188, 181]            // round ごとの合計点（停滞検知に使う）
}
```
