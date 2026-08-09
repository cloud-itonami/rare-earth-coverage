# rare-earth-coverage

レアアース／重要金属のバリューチェーンを**観測する**と名乗る actor の repo。
`cloud-itonami/rare-earth-coverage`（superproject `com-junkawasaki/root` の west project）。

**在るのは 3 つ**: 62 の actor と 66 の flow を Cypher で書いた `actor-manifest.jsonld`、
それを 12 セル × 7 ゲートで包む `src/rare_earth_coverage/murakumo.cljc`、走らない
`actor-manifest.test.ts`。**無いのは、それを実行する者**。

この README の数値はすべて 2026-08-09 に実測した。**手順は
[`docs/operator-quickstart.md`](docs/operator-quickstart.md) にあり、10 手順すべて
コピーして踏める**。決定の記録は [`docs/adr/0001-descriptor-with-a-gate-that-does-not-cover-its-payload.md`](docs/adr/0001-descriptor-with-a-gate-that-does-not-cover-its-payload.md)。

| | |
|---|---|
| 名乗る DID | `did:web:rare-earth-coverage.etzhayyim.com` — **A レコードが無い** |
| 解決する DID | `did:web:etzhayyim.com:actor:rare-earth-coverage` — 200。ただし**この repo の写しとは中身が違う** |
| PDS | `pds.aozora.app` は DID を知っているが **collections `[]` / records `[]`** |
| 走るもの | `murakumo.cljc` の gate 1 本（12 セル × 7 ゲート） |
| 走らないもの | `actor-manifest.test.ts`（`package.json` が無い）、5 本の pipeline すべて |
| 後継 | **居ない**（`:actor/supersedes` でこの repo を名指す manifest はゼロ） |

## 「coverage」を計算しているのは COUNT 3 本と LLM プロンプト 1 個

`CLAUDE.md` と manifest の `riskAssessmentAxis` は、この actor の解析中核を
「多世代（子・孫）× wellbecoming のリスク評価軸」と宣言し、①独占／チョークポイント
依存と②不可逆な多世代環境リスクを測って **resilience / de-monopolization /
restoration** へ routing すると書いている。

**その語彙は 27,548 バイトの Cypher に 1 度も出てこない。**

```
risk 0   monopoly 0   chokepoint 0   generational 0   wellbecoming 0
resilience 0   concentration 0   HHI 0   score 0
```

manifest 全体で `risk` は 5 箇所、すべて散文（`riskAssessmentAxis` に 2、`profile` に 1、
`pipelines` に 1）。最後の 1 つが実質の全てで、6 時間ごとの cron の 4 段目に居る:

```
coverageNodes   MATCH (n:RareEarthActor) … RETURN count(n)
coverageFlows   MATCH ()-[r:RARE_EARTH_FLOW]->() … RETURN r.kind, count(r)
coverageStages  MATCH (n:RareEarthActor) … RETURN n.stage, count(n)
coverageSummary agent.chat ← 上の 3 つを貼り付けて
                「largest concentration risks を要約せよ」
coverageSnapshot MERGE (c:RareEarthCoverageSnapshot …) SET c.summary = $coverageSummary.text
```

集中度は**計算されない**。件数を 3 通りに数えて LLM に渡し、返ってきた自由文を
`c.summary` に格納する。指標も、閾値も、しきい値を跨いだときの挙動も無い。

**これは「壊れている」ではなく「まだ書かれていない」。** 62 の actor と 66 の flow は
参照整合が取れていて（下記）、HHI なり輸出依存度なりを載せる土台としては使える。
無いのは載せる側であって、載せる場所ではない。

## seed そのものは、この repo で一番まともな部分

Cypher は 46,794 B の manifest の **58.9%（27,548 B）** を占める。中身は 2 本の
`UNWIND … MERGE`（`seedActors` / `seedFlows`）と 1 本の snapshot `MERGE`、それに 3 本の
`MATCH` クエリだけ。実測した形:

