# operator quickstart

**この repo で今日実際にできることは、gate を 1 本走らせることだけ**（手順 4・9）。
残りは「何が在って何が無いか」を自分の手で確かめる手順で、手順 0 から 9 までの 10 個ある。

10 手順とも 2026-08-09 に実際に踏んで、出力をそのまま貼ってある。**インストールは要らない**
—— `git` / `nbb` / `node` / `curl` / `dig` / `gh` / `python3` だけ。

前提として 2 つのパスを置く:

```bash
REPO=$HOME/github/com-junkawasaki/orgs/cloud-itonami/rare-earth-coverage
ROOT=$HOME/github/com-junkawasaki                       # superproject
```

`$REPO` は west の checkout。**手順 0 のとおり、そこが最新とは限らない。**

---

## 手順 0 — checkout が pin に居るのか main に居るのか

west の pin は「その時 manifest に書かれた commit」であって、upstream の最新ではない。
**この repo では、その差がちょうど「唯一走るもの」の有無になっている。**

```bash
grep -A3 'name: rare-earth-coverage' "$ROOT/manifest/west.yml" | grep revision
git -C "$REPO" rev-parse HEAD
gh api repos/cloud-itonami/rare-earth-coverage/compare/$(git -C "$REPO" rev-parse HEAD)...main \
  --jq '{status, ahead_by, behind_by}'
```

実測:

```
      revision: b812981f9ffd08526c879328c3e013e944e75d72
b812981f9ffd08526c879328c3e013e944e75d72
{"ahead_by":2,"behind_by":0,"status":"ahead"}
```

pin と checkout は一致しているが、**どちらも main より 2 commit 遅れている**
（`behind_by: 0` かつ `status: ahead` なので純 fast-forward。force-push ではない）。
何が足りないか:

```bash
gh api repos/cloud-itonami/rare-earth-coverage/compare/b812981f9ffd08526c879328c3e013e944e75d72...main \
  --jq '.files[] | "\(.status) +\(.additions) \(.filename)"'
```

```
added +198 src/rare_earth_coverage/murakumo.kotoba
```

**pin だけを見ると、この repo は「実行できるものが 1 行も無い descriptor」に見える。**
手順 4 で走らせる gate は、この 198 行のことである。

**したがって以降の手順は main で踏む。** 共有 checkout を書き換えないよう worktree を切り、
`$REPO` をそちらへ向け直す:

```bash
git -C "$REPO" fetch cloud-itonami main
git -C "$REPO" worktree add /tmp/rec-main cloud-itonami/main
REPO=/tmp/rec-main                                   # 以降これを使う
```

> remote 名は `origin` ではなく `cloud-itonami`（west の規約）。`origin/main` を渡すと
> `unknown revision` で落ちる。読み終えたら `git -C ~/github/com-junkawasaki/orgs/cloud-itonami/rare-earth-coverage worktree remove /tmp/rec-main`。

---

## 手順 1 — 後継が居ないことを確かめる（最初に確かめるべきこと）

「古い形だから捨ててよい」と「後継が居るから捨ててよい」は別。前者は移行の話、
後者は削除の話で、**この repo は前者であって後者ではない。**

```bash
grep -rn 'actor/supersedes' "$ROOT"/orgs/cloud-itonami/*/manifest.edn
grep -rl 'supersedes'       "$ROOT"/orgs/cloud-itonami/*/manifest.edn | wc -l
grep -rn 'supersedes'       "$ROOT"/orgs/cloud-itonami/*/manifest.edn | grep -c 'rare-earth-coverage'
```

```
…/business-manager/manifest.edn:13: :actor/supersedes "actor-manifest.jsonld (legacy T1 MCP-Compose / RisingWave-Cypher; deprecated, removed after 1 R-cycle)"
…/kamado/manifest.edn:11:           :actor/supersedes ["oil-refining"]
       7
0
```

**`:actor/supersedes` というキーを持つ manifest は艦隊全体で 2 本しかない。** うち repo を
名指しているのは `kamado` の 1 本（`["oil-refining"]`）で、`business-manager` の方は
「自分の legacy jsonld を置き換える」という自分自身についての散文。
**`rare-earth-coverage` を名指す行は 0。**

