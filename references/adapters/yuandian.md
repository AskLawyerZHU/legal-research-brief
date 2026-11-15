# 适配器 · 元典（华宇元典）

本机 connector 名称：`yuandian-mcp`（**不是** `yuandian-law`，旧文档里的 server 名已过时）。
数据范围：中国大陆法律法规 + 裁判文书 + 权威案例。

> 本文件是本技能唯一的「工具名发生地」。SKILL.md 里只写抽象动作，翻译成具体工具名只在这里做。元典改版只改这一个文件。

## 一、连通性探测（开工必做，不得跳过）

调一次最轻的检索，确认**实际返回数据**，而不是看配置说已连接：

```
search_articles(keyword="民法典", top_k=1)
```

- 返回条目 → 能力卡写「元典：已验证连通」
- 报错／空白／超时 → 写「元典：不可用」，转 `no-db.md`，**不要用记忆补条文**

## 二、抽象动作 → 工具名

| 抽象动作 | 元典工具名 | 必填 | 常用可选参数（现用名） |
|---|---|---|---|
| `search_articles` | `yuandian_search_legal_articles` | `keyword` | `regulation_name`、`validity_status`、`primary_effect_level`、`region`、`top_k` |
| `search_regulations` | `yuandian_search_laws_and_regulations` | 无（全可选） | `keyword`、`regulation_name`、`validity_status`、`primary_effect_level`、`region`、`top_k` |
| `get_regulation_detail` | `yuandian_get_law_or_regulation_detail` | `regulation_id` 或 `regulation_name` | `reference_date`（按指定日期取有效版本） |
| `get_article_detail` | `yuandian_get_legal_article_detail` | 见工具 schema | 传 `article_id`，或以法规名＋条款号定位 |
| `search_cases` | `yuandian_search_judicial_cases` | 至少 1 个过滤条件 | `keywords`、`cause_of_action`、`court_name`、`province`、`decision_date_start/end`、`cited_articles`、`case_number`、`document_type`、`top_k` |
| `search_authoritative_cases` | `yuandian_search_authoritative_cases` | 至少 1 个过滤条件 | `keywords`、`authoritative_case_sources`、`cause_of_action`、`court_name`、`province`、`decision_date_from/to`、`top_k` |
| `semantic_search_articles` | `yuandian_semantic_search_legal_articles` | `query` | `limit`、`rewrite_enabled`、`law_article_filter{validity_status, regions, primary_effect_level, effective_date_from/to}` |
| `semantic_search_cases` | `yuandian_semantic_search_cases` | `query` | `limit`、`rewrite_enabled`、`case_document_filter{court_level, court_names, cause_of_action, trial_procedure, decision_date_start/end, province, city, authoritative_only, authoritative_case_sources}` |
| `get_case_detail` | `yuandian_get_case_detail` | `case_id` 或 `case_number` | `case_kind`：`ptal`（普通）／`qwal`（权威） |
| `make_share_link` | 无独立工具：调 `get_case_detail` 或 `get_law_or_regulation_detail`，读返回的 `url` + `share_access` + `share_expires_at` | — | 本机实测**只返回 `login_required`**，见第五节 |

### 返回值字段（实测，与请求参数不同名）

请求用语义化参数名，**返回的字段不少是拼音缩写**，取错就取不到值：

| 返回字段 | 含义 | 备注 |
|---|---|---|
| `article_id` / `regulation_id` | 法条 ID / 法规 ID | 传给 detail 类工具用 |
| `validity_status` | 现行有效／失效／已被修改 | 与请求参数同名 |
| `primary_effect_level` | 法律／司法解释／行政法规… | 与请求参数同名 |
| `effective_date` / `publication_date` | 施行日／发布日 | 引用时都要写 |
| `issuing_authority` | 发布机关 | 引用时带上 |
| `cj` | 法院层级（基层／中级／高级／最高） | **不是** `court_level` |
| `spcx` | 审判程序（一审／二审／再审…） | **不是** `trial_procedure` |
| `xzqh_c` / `xzqh_p` | 地级市／省 | **不是** `city` / `province` |
| `case_kind` | 普通案例／权威案例 | 转 `case_kind=ptal/qwal` 查详情 |
| `_score`（法条）／`score`（案例） | 相关度分值 | **相对值，不是百分比**，不可写进报告 |

