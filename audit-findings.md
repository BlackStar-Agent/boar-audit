# 审计发现：BOAR / LabRat 研究记录（poidh 赏金 #1520）

审计范围：`boar-research` v1.1.2（commit `6b7caca`，含 errata E1-E8）、`boar-bench`（含 commit `b41a25a` 的两个修复）、`labrat-boar-fork`（含 5 个修复 commit）。所有引用都给出文件与行号/复算过程；HF traces 数据集在本次环境不可达（DNS 污染），故全部结论基于仓库内公开代码、冻结文档与聚合数据，均可本地复现。已对照 `docs/ERRATA-BOUNTY-1520.md`（E1-E8，claims 9221-9336）与 boar-bench/labrat 两个仓库的全部赛后修复 commit，确认以下发现**均未被已有 claim 覆盖**。

---

## 发现 1（主发现）：支撑"reading binds（阅读是瓶颈）"结论的 ORACLE-REAL 实验，其 30 个条目中 23 个（77%）在干预前事实就已保留在上下文里——该实验对"交付 vs 阅读"这个问题大部分是退化的（degenerate）；而同一记录里功效更高的 SEFIX 2×2 分解明确显示"把缺失的事实交付进上下文"每条目 +0.077（CI 低点 +0.003，排除零）。两条结论在档案里并存且从未互相引用。

**证据链（全部本地可验证）：**

1. **ORACLE-REAL 的 KILL 判定**（`rnd/research/coordinator/arms/oracle-real/ORACLE-REAL.md`）：
   - 干预内容：dU oracle − control = **+0.015 [-0.029, +0.059]**（n 30），判定 KILL："the binding stage on real items is **reading**, retrieval/keep levers have a real-item ceiling of about +0.015"。
   - 同文件 "Post-hoc leads" 一节自己写明：**"Fact already kept (23): +0.008. Fact not kept in K3 before (7): +0.037 [-0.136, +0.155]"**。
   - "Caveats" 一节写明：**"The 23 'already kept' items had the fact in context before; on them the arm only moves it to [1] in full text."**
   - 冻结的 arm 定义（`ORACLE-criterion.md` "Arm (frozen)"）：**"if it was already kept it is moved there"** —— 即对这 23 条，干预 = 把已在上下文中的事实移到第 [1] 位。
   - 条目选取规则（`ORACLE-criterion.md` "Items (frozen)"）：keyed v0.1 条目、gold chunk 在 K3 包中 —— 注意这个规则**并不要求**事实未被保留，只是事后才发现 23/30 已保留。

2. **SEFIX-124 的 2×2 分解**（`rnd/research/coordinator/arms/coverage-map/sefix/answers-sefix124.md`，"chunk-level 2×2 (pooled)" 表）：
   - **B0_A__X_P（对照缺失→实验交付，n 18）：dU +0.0773 [+0.0028, +0.1529]，CI 排除零**，correctness 2.028→2.569。
   - SEFIX-ALL 的同型格子（B0_A__F_P，n 21）：dU **+0.0918 [+0.0241, +0.1615]**。
   - B0_P__X_P（两边都在，n 17）：dU −0.0004 —— **与 ORACLE-REAL 的 23 条"already kept"格子（+0.008）完全一致**：重排序不移动分数。
   - 也就是说：ORACLE-REAL 判 KILL 所依据的总体 +0.015，是"重排序格子（约 0）×23/30 + 交付格子（+0.037，n=7 功效不足）×7/30"的加权混合；它**从未按 kept/not-kept 分层报告主判定**。

3. **同一周内方向相反的第三项证据**（QFORM secondary，`rnd/research/coordinator/arms/qform/QFORM-RESULTS.md`）：gold chunk 保留时 U 0.88（n 43），未保留时 U 0.19（n 147）——但该 0.88 有方法学折扣（reference = item `answer`、rubric = item `rubric`，即参照答案与评分细则都出自同一 pack-fact gold，见其 Setup 段）。

4. **档案内部的不一致状态**：
   - `data/facts.json` 中 F-027（2026-10-13，标记 **CANONICAL**，来自 180 条自测的 ORACLE-ST）：claim 为 **"handing the model the exact fact-bearing chunk shows the headroom is retrieval, not reading"** —— 从未被 superseded。
   - F-2141（ORACLE-REAL/CHUNKCLEAN/DILUTION-K 的汇总 null-result）：caveat 为 **"reading capacity binds"**。
   - F-1958（ORACLE-REAL 行）：caveat 为 "arm verdict: KILL (reading binds)"。
   - 三者并存，无 `superseded_by`/互引；`grep -rln "reading binds" arms/coverage-map/` 为空，SEFIX 系列文件从不引用 ORACLE-REAL，反之亦然。
   - 后续决策确实建立在这个判定上：QUEUE.md 10-21 之后检索侧仅剩 0-LLM replay 线（RETRIEVE-SWEEP 等），SEFIX-124 汇总 +0.013 被判 "not a candidate"（其稀释原因正是只有 18/97 条目改变，见第 5 点）。