> 数え方の落とし穴: `supersedes` を素で grep すると 7 本出るが、5 本は非目標欄などの
> 散文（"no RisingWave — supersedes the legacy …"）で、**後継宣言ではない**。
> キー名 `actor/supersedes` で引くこと。

基層の方は退役が宣言されている:

```bash
grep -rl '2605262130' "$ROOT"/orgs/cloud-itonami/*/manifest.edn | wc -l   # → 19
ls "$ROOT"/90-docs/adr/ | grep 2605262130 | wc -l                          # → 0
ls "$ROOT"/90-docs/adr/ | wc -l                                            # → 2022
```

19 本の manifest が ADR-2605262130（kotoba Datom log が正本、RisingWave / Cypher は不可）を
引くが、**ADR 本体は superproject の `90-docs/adr/`（2,022 件）に無い**。引用だけが在る。

---

## 手順 2 — manifest の形を数える

```bash
cd "$REPO" && node -e '
const fs=require("fs"), buf=fs.readFileSync("actor-manifest.jsonld"), m=JSON.parse(buf.toString("utf8"));
let cy=""; for (const p of m.pipelines) for (const s of p.steps) if (s.args && s.args.template) cy+=s.args.template;
console.log("manifest", buf.length+"B  cypher", cy.length+"B  =",(100*cy.length/buf.length).toFixed(1)+"%");
const q="\x27";
const grab=(k)=>[...cy.matchAll(new RegExp(k+":"+q+"([^"+q+"]+)"+q,"g"))].map(x=>x[1]);
const paths=grab("\\{path"), ends=new Set([...grab("source"),...grab("target")]);
console.log("actor rows", paths.length, "| json actors", m.actors.length,
            "| 差分", paths.filter(p=>!m.actors.some(a=>a.path===p)).length);
console.log("flow rows", grab("\\{id").length, "| 端点", ends.size,
            "| dangling", [...ends].filter(e=>!paths.includes(e)).length,
            "| flow の付かない actor", paths.filter(p=>!ends.has(p)));
const hist=(k)=>{const o={}; for (const v of grab(k)) o[v]=(o[v]||0)+1; return o;};
console.log("stage", JSON.stringify(hist("stage")));
console.log("juris", JSON.stringify(hist("jurisdiction")));
console.log("kind",  JSON.stringify(hist("kind")));
console.log("pipelines", m.pipelines.length, "steps", m.pipelines.reduce((a,p)=>a+p.steps.length,0));
'
```

実測:

```
manifest 46794B  cypher 27548B  = 58.9%
actor rows 62 | json actors 62 | 差分 0
flow rows 66 | 端点 59 | dangling 0 | flow の付かない actor [ 'mineral:ndpr', 'mineral:dytb', 'intel:usgs' ]
stage {"taxonomy":8,"regulation":3,"mining":1,"extraction":5,"separation":10,"processing":7,
       "magnet-manufacturing":12,"policy":7,"finance":4,"intelligence":1,"demand":4}
juris {"GLOBAL":11,"CN":15,"MM":1,"US":13,"AU":6,"MY":1,"JP":5,"CA":1,"GB":2,"DE":1,"EU":4,"FR":2}
kind {"policy":8,"dependency":14,"regulation":2,"ownership":1,"resource_flow":34,
      "capital_flow":4,"offtake":1,"project":1,"governance":1}
pipelines 5 steps 12
```

**差分 0 / dangling 0 は本物の良い性質**。JSON の `actors` 配列は Cypher の完全な射影で、
66 本の flow はすべて実在の actor 同士を結ぶ。この seed は、指標を載せる土台としては
整っている。