detail 接口（`get_case_detail`）独有、**直接可用**的字段——省掉人工摘要：

| 返回字段 | 含量 | 用法 |
|---|---|---|
| `focus` | 已提炼的争议焦点列表（换行分隔） | 直接做争点分析的起点，**仍需律师复判是否属于本案争点** |
| `applied_laws` | 判决援引的全部法条数组 | 直接做「法条索引」节的底稿，再逐条查现行有效性 |
| `judgment_result` | 判决主文 | 引用判项原文，不要转述数字 |
| `case_basic_facts` / `established_facts` | 诉辩要旨／法院认定事实 | 事实比对，区分「当事人主张」与「法院认定」 |
| `court_reasoning` | 本院认为部分全文 | 类案要旨必须从这里摘，不能从 `content` 摘要摘 |
| `related_case_numbers` | 关联案号数组（常含二审案号） | 追踪后续程序；有值时报告须注明「本案存在关联程序」 |
| `parties` / `judge` / `lawyer` | 当事人／审判人员／代理律师 | `parties` 多为化名，**不可当作真实姓名引用** |

请求侧写法（如 `case_document_filter.court_level`）与返回侧字段名不一致，这是最常见的取错点。

## 三、参数默认值（三个坑）

1. **不传 `validity_status` 时默认只返回「现行有效」**。要查失效条文（比如做沿革比较），必须显式传 `失效`／`已被修改`／`部分失效`。
2. **`top_k` 默认 10，最大 50**。要穷举必须分批换关键词，且如实说明未穷举。
3. **案例分页最多翻阅前 500 条**，超出静默截断。命中数 > 500 时先收窄条件，不要假装翻完了。

## 四、`cited_articles` 反查（最有用的一招）

用于回答「这条法条在类案里到底怎么被用」。格式要求严格：**「中华人民共和国＋法规全名＋中文数字条款号」**，一条一个元素。

```
cited_articles = ["中华人民共和国民法典第五百零九条"]
article_match_mode = "and"   # 默认 and
```

这是把「法条」和「案例」两条轨连起来的接口：先 `search_articles` 定条号，再用 `cited_articles` 反查适用该条的判决，比纯关键词准得多。

## 五、回链规则（双层引用）

**检索结果里的 `url` 需要登录，且不带分享凭证——不能直接贴给别人。**

要转交，调 detail 接口，看返回的两个字段：

| `share_access` | 含义 | 怎么用 |
|---|---|---|
| `login_free` | 免登录分享链接 | 给可点链接，**必须同时写明**「有效期至 `share_expires_at`」 |
| `login_required` | 仍需登录 | 不给链接，只给永久标识，让对方自行在库内检索 |

### 实测结论（2026-11-14，必须照此降级）

本机当前账号层级，**普通案例（`ptal`）与权威案例（`qwal`）两条路径实测均返回
`share_access: "login_required"`、`share_expires_at: null`**。

所以：

1. **不要承诺免登录链接。** 除非某次真实返回 `login_free`，否则一律按「需登录」处理。
2. **双层引用的重心必须放在永久标识层。** 现在能交付给客户／非同行的，只有文字标识＋原文摘录，不是链接。
3. **要转交全文时，靠摘录不靠链接。** 把 `court_reasoning`、`judgment_result` 原文摘进报告，注明出处标识；不要写「详见链接」。
4. **如果哪天返回了 `login_free`**，那是临时凭证：必须写到期日，且**不得缓存复用**——下次检索重新取。

引用一律写成两层（实测下的标准写法）：