**为什么重要：**

"reading binds vs retrieval binds" 是这份研究最重要的机制性结论——它决定了剩余预算投给"换更大的模型/读得更准"还是"把事实送进上下文"。最终档案的 headline 机制结论（F-2141 "reading capacity binds"）由一个 77% 条目退化的实验支撑；而记录自己拥有的、样本量更大的分层证据（SEFIX 2×2，n 18，CI 排除零）指向相反方向。按 SEFIX 格子估算，若交付覆盖率从 B0 的水平提到 SEFIX 水平，97 条真实条目上可期望约 +0.013~+0.018 的总体 U 提升（18/97 × +0.077），这与 SEFIX 两种子实测 +0.013 吻合——**是正贡献，只是被 79% 未改变条目稀释后被判"not a candidate"**。判定规则（汇总 dU ≥ +0.03）把"每条目有效但覆盖面窄"的干预与"无效"干预混为一谈。

**建议的修复/验证（可复现步骤）：**

1. `python3 - <<'EOF'` 读取 `arms/coverage-map/sefix/answers-sefix124.md` 的 2×2 表与 `arms/oracle-real/ORACLE-REAL.md` 的 Post-hoc 段，逐行核对上面引用的数字。
2. 重算 ORACLE-REAL 判定在"not-kept"层的功效：n=7、观测 +0.037、其 CI [-0.136,+0.155] —— 按该 CI 反推，n=7 根本无法区分 +0.037 与 SEFIX 的 +0.077；即两实验在交付格子上**相容**，矛盾只在"已保留格子"被当成了主判定。
3. 一次低成本的判决实验：在 v0.1/v0.4 的"事实未保留"条目上（FACTPRESENT 的 not-kept 子集，约 40-60 条）跑 SEFIX-124 交付 arm vs 对照，Haiku N=4 双序（约 200-500 calls），对**该子集**报告 dU；若复现 +0.05~+0.10，则 F-2141 应改写为"交付有效但覆盖窄；阅读只在事实已在上下文时绑定"。

---

## 发现 2：frozen ruler 规范为消除"同家族出题+写参照+当裁判"偏倚而强制要求的 judge×writer 交叉表（以及 time-sensitive in/excluded 等行）从未产出；两个 judge 家族与各自同家族参照配对（G1=Opus×Claude refs、G2=Codex×Codex refs），使 headline 的 0.165 vs 0.242 之差无法把"裁判宽严"与"参照家族被自家裁判抬分"分开——记录自己引用的文献量级（3.4-8.4pp）恰好与这个差同量级。

**证据链：**

1. **规范要求**（`rnd/research/coordinator/RULER-SPEC-v1-r4-2026-10-21.md`，sha256 `be8d7554…` 与 `.sha256` 文件核对一致）：
   - l.38：*"Deviation from R-D3, stated [F5]: the v0.4 items were written half by Claude and half by Codex, the same two families that judge and write the references. **Mitigation, not a cure: every ratio is also published as a judge family × writer family table.**"*
   - l.51（§5 报告行清单）：*"by `writer`; **judge family × writer family [F5]**; … time_sensitive excluded vs included [R-D4]; multi-turn alone; over_refusal items (refusal rate)…"*
   - l.201（§6.4）：*"Judge × writer table printed [F5]."*
   - l.25（§0.6）：bar 的定义使用 **"min over the two judge families"** ——即家族配对结构直接进入判定。

2. **实际产出缺失**：
   - `arms/device-final/tools/m1_agg.py`（metrics() 函数）：唯一的分层是 `strata = {lab: … for lab in ("AVAILABLE","PARTIAL","NONE")}`（factavail）；**没有 writer、没有 time_sensitive、没有 type/family/multi-turn/over_refusal 分行**。`DEVICE-FINAL.md` 的 M1 表也只有这一种分层。
   - `grep -rn "writer" arms/device-final/ arms/native-rc/` → 无结果（两份最终报告及全部聚合工具都不含 writer 字段）。
   - TRACES.tsv 列出的 4 个最终 judgment 文件（`judge/dev-s4{2,3}.{opus,codex}.judgments.jsonl`）只存在同家族配对：**不存在任何 Claude 裁判×Codex 参照（或反向）的运行**。
   - `data/facts.json` 中关于 writer 的唯一相关事实 F-1581（"Same-family judge lift (literature): 3.4-8.4 pp"，caveat: "motivates judge × writer family table"）——文献量级被记录，但表格本身从未产出。