> **`source:` を素で数えると 84 出る。正しくは 66。** 余分な 18 は子 DID の
> `did:web:…:resource:myanmar-ion-clays` の**尻尾**で、`re|source:` が引っかかる。
> 上のコマンドが引用符まで含めて `source:'…'` を要求しているのはこのため。緩めると
> **存在しない端点が 5 つ生えて「dangling 5」という嘘の結論**になる（この文書の草稿が
> 実際にそうなった）。**この種の誤りは両方向に出る** —— 緩い正規表現は多く数え、
> きつい正規表現は取りこぼす。数が動いたら必ず生テキストを見ること。

---

## 手順 3 — 名乗っている risk 機能が Cypher に 1 行も無いことを確かめる

`CLAUDE.md` と `riskAssessmentAxis` は解析中核を「多世代（子・孫）× wellbecoming の
リスク評価軸」と宣言している。その語彙を Cypher に対して数える:

```bash
cd "$REPO" && node -e '
const fs=require("fs"), src=fs.readFileSync("actor-manifest.jsonld","utf8"), m=JSON.parse(src);
let cy=""; for (const p of m.pipelines) for (const s of p.steps) if (s.args && s.args.template) cy+=s.args.template;
for (const t of ["risk","monopoly","chokepoint","generational","wellbecoming","resilience","concentration","HHI","score"])
  console.log("  "+t.padEnd(14)+"manifest="+String(src.split(t).length-1).padStart(3)+"  cypher="+(cy.split(t).length-1));
console.log("\n risk の 5 件がどこに居るか:");
for (const k of Object.keys(m)) { const n=JSON.stringify(m[k]).split("risk").length-1; if (n) console.log("   "+k+": "+n); }
'
```

実測:

```
  risk          manifest=  5  cypher=0
  monopoly      manifest=  1  cypher=0
  chokepoint    manifest=  1  cypher=0
  generational  manifest=  2  cypher=0
  wellbecoming  manifest=  1  cypher=0
  resilience    manifest=  1  cypher=0
  concentration manifest=  4  cypher=0
  HHI           manifest=  0  cypher=0
  score         manifest=  0  cypher=0

 risk の 5 件がどこに居るか:
   riskAssessmentAxis: 2
   pipelines: 1
   profile: 1
```

`pipelines` の 1 件が実質の全て。読む:

```bash
cd "$REPO" && node -e '
const m=JSON.parse(require("fs").readFileSync("actor-manifest.jsonld","utf8"));
const S={}; for (const p of m.pipelines) for (const s of p.steps) S[s.id]=s;
for (const id of ["coverageNodes","coverageFlows","coverageStages","coverageSummary","coverageSnapshot"])
  console.log("### "+id+" ("+S[id].fn+")\n"+JSON.stringify(S[id].args).slice(0,420)+"\n");
'
```

3 本とも `RETURN count(...)`、4 段目が `agent.chat` で
`"Return a concise summary of the largest concentration risks"`、5 段目がその自由文を
`SET c.summary = $coverageSummary.text` で格納する。**集中度の指標も閾値も無い。**

> **数え方の注意**: `split(t).length-1` は部分文字列を数えるので `risk` は
> `riskAssessmentAxis` にも当たる。**多めに出る方向に間違える**ので、0 という結果は
> 安全側（見落としではない）。逆に「0 でない」を根拠にするときは中身を読むこと。

---

## 手順 4 — gate を実際に走らせる（この repo で唯一動くもの）

```bash
cd "$REPO" && cat > /tmp/gate-check.cljs <<'EOF'
(require '[rare_earth_coverage.murakumo :as m])
(def all (into {} (map (fn [g] [g true]) m/common-gates)))
(def six (into {} (map (fn [g] [g true]) (butlast m/common-gates))))
(println "cells=" (count m/cell-specs) "gates=" (count m/common-gates))
(println "6/7 →" (frequencies (map (comp :status val) (m/all-cell-plans {:attestations six}))))
(println "7/7 →" (frequencies (map (comp :status val) (m/all-cell-plans {:attestations all :computed-at "t" :request-id "r"}))))
(println "effect ops ="
  (distinct (map :op (mapcat :effects (vals (m/all-cell-plans {:attestations all :computed-at "t" :request-id "r"}))))))
EOF
kbb --backend sci --classpath src /tmp/gate-check.cljs
```

実測:

```
cells= 12 gates= 7
6/7 → {:blocked 12}
7/7 → {:ready 12}
effect ops = (:mst/put-record)
```

**attestation を 1 つ抜くと 12 セル全部が `:blocked` になり、effect が 0 になる。**
deny-by-default は劇場ではない。

覆っている範囲を確かめる（12 セルは manifest の 12 の面と全単射）:

```bash
cd "$REPO" && cat > /tmp/map-cells.cljs <<'EOF'
(ns map-cells (:require [rare_earth_coverage.murakumo :as m] ["node:fs" :as fs] [clojure.string :as str]))
(def mf (js->clj (js/JSON.parse (fs/readFileSync "actor-manifest.jsonld" "utf-8"))))
(def cells (set (map name (keys m/cell-specs))))
(defn tail [s] (str/lower-case (last (str/split s #"\."))))
(def declared (set (concat (map tail (keep #(get-in % ["trigger" "nsid"]) (get mf "pipelines")))
                           (map tail (get-in mf ["triggers" "subscribeRepos" "collections"]))
                           (map tail (get mf "requiredCollections"))
                           (map str/lower-case (get mf "requiredLoops")))))
(println "cells" (count cells) "declared" (count declared)
         "| cell だけ" (sort (remove declared cells)) "| 宣言だけ" (sort (remove cells declared)))
(println "cron の step id ="
  (vec (mapcat (fn [p] (when (= "cron" (get-in p ["trigger" "type"])) (map #(get % "id") (get p "steps"))))
               (get mf "pipelines"))))
EOF
kbb --backend sci --classpath src /tmp/map-cells.cljs
```

```
cells 12 declared 12 | cell だけ () | 宣言だけ ()
cron の step id = [seedActors seedFlows seedDigest coverageNodes coverageFlows coverageStages coverageSummary coverageSnapshot]
```

**ここが要点**: セルは *collection と loop* の上に定義されていて、*pipeline step* の上には
無い。effect は `:mst/put-record` の 1 種類だけで、Cypher を実行する effect は無い
（cljc に Cypher は 1 文字も無い）。つまり **27,548 B の Cypher と 8 つの cron step は、
7 つのゲートのどれも通らない。** gate が守っているのは宣言された面であって、
実際に書かれるものではない。

---

## 手順 5 — 3 つの identity のうち、どれが解決するか

```bash
for u in https://rare-earth-coverage.etzhayyim.com/.well-known/did.json \
         https://etzhayyim.com/actor/rare-earth-coverage/did.json ; do
  printf '%-62s ' "$u"; curl -sS -o /tmp/d.json -w '%{http_code}\n' --max-time 12 "$u"
done
dig +short rare-earth-coverage.etzhayyim.com A ; echo "(空 = A レコード無し)"
```

```
https://rare-earth-coverage.etzhayyim.com/.well-known/did.json curl: (6) Could not resolve host: rare-earth-coverage.etzhayyim.com
000
https://etzhayyim.com/actor/rare-earth-coverage/did.json       200
(空 = A レコード無し)
```

`000` は「サーバが 000 を返した」ではなく「**接続に至らなかった**」。名前解決で止まっている。

**manifest / test / `murakumo.cljc` が名乗る DID にはホストが無い。** 解決するのは
repo の `.well-known/did.json` が持つ方だけ。ただしそこで返るのは repo の写しではない:

```bash
cd "$REPO" && curl -sS --max-time 12 https://etzhayyim.com/actor/rare-earth-coverage/did.json > /tmp/served.json
python3 - <<'EOF'
import json
a=json.load(open(".well-known/did.json")); b=json.load(open("/tmp/served.json"))
for k in sorted(set(a)|set(b)):
    if a.get(k)!=b.get(k):
        print(f"{k}\n  repo:   {json.dumps(a.get(k),ensure_ascii=False)[:180]}\n  served: {json.dumps(b.get(k),ensure_ascii=False)[:180]}")
EOF
```