| | |
|---|---|
| actor 行 | 62（JSON の `actors` 配列 62 件と **path が完全一致**、過不足ゼロ） |
| flow 行 | 66、端点 59 種、**dangling ゼロ**（存在しない actor を指す flow は 1 本も無い） |
| flow の付かない actor | 3（`mineral:ndpr` / `mineral:dytb` / `intel:usgs`） |
| stage | 11 — magnet-manufacturing 12 / separation 10 / taxonomy 8 / processing 7 / policy 7 / extraction 5 / finance 4 / demand 4 / regulation 3 / mining 1 / intelligence 1 |
| 法域 | 12 — CN 15 / US 13 / GLOBAL 11 / AU 6 / JP 5 / EU 4 / GB 2 / FR 2 / CA 1 / DE 1 / MM 1 / MY 1 |
| role | 33 |
| flow relation / kind | 14 種 / 9 種（resource_flow 34・dependency 14・policy 8 が大半） |
| 子 DID | 43 種（`deps:` に 78 回出現） |

**43 の子 DID は全部、解決しない親 DID の下に生えている**（`did:web:rare-earth-coverage.etzhayyim.com:operator:mp-materials` 等）。

## 名前が 3 つに割れていて、割れ目は 1 つの commit にある

```
manifest @id / test / murakumo.cljc  did:web:rare-earth-coverage.etzhayyim.com   → A レコード無し
.well-known/did.json                 did:web:etzhayyim.com:actor:rare-earth-coverage → 200
```

`b812981`（2026-07-02、"migrate did:web to etzhayyim.com scheme"）が変更したのは
**`.well-known/did.json` の 4 行だけ**。manifest も test も動かなかった。2 週間後の
`11c7fd6` で追加された `murakumo.cljc` は、動かなかった方（解決しない DID）を
`actor-did` として焼き込んでいる。

さらに、**解決する方の DID を引いても、この repo の写しは返ってこない**:

| | repo の `.well-known/did.json` | 実際に配信されている doc |
|---|---|---|
| `@context` | ed25519-2020 | jws-2020 |
| PDS | `pds.etzhayyim.com`（**530**） | `pds.aozora.app`（**200**） |
| `alsoKnownAs` | 4 件 | `[]` |
| service | 2 件 | `atproto_pds` + `xrpc-libp2p`（libp2p multiaddr） |
| `_meta` | 無し | `source: kotoba`, `primaryLexicon: com.etzhayyim.rare-earth-coverage` |

配信元は別系（`source: "kotoba"`）で、**この repo の did.json を serve している者は
どこにも居ない**（serve するはずのホスト自身に DNS が無い）。

lexicon 名前空間も同じ線で割れている —— manifest の collection は
`com.etzhayyim.apps.rareEarth.*`、`murakumo.cljc` の `collection` 関数が組み立てるのは
`com.etzhayyim.rare-earth-coverage.*`。**cljc は配信されている DID doc 側に付き、
pipeline は jsonld 側に付いている。**

## gate は本当に閉まる。ただし payload はその外を通る

`src/rare_earth_coverage/murakumo.cljc`（198 行、依存は `clojure.string` のみ）。
実測（`nbb --classpath src`）:

```
7 ゲート中 6 つを与える  → 12 セル全部 :blocked / effects 0
7 つ与える               → 12 セル全部 :ready
```

deny-by-default は劇場ではない。12 セルは manifest が宣言する 12 の面と**全単射**で対応する
（xrpc NSID 3 + `triggers.subscribeRepos` collection 3 + `requiredCollections` 2 +
`requiredLoops` 4）。過不足ゼロ。

**しかし `cell-plan` が出す effect は `:mst/put-record` の 1 種類しかない。** Cypher を
実行する effect も、`graph.write` を表す effect も無い（cljc 内に Cypher は 1 文字も無い）。
つまり **27,548 B の Cypher と 2 本の cron pipeline は、7 つのゲートのどれも通らない。**
gate が覆っているのは「この actor が持つと宣言した面」であって、「この actor が実際に
書くもの」ではない。

## 対になる appview は別 repo に居て、数が合っていない

`orgs/cloud-itonami/rare-earth`（`com-etzhayyim-app-rare-earth`、nanoid は同じ `re4c0v26`）が
UI 側。その `PROJECT.jsonld` はこの actor の DID を `primaryActorDid` として名指し、
`wrangler.jsonc` は `rare-earth.etzhayyim.com/*` を route に取る（**このホストにも DNS が無い**）。

`App.svelte`（15,294 B）は起動時に `/xrpc/…listActors` `/xrpc/…listFlows`
`/api/rare-earth/coverage` を fetch する。PDS の records は 0 件なので、返るものは無い。

