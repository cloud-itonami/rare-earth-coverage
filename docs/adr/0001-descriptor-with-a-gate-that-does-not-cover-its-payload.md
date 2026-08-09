# ADR-0001 — rare-earth-coverage は「gate が payload を覆っていない descriptor」である

- **Status**: accepted
- **Date**: 2026-08-09
- **Scope**: `cloud-itonami/rare-earth-coverage`（superproject `com-junkawasaki/root` の west project）

## Context

この repo は 2026-06-24 に `etzhayyimcojp/20-actors` から切り出された actor snapshot
（`c301e8a`）で、以降 3 commit しか動いていない。README が無く、**何が在って何が無いかを
述べる場所が repo 内に存在しなかった。**

同じ形の兄弟が艦隊に 38 本ある（`actor-manifest.jsonld` を持つ repo は計 39 本）。
**それらの結論をここへ転写すると誤る。** 実測して分かった差は下記のとおりで、特に
「gate は在るが payload を覆っていない」という形は、隣の `oil-shipping`（gate が在り、
descriptor は未実装）とも `oil-refining`（`kamado` が supersede 済み）とも違う。

## Decision

**この repo を「実装が無い descriptor」でも「supersede 済みの残骸」でもなく、
『seed は整っており、gate も閉まるが、gate が seed を覆っていない descriptor』として
記述する。** 根拠（すべて 2026-08-09 実測。手順は `docs/operator-quickstart.md`）:

1. **後継が居ない。** `:actor/supersedes` というキーを持つ manifest は `cloud-itonami`
   1,769 repo 中 2 本だけ（`kamado` = `["oil-refining"]` と `business-manager` の自己言及）で、
   **この repo を名指す行は 0**。一方、この repo が全面的に依拠する RisingWave / Cypher /
   `graph.write` は ADR-2605262130 を引く 19 本の manifest が「legacy・非正本」と宣言している。
   **基層は退役宣言済み、しかし後継は不在**という中間状態。

2. **seed は参照整合が取れている。** 62 の actor 行は JSON の `actors` 配列と過不足なく
   一致し、66 の flow は 59 の端点すべてが実在の actor を指す（dangling 0）。11 stage /
   12 法域 / 33 role / 14 relation。**これは捨てるべきものではなく、指標を載せる土台。**

3. **名乗っている解析中核が存在しない。** `riskAssessmentAxis` は多世代（子・孫）×
   wellbecoming のリスク評価を宣言するが、`risk` `monopoly` `chokepoint` `generational`
   `wellbecoming` `resilience` `concentration` `HHI` `score` は 27,548 B の Cypher に
   **1 度も現れない**。実体は 3 本の `RETURN count(…)` を LLM プロンプトに貼り、返った
   自由文を `c.summary` に格納する `agent.chat` 1 段。指標も閾値も無い。

4. **gate は本当に閉まるが、payload はその外を通る。** `murakumo.cljc` の 12 セルは
   manifest が宣言する 12 の面と全単射で、7 ゲートは 1 つ欠けるだけで 12 セル全部を
   `:blocked` にする（7/7 のゲートが単独で効くことを確認済み）。**しかし `cell-plan` が
   出す effect は `:mst/put-record` のみで、Cypher を実行する effect は無い** ——
   27,548 B の Cypher と 8 つの cron step はどのゲートも通らない。

5. **identity が 3 つに割れている。** manifest / test / cljc が名乗る
   `did:web:rare-earth-coverage.etzhayyim.com` には A レコードが無い。解決するのは
   `.well-known/did.json` の `did:web:etzhayyim.com:actor:rare-earth-coverage` だけだが、
   **そこで配信される doc は repo の写しと別物**（`@context` の suite・PDS・`alsoKnownAs`・
   service・`_meta` が全て相違、配信側は `source: "kotoba"`）。割れ目は `b812981`
   （2026-07-02、did.json だけを 4 行変更）に由来し、2 週間後の `murakumo.cljc` は
   解決しない方を焼き込んだ。lexicon 名前空間も同じ線で割れる。

6. **一度も走っていない。** PDS（`pds.aozora.app`）は DID を知っているが collections も
   records も 0 件。対になる appview（`cloud-itonami/rare-earth`）は同じグラフを
   62/66・40/39・20/18 と **3 通りに数え**、その `coverage.json` は自分自身とも
   食い違う（`stageCoverage` 合計 40 に対し行数 20）。route の
   `rare-earth.etzhayyim.com` にも DNS が無い。

7. **形を守る唯一の gate が、壊れた identity を守っている。** `actor-manifest.test.ts` の
   12 assertion は nbb に写すと 12/12 通るが、その 2 番目が「解決しない DID」を正しい値として
   固定している。そして `package.json` が無いので走らない —— `actor-manifest.test.ts` を持つ
   艦隊 17 本のうち `package.json` を持つものは **0 本**。

## Consequences

- **この repo を「空」と呼ばない。** 走るものが 1 本あり（gate）、整合の取れた seed がある。
  空なのは gate と seed の間。
- **west の pin（`b812981`）は main より 2 commit 遅れており、その差分は
  `murakumo.cljc` の 198 行だけ。** pin を進めない限り、成熟度計測は
  「この repo で唯一走るもの」を存在しないものとして数える。この ADR と同じ着地で pin を
  進める。
- **identity を直すと `it("DID valid")` が落ちる。それが正しい落ち方。** 直すときは同時に
  `package.json` を入れ、落ちることを確認してから直す。
- 次に進む順序は `docs/operator-quickstart.md` の末尾に置いた（pin → DID → Cypher の宛先 →
  指標 → appview の数）。

## Alternatives considered

- **「legacy jsonld なので supersede 済みとして扱う」** — 却下。実測すると後継は居ない
  （決定 1）。基層の退役宣言と、この repo の後継の不在は別の事実であり、混ぜると
  「捨ててよい」という誤った結論になる。
- **同時に identity を直す / `package.json` を入れる** — 却下。この反復の対象は 1 軸
  （docs）で、identity 修正は test を落とす変更を伴う。**落ちることを確認できる状態を
  先に作る**方が順序として正しく、その順序自体を quickstart に書いた。
- **`coverageSummary` を計算に置き換える** — 却下（今回は）。土台（決定 2）は在るので
  やる価値はあるが、宛先グラフが未確定なまま指標を書くと二重に捨てることになる。