`@context` の suite、PDS の宛先、`alsoKnownAs`、service の数、`_meta` —— 全部違う。
配信側は `"source": "kotoba"` と名乗り、`primaryLexicon` に
`com.etzhayyim.rare-earth-coverage` を持つ。**この repo の did.json を serve している者は
どこにも居ない**（serve するはずのホスト自身に DNS が無い）。

割れ目の出所は 1 つの commit:

```bash
git -C "$REPO" show --stat b812981 | tail -3
```

```
 .well-known/did.json | 8 ++++----
 1 file changed, 4 insertions(+), 4 deletions(-)
```

2026-07-02 に did.json だけが移行し、manifest も test も動かなかった。2 週間後に
追加された `murakumo.cljc` は**動かなかった方**を `actor-did` に焼き込んでいる。
lexicon 名前空間も同じ線で割れる（manifest は `com.etzhayyim.apps.rareEarth.*`、
cljc は `com.etzhayyim.rare-earth-coverage.*`）。

---

## 手順 6 — PDS に record があるか（無い）

```bash
D=did:web:etzhayyim.com:actor:rare-earth-coverage
curl -sS --max-time 15 "https://pds.aozora.app/xrpc/com.atproto.repo.describeRepo?repo=$D"; echo
curl -sS --max-time 15 "https://pds.aozora.app/xrpc/com.atproto.repo.listRecords?repo=$D&collection=com.etzhayyim.apps.rareEarth.actor&limit=3"; echo
curl -sS -o /dev/null -w 'pds.etzhayyim.com → %{http_code}\n' --max-time 15 https://pds.etzhayyim.com/xrpc/_health
```

```
{"did":"did:web:etzhayyim.com:actor:rare-earth-coverage","handle":"handle.invalid","collections":[],"handleIsCorrect":false}
{"records":[]}
pds.etzhayyim.com → 530
```

**PDS は DID を知っているが、collection も record も 0 件。** 5 本の pipeline は一度も
走っていない。なお repo の did.json が指す `pds.etzhayyim.com` は 530 を返し、実際に
DID を持っているのは配信側 doc が指す `pds.aozora.app`。

> これは弱い証拠であることに注意: 「この PDS に無い」は「どこにも無い」ではない。
> 断言できるのは、**DID doc が指す PDS に無い**ことまで。

---

## 手順 7 — 対になる appview と、3 通りの数

UI は別 repo（`orgs/cloud-itonami/rare-earth` = `com-etzhayyim-app-rare-earth`、nanoid は
同じ `re4c0v26`）に居る。

```bash
A="$ROOT/orgs/cloud-itonami/rare-earth/appview/rare-earth-ui-re4c0v26"
grep primaryActorDid "$ROOT/orgs/cloud-itonami/rare-earth/PROJECT.jsonld"
grep -A2 '"routes"' "$A/wrangler.jsonc"
dig +short rare-earth.etzhayyim.com A ; echo "(空 = A レコード無し)"
node -e '
const j=require("fs").readFileSync(process.argv[1],"utf8"), c=JSON.parse(j);
console.log("metrics:", JSON.stringify(c.metrics));
console.log("rows: actors",c.actors.length,"flows",c.flows.length,
            "juris",new Set(c.actors.map(a=>a.jurisdiction)).size,
            "| stageCoverage 合計", c.stageCoverage.reduce((a,s)=>a+s.count,0));
console.log("updatedAt:", c.updatedAt);
' "$A/svelte/static/rare-earth/coverage.json"
grep -c "fetch(" "$A/svelte/src/App.svelte"
```

```
"etzhayyim:primaryActorDid": "did:web:rare-earth-coverage.etzhayyim.com",
  "routes": [ { "pattern": "rare-earth.etzhayyim.com/*", "zone_name": "etzhayyim.com" } ]
(空 = A レコード無し)
metrics: {"actorCount":40,"flowCount":39,"jurisdictionCount":9,"activeDiversificationProjects":5}
rows: actors 20 flows 18 juris 10 | stageCoverage 合計 40
updatedAt: 2026-04-13T17:45:00Z
4
```

同じグラフが **3 通りに数えられている**:

| 出所 | actor | flow |
|---|---|---|
| この repo の seed（手順 2） | 62 | 66 |
| appview `coverage.json` の `metrics` | 40 | 39 |
| **同じファイル**の `actors` / `flows` 配列 | 20 | 18 |

`coverage.json` は自分自身とも食い違う（`stageCoverage` の合計 40 に対し行は 20）。
`src/coverage-data.ts`（21,630 B）が 4 つ目の写しを持つ。**どれも pipeline の出力ではない。**
`App.svelte` は起動時に `/xrpc/…listActors` などを 4 回 fetch するが、手順 6 のとおり
返るものは無く、route のホストには DNS が無い。

---

## 手順 8 — `.ts` テストが走らないことと、写すと 12/12 通ること

```bash
cd "$REPO" && ls package.json node_modules 2>&1 | head -2; grep -c 'it(' actor-manifest.test.ts
ls "$ROOT"/orgs/cloud-itonami/*/actor-manifest.test.ts | wc -l
for d in "$ROOT"/orgs/cloud-itonami/*/; do
  [ -f "$d/actor-manifest.test.ts" ] && [ -f "$d/package.json" ] && echo "$d"; done | wc -l
```

```
ls: node_modules: No such file or directory
ls: package.json: No such file or directory
12
      17
       0
```

**この repo だけの問題ではない** —— `actor-manifest.test.ts` を持つ 17 本のうち、
`package.json` を持つものは 0 本。**艦隊の 17 本とも一度も走っていない。**

12 個の assertion を nbb に写して走らせる（vitest も npm install も要らない）:

```bash
cd "$REPO" && cat > /tmp/check12.cljs <<'EOF'
(ns check12 (:require [clojure.string :as str] ["node:fs" :as fs]))
(def m (js->clj (js/JSON.parse (fs/readFileSync "actor-manifest.jsonld" "utf-8"))))
(def VP #{"graph.query" "graph.write" "graph.vectorSearch" "agent.chat" "agent.invoke"
          "identity.resolve" "browser.fetch" "signal.encrypt" "consent.check"
          "derive:social" "dmn.evaluate" "form.collect"})
(def n-pass (atom 0)) (def n-fail (atom 0))
(defn chk [label ok? detail]
  (if ok? (do (swap! n-pass inc) (println "  ok   " label))
          (do (swap! n-fail inc) (println "  FAIL " label "—" detail))))
(def pipes (get m "pipelines"))
(defn by-cron [c] (first (filter #(= c (get-in % ["trigger" "cron"])) pipes)))
(chk "@context valid" (= (get m "@context") "https://etzhayyim.com/ns/actor/v1") (get m "@context"))
(chk "DID valid" (= (get m "@id") "did:web:rare-earth-coverage.etzhayyim.com") (get m "@id"))
(chk "runtime" (= (get m "runtime") "k8s-langserver") (get m "runtime"))
(chk "nanoid" (= (get m "nanoid") "re4c0v26") (get m "nanoid"))
(chk "capabilities valid" (every? VP (get m "capabilities")) (get m "capabilities"))
(chk "no fn:custom" (not-any? #(= "custom" (get % "fn")) (mapcat #(get % "steps") pipes)) "")
(chk "five pipelines" (= 5 (count pipes)) (count pipes))
(let [st (get (by-cron "15 */6 * * *") "steps")]
  (chk "seed cron: actors → flows → digest"
       (and (= 3 (count st)) (= "seedActors" (get (nth st 0) "id"))
            (= "seedFlows" (get (nth st 1) "id")) (= "derive:social" (get (nth st 2) "fn")))
       (mapv #(get % "id") st)))
(let [st (get (by-cron "0 */6 * * *") "steps")]
  (chk "coverage cron: nodes/flows/stages/summary/snapshot"
       (and (= 5 (count st)) (= "coverageStages" (get (nth st 2) "id"))
            (= "coverageSummary" (get (nth st 3) "id")) (= "coverageSnapshot" (get (nth st 4) "id")))
       (mapv #(get % "id") st)))
(let [nsids (set (keep #(get-in % ["trigger" "nsid"]) pipes))]
  (chk "xrpc covers get / listActors / listFlows"
       (every? nsids ["com.etzhayyim.apps.rareEarth.coverage.get"
                      "com.etzhayyim.apps.rareEarth.coverage.listActors"
                      "com.etzhayyim.apps.rareEarth.coverage.listFlows"]) nsids))
(chk "actor set ≥ 30" (>= (count (get m "actors")) 30) (count (get m "actors")))
(chk "isBot true" (= true (get-in m ["profile" "isBot"])) (get-in m ["profile" "isBot"]))
(println (str "\n" @n-pass " passed, " @n-fail " failed"))
EOF
kbb --backend sci /tmp/check12.cljk
```