同じグラフの数え方が **3 通り**ある:

| 出所 | actor | flow | 法域 |
|---|---|---|---|
| この repo の seed | **62** | **66** | 12 |
| appview `coverage.json` の `metrics` | **40** | **39** | 9 |
| 同じファイルの `actors` / `flows` 配列 | **20** | **18** | 10 |

`coverage.json` は**自分自身と食い違っている**（`stageCoverage` の合計は 40 で、
同梱の行数 20 と合わない）。さらに `src/coverage-data.ts`（21,630 B）が 4 つ目の写しを持つ。
**どれも pipeline が生成したものではない** —— pipeline は一度も走っていない。
`updatedAt` は 2026-04-13。

## 兄弟 39 本の中でのこの repo の位置（ローカル checkout 実測）

| | 件数 |
|---|---|
| `cloud-itonami` の repo | 1,769 |
| うち `actor-manifest.jsonld` を持つ（この repo と同じ legacy 形） | **39** |
| うち `manifest.edn`（EDN 正本形）も持つ | **0** — 2 形式は共存しない。移行は置換 |
| `manifest.edn` を持つ repo | 106 |
| `actor-manifest.test.ts` を持つ | 17 |
| **うち `package.json` を持つ** | **0** —— 17 本とも一度も走っていない |
| `src/*/murakumo.cljc` を持つ | 85 |
| README.md が無い repo | 153（この repo はこの README でそこから抜ける） |

`kamado` の非目標 N6 と `business-manager` の見出しは、この repo が全面的に依拠している
**RisingWave / Cypher / `graph.write` を「legacy・deprecated」と名指し**、kotoba Datom log を
正本とする ADR-2605262130 を引く（`cloud-itonami` の manifest 19 本が同じ ADR を引用）。
**その ADR ファイル自体は superproject の `90-docs/adr/`（2,022 件）に無い。**

つまりこの repo は「**基層が艦隊全体で退役宣言された、しかし後継の居ない descriptor**」であって、
supersede 済みの残骸ではない。`:actor/supersedes` というキーを持つ manifest は艦隊に 2 本しか
無く（`kamado` = `["oil-refining"]` と `business-manager` の自己言及）、**この repo を名指す行は
0 本**。

## pin が遅れていると、gate は「無い」ことになる

west の pin は `b812981`（2026-07-02）で、remote main `a0cdfe9` より **2 commit 遅れている**
（`ahead_by: 2 / behind_by: 0` の純 fast-forward）。差分は
`src/rare_earth_coverage/murakumo.cljc` の 198 行**だけ** —— つまり pin だけを見ると、
**この repo で唯一走るものが存在しないように見える。**

成熟度計測（ADR-2608052000）はローカル checkout を読むので、pin を進めない限り
`axis-substrate` は 0 のままになる。

## 検証していないこと

- `runtime: "k8s-langserver"` の実体がどこかに在るか（探していない）
- Cypher の宛先グラフ（`RareEarthActor` / `RARE_EARTH_FLOW` を持つ store）が在るか
- appview worker が Cloudflare に deploy 済みか（DNS が無いので外からは判定不能。
  アカウント側は見ていない）
- 43 の子 DID を読む者が居るか
- 62 の actor 行の**内容**が正しいか（形と参照整合は測った。事実性は測っていない）

## この repo を動かしたい場合、外に要るもの

1. **Cypher を受ける graph store**、または（艦隊の方針どおりなら）Cypher を捨てて
   kotoba Datom log 上に seed を書き直すこと。ADR-2605262130 を引く 19 本はそちら。
2. **どちらの DID を正とするか の決着**。解決するのは `did:web:etzhayyim.com:actor:*` の
   方だが、manifest / test / cljc の 3 つは反対側を指している。片方を直すと `it("DID valid")`
   が落ちる —— **それが正しい落ち方**。
3. **`package.json`**。12 個の `it(` は 1 度も走っていない。nbb に写して走らせると
   12/12 通る（手順 8）—— つまり test は形を固定してはいるが、**固定している形の中に
   解決しない DID が含まれている。**
4. **appview 側の 3 つの数の統一**。62/66 が seed の値で、40/39 と 20/18 は出所が無い。