3. **量级对照**（复算）：headline G1 0.1645 vs G2 0.2424，差 **+0.078**。XFAM 实验（QUEUE.md l.1238/1248）在 v0.2 上测得 Codex 裁判对**同一批答案、同一批 Claude 参照**整体比 Claude 裁判宽 **+0.10 raw / +0.13 floor-adj**；REFVAR（`ruler/REFVAR-2026-10-21.md` l.46）自己声明 *"the same-family lift … is not measured"*。也就是说：0.078 的家族差完全落在"裁判宽严 +0.10"的已知区间内，**留给"自家族参照抬分"的残差无法从现有数据分离**——而 F-1581 引用的文献 lift（3.4-8.4pp）作用于参照分母时方向恰好是压低 U。
   - 附带的小验证（本地可复现）：对 `bench/v0.4/refs/v2/{claude,codex}.jsonl` 做 checklist 词面覆盖，自家族参照 0.5998 vs 异家族 0.5646（+3.5pp，n=329/329）——文本层面确有轻微自家族覆盖优势，judge 行为层面则完全未测。

4. **计划内本可解决该问题的交叉实验被砍**：CV5-LADDER-CODEX（`ruler/CV5-LADDER-CODEX-criterion.md`，408 calls 的完整 packet 已备好，`ruler/codex-ladder/`）deadline 10-23 18:00；`ruler/codex-ladder/out/` 目录为空（仅 `.keep`），无任何报告文件 —— **未运行**。

**为什么重要：**

headline "0.165/0.242，取低者低于 0.50"的判定结构（§0.6 min-over-families）把两个"裁判×自家参照"的比值当成两个独立证据源；若自家族参照抬分存在（文献 3.4-8.4pp），两个家族的 U 都被系统性压低，min 也不例外。这不影响"远低于 0.50"的方向性结论（差距太大），但影响：(a) 两个家族差 0.078 的解读（现在是"裁判宽严"与"参照偏倚"的未分离混合）；(b) 任何未来复测/他人复现时的可预期偏移；(c) 规范自设的 F5 缓解措施事实上未执行——与 E4（漏打一行分层）同类但范围更大：**§5 列出的整组报告行（judge×writer、time-sensitive、type、multi-turn、over_refusal）在最终 DEVICE-FINAL 中全部缺失**。其中 time-sensitive 占 headline 权重 25.2%（18/71 条，16.5/65.5，复算见下），对离线 app 是结构性不利项，规范明说"reported, not corrected"（l.88）——但最终报告没有 report。

**建议的修复/验证（可复现步骤）：**

1. 复算权重占比：`python3 -c "import json;qs=[json.loads(l) for l in open('boar-bench/bench/v0.4/items.jsonl')];h=[q for q in qs if not q['tags']['anchor_vexact']];print(sum(0.5 if q['tags']['health_safety'] else 1 for q in h if q['tags']['time_sensitive']),sum(0.5 if q['tags']['health_safety'] else 1 for q in h))"` → `16.5 65.5`。
2. 补 judge×writer 表（需要 judge 原始行，HF 数据集恢复可达后）：对 `judge/dev-s4{2,3}.{opus,codex}.judgments.jsonl` 按 `queryId` 前缀（cw-/cx-）分组重算 U_total（权重同 m1_agg），4 格：{opus,codex}×{cw,cx}。若 G1 在 cx 条目上的 U 系统性高于 G2 在 cx 条目上的 U（同条目跨家族对比），即可把裁判宽严与参照家族效应分离。
3. 低成本直接实验：用既有 160-call 上限的一小部分跑 2×2 交叉（Opus×Codex refs、Codex×Claude refs，各抽 20-30 条 headline），直接测参照家族配对效应；这是规范 F5 本来的意图。

---

## 发现 3（次要，方法学）：QFORM 的"gold 保留 ⇒ 1.7B 达到参照水平（U 0.88）"这一被用于战略转向的数字，其参照答案与评分细则都派生自同一条 pack-fact gold，且裁判为被记录在案会压缩/误判分数的 Haiku 5.5-low——该 0.88 高估了"保留即达标"，而记录随后用它论证"packs are the strategy"的方向性决策。

**证据链：**