```
《民法典》第五百零九条（现行有效｜法律｜全国人大常委会｜2020-05-28 通过，2021-01-01 施行）
来源：元典法规库｜检索日期 2026-11-14
```

案例写：

```
（2025）京0114民初27373号｜北京市昌平区人民法院｜2026-03-25｜一审
来源：元典案例库｜检索日期 2026-11-14
可点：{url}（需登录，不建议转交；全文摘录见下）
```

**永久标识层绝不能省**——链接一定会过期、也大概率需要登录，文字标识不会。

不要自行拼接 `shareCode`，也不要缓存别人的免登录链接当长期引用。

## 六、旧名对照（给沿用旧写法的技能做转换）

本机若遇到下列旧名，说明是过时写法，按右列替换：

| 旧工具名 | 现用工具名 |
|---|---|
| `yuandian_rh_ft_search` | `yuandian_search_legal_articles` |
| `yuandian_rh_fg_search` | `yuandian_search_laws_and_regulations` |
| `yuandian_rh_fg_detail` | `yuandian_get_law_or_regulation_detail` |
| `yuandian_rh_ptal_search` | `yuandian_search_judicial_cases` |
| `yuandian_rh_qwal_search` | `yuandian_search_authoritative_cases` |
| `yuandian_rh_case_details` | `yuandian_get_case_detail` |
| `yuandian_law_vector_search` | `yuandian_semantic_search_legal_articles` |
| `yuandian_case_vector_search` | `yuandian_semantic_search_cases` |

参数旧名同样失效，常见对照：`fgmc→regulation_name`、`sxx→validity_status`、`xljb_1→primary_effect_level`、`dy→region`、`qw→keywords`、`ah→case_number`、`ay→cause_of_action`、`jbdw→court_name`、`wszl→document_type`、`ajlb→case_category`、`ja_start/ja_end→decision_date_start/end`、`refer_date→reference_date`、`return_num→limit`、`fayuan→court_names`、`cj→court_level`、`type→case_kind`、`source→authoritative_case_sources`、`dianxing→authoritative_only`、`wenshu_filter→case_document_filter`、`fatiao_filter→law_article_filter`。

> 服务端对部分旧参数名仍可能兼容，但**旧工具名一定找不到**。判断能不能跑，看工具名。

## 七、检索策略建议

- 已知法规名＋条款号 → 直接 `get_article_detail`，不要先做大范围检索。
- 只有问题、没有术语 → 先 `semantic_search_articles` 拿候选条文，再用条文原词做 `search_articles` 精确召回。
- 类案两库必查：`search_authoritative_cases`（定调）＋ `search_judicial_cases`（本地实践）。只查前者会脱离本地裁判习惯，只查后者会漏指导性案例。
- 类案可按 `court_level`（基层／中级／高级／最高）、`trial_procedure`（一审／二审／再审审查／再审）分层取，结论里必须标出层级。

### 权威库对口语词几乎零容忍（2026-11-15 实测）

同一主题，两组关键词结果差到「0 条 vs 11 条」：

| 检索式 | 库 | 返回 |
|---|---|---|
| `规培 违约金` | 权威库 | **0 条** |
| `服务期 违约金 事业单位` | 权威库 | **11 条**（南京、重庆、安庆、云南、江门、江西、北京、新疆、福建） |

**原因**：权威案例（典型案例／参考案例）的标题与要旨使用**规范术语**，「规培」这类行业口语只出现在基本案情里，不进标题与裁判要点。

**规则**：

1. 权威库返回 0 条时，**先换规范术语，再加检索次数**——常见映射：规培→服务期／专业技术培训；医生辞职→辞职纠纷／人事争议；主播跳槽→竞业限制；欠钱不还→民间借贷。
2. 普通库口语词容错高（同一主题命中 1,623 条），所以**两库要用不同关键词**：普通库可以用当事人说法原词，权威库必须换成法言法语。
3. 权威库 0 条不能写「无此类案例」，只能写「以该关键词未检索到」，并列出改用过的词。