```
  ok    @context valid
  ok    DID valid
  ...
12 passed, 0 failed
```

**12/12 通る。だから安心してよい、ではない。** 2 番目の assertion
（`it("DID valid")`）は `did:web:rare-earth-coverage.etzhayyim.com` を**正しい値として
固定している** —— 手順 5 で A レコードが無いことを確かめた、あの DID である。
この repo で唯一形を守っている gate が、壊れた identity を守っている。

**したがって identity を直すと、この test は落ちる。それが正しい落ち方**であり、
落ちたときに `package.json` が無くて走らせられない、という状態が今日の姿。

---

## 手順 9 — gate が本当に落ちることを確かめる（信じる前に壊す）

手順 4 の出力（`{:blocked 12}` / `{:ready 12}`）は、**gate が実際には何も見ていなくても
同じように見える可能性がある**。確かめる:

```bash
cd "$REPO" && cat > /tmp/gate-falsify.cljs <<'EOF'
(require '[rare_earth_coverage.murakumo :as m])
(def all (into {} (map (fn [g] [g true]) m/common-gates)))
;; 1) 7 つのうち 1 つずつ落として、必ず :blocked になるか
(println "1 つ落とすと blocked になるゲート数 ="
  (count (filter (fn [g]
                   (= :blocked (:status (m/cell-plan :get {:attestations (dissoc all g)}))))
                 m/common-gates)) "/" (count m/common-gates))
;; 2) 値が false のときも通らないか（キーの有無ではなく値を見ているか）
(println "値 false → " (:status (m/cell-plan :get {:attestations (assoc all (first m/common-gates) false)})))
;; 3) 無い cell を頼むと落ちるか
(println "未知の cell → "
  (try (m/cell-plan :no-such-cell {:attestations all}) (catch :default e (ex-message e))))
EOF
kbb --backend sci --classpath src /tmp/gate-falsify.cljs
```

```
1 つ落とすと blocked になるゲート数 = 7 / 7
値 false →  :blocked
未知の cell →  unknown cell
```

7 つのゲートは**どれも単独で効く**（1 つでも欠けると止まる）。キーの存在ではなく値を
見ている。未知の cell は握り潰さずに落ちる。**この 1 本だけは、信用してよい。**

---

## ここから先に進みたい場合

順番に意味がある:

1. **west の pin を main まで進める**（手順 0）。進めないと、以降の全部が「無い」ものとして
   計測される。`kbb --backend sci scripts/gen-west-manifest.cljk --entry rare-earth-coverage`。
2. **DID をどちらかに決める**（手順 5）。決めると `it("DID valid")` が落ちるので、
   同時に `package.json` を入れて落ちることを見る（手順 8）。
3. **Cypher の宛先を決める**。受ける graph store を用意するか、ADR-2605262130 を引く
   19 本と同じく kotoba Datom log 上に seed を書き直すか。後者なら `graph.write` と
   `agent.chat` の 2 capability が消え、gate の外を通っていた 27,548 B が gate の中に入る。
4. **`coverageSummary` を計算に置き換える**（手順 3）。62 actor / 66 flow / 12 法域は
   参照整合が取れているので、HHI なり単一源依存フラグなりを載せる土台はもう在る。
   載っていないのは指標であって、データではない。
5. **appview の 3 つの数を 1 つにする**（手順 7）。62/66 が seed の値で、40/39 と 20/18 は
   出所が無い。