1. `arms/qform/QFORM-RESULTS.md` Secondary 段 Setup：*"Judge: judge.mjs judge.v1, Haiku 5.5-low, N 4 shards, both orders, **reference = item `answer`, rubric = item `rubric`**（judge/bench/qform192）"*。条目集是"48 pack facts × 4 种问法"，gold/rubric/answer 均由协调 agent 从同一 pack 事实写就——参照与被评答案共享同一事实来源，pairwise 比较天然向"覆盖该事实的答案"倾斜。
2. Haiku 作为该比较裁判的已知缺陷（`ruler/HAIKU-CAL-2026-10-21.md` TL;DR）：*"Haiku is a good CE detector and a poor 'correct' detector… kappa on 'correct >= 4' 0.49… **warned answers are where Haiku fails most: kappa 0.22**"*；以及 CV5-LADDER-OPUS（`ruler/CV5-LADDER-OPUS-2026-10-21.md`）：Haiku 对 1.7B/1.5B 档的压缩（*"The Haiku L1 judge compressed the 1.7B's lead over the floor"*，1.7B−1.5B 在 Haiku 下 +0.038、Opus 下 +0.090）。
3. 该数字直接驱动方向决策：QUEUE.md l.1594（10-21 16:05）：*"r4to: stop waiting for the model to answer by itself; win the post author-style questions by building packs that support them; the research to present is **with the reference the model answers well, without it badly (QFORM 0.88 vs 0.19, Haiku, 190 pack answers)**"*，并开启 coverage lane。
4. 缓和因素（诚实披露）：0.88 vs 0.19 的**方向**与 SEFIX 2×2 的交付格子（发现 1）一致，故"交付重要"不被此缺陷推翻；受影响的是"保留后即达参照水平（≈0.88/0.9）"这一**高度**表述，及据此对 pack 覆盖率的优先级排序。

**为什么重要：**

这是整个项目最大的一次战略转向（从模型侧转向 pack/交付侧）所引用的核心量化依据；若 0.88 因参照同源 + Haiku 宽严而偏高 0.1-0.15，则"把每条事实装进包里就接近解决"的预期被高估，可能导致后期对 pack 覆盖的投入-回报预期失真（后期 NIGHT A kit 实验"adding qa-ethereum/nist-crypto adds 0 v0.1 facts"已显现覆盖边际收益递减）。

**建议的修复/验证（可复现步骤）：**

1. 核对来源：读 `arms/qform/QFORM-RESULTS.md` Secondary 段与 `arms/qform/gen/`（gold/rubric 生成物在 HF traces）确认 reference 与 rubric 同源。
2. 判决实验：对 QFORM 的 43 条"gold 保留"条目用 **Opus**（judge.v1）重判、并改用**独立参照**（例如按 v0.4 流程由 Claude/Codex 联网另写参照），报告 U 的变化；约 43×2 orders×2 参照 ≈ 172-344 calls，在项目原有单实验预算内。

---

## 已核实为正确/无新发现的部分（负面结果，供评审参考）

- **U-total 复算**：`m1.json` 内部自洽——`sum(w·u)=U×sum(w·r)`（44.0=0.672×65.5 侧验证了 refusal 的 z 置零路径：5 条 z 条目对 `abs_app` 贡献 1.67、对 u 贡献 0，隐含 refusal 固定文本被裁判打 ~1.35/5 的低分，合理）；bootstrap（B 2000, seed 11, mulberry32）与 boar-bench `scorers/stats.py` 逐位一致。
- **种子方差**：s42/s43 的 refusal 条目（各 5 条）与覆盖率（各 68/75）完全相同，temperature 0.7 下两次生成高度收敛；pooled CI（[0.127,0.205]）与单种子 CI 几乎等宽，未见因"两种子"而人为收窄。
- **冻结完整性**：`tools/verify-criteria.py` → ok 333 / bad 0（4 个例外均在 `docs/PIN-EXCEPTIONS.md` 声明）；`boar-bench tools/verify-pins.py` → pins ok 153 / bad 0；RULER-SPEC r4 的 sha256 与 `.sha256` 文件一致。两个仓库的全部赛后 commit 均为文档/修复性质，无改动已冻结题目/参照的痕迹（v0.4 `items.jsonl` 自冻结以来字节未变，SHA256SUMS 通过）。
- **judge 盲化**：`blind()` 对双方对称地剥除 Sources 块、URL、`[1]` 引用号、markdown 强调——LabRat 的 `[1]` 引用与参照的 URL 引用被同等抹除，无单侧泄漏（node 实测验证）。
- **E1-E8 及 5 个 labrat 修复 commit 覆盖的问题**（eval runner pack id、health gate、score.py bar、m1.json CR/CE、current-city 行、location stratum 等）不再重复。
