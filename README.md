# 法律检索简报（legal-research-brief）

把「查法条、找类案」变成一份**每条都能追溯出处**的检索简报，需要时再转成可直接发出的报告网页。

方法论固定，数据源可换，缺数据时明确说缺——不猜、不编、不假装穷举。

## 它和常见的"AI 法律检索"差别在哪

| | 常见做法 | 本技能 |
|---|---|---|
| 引用 | 只给结论和"相关规定" | 每条带条款号 / 案号 / 法院 / 裁判日期 / 检索日期 |
| 链接 | 给个链接就算 | 永久标识为主；链接只在真能免登录时给，需登录就明说、改给原文摘录 |
| 检索不到 | 凭记忆补一条像样的 | 停下来标 `[待检索]`，给检索式，不写条文 |
| 类案 | 只找支持己方的 | 对立案例必查，分歧并列不合并 |
| 可视化 | 配图或截图，看着热闹 | 关系图／结果分布图，自包含 SVG，**图上保留待核验区**，不比文字更敢说 |
| 给非同行 | 直接给结论 | 只给条文与清单，默认弹"请律师复核" |

## 两种运行模式

- `practitioner`：律师 / 法务 / 法学生。输出完整简报，含争点分析与倾向。
- `general`：当事人 / 业务方。只输出条文原文、事实清单、检索路径，**不做争点判断**，交付前必弹提示。

模式按对象判断，判不准会问一句。

## 适配哪些环境

| 环境 | 状态 | 走哪个适配器 |
|---|---|---|
| 元典（yuandian-mcp） | 已验证 | `references/adapters/yuandian.md` |
| 北大法宝 | 需自行配置 | `references/adapters/pkulaw.md` |
| 没有任何数据库 | 支持降级 | `references/adapters/no-db.md` |

不会锁定供应商：只要环境里有一个「输入检索词、返回真实结果」的工具，改适配器即可，`SKILL.md` 不用动。

## 安装

三种装法，任选其一。

**A. 从 GitHub 克隆**（便于后续 `git pull` 更新）

```bash
git clone git@github.com:AskLawyerZHU/legal-research-brief.git \
  ~/.workbuddy/skills/legal-research-brief
```

SSH 不通时改走镜像下载 zip：

```bash
curl -L -o /tmp/lrb.zip "https://ghfast.top/https://github.com/AskLawyerZHU/legal-research-brief/archive/refs/tags/v1.2.0.zip"
unzip -q /tmp/lrb.zip -d /tmp/lrb && mv /tmp/lrb/legal-research-brief-1.2.0 ~/.workbuddy/skills/legal-research-brief
```

`ghfast.top` 不通就换 `gh-proxy.com`，两者读同一上游。

**B. SkillHub**：搜「法律检索简报」一键安装。

**C. 手动拷贝**：把本目录放到下列位置之一，然后配置数据源。

1. 用户级（所有项目可用）：`~/.workbuddy/skills/legal-research-brief/`
2. 项目级（团队共享）：`<项目>/.workbuddy/skills/legal-research-brief/`
3. 配置数据源：元典走连接器；法宝在 `~/.workbuddy/mcp.json` 的 `mcpServers` 里加一项（Token 只放本机，不要写进技能）。
4. 跑一次 `references/self-test.md` 的「环境」五项。

**不做第 3 步也能用**——会进入降级模式，只输出检索式和待查清单。这是设计好的行为，不是故障。

## 目录

```
SKILL.md                        方法主体（只写抽象动作，不含具体工具名）
references/
  workflow.md                   争点拆解、关键词设计、类案三档、交叉复核
  example-brief.md              完整范例：一份写完的简报长什么样（写第一稿前必看）
  source-tagging.md             五种来源标签、双层引用、时效与分歧标注
  report-format.md              简报六节结构、网页填写要点、对外版改动
  responsibility.md             两种模式、脱敏、免责、硬边界
  self-test.md                  环境 / 结构 / 降级 / 复核 / 打包五项自检
  visualization.md              何时该出图、五种图型、布局硬约束、图里绝不能做的事
  wechat-sources.md             法院发布与公众号文章：源可信度分级、三种检索通道、可选增强安装
  profile.example.json          本机配置模板（复制为 profile.json 后生效）
  adapters/
    yuandian.md                 元典：现用工具名、参数、回链规则、旧名对照
    pkulaw.md                   法宝：配置位置与能力对应
    no-db.md                    无库降级：合法输出与措辞纪律
assets/
  report-template.html          报告网页模板（内联样式，可离线打开、可打印）
  relation-map-template.html    法律关系图模板（自包含 SVG，打开即见、可打印）
  chart-templates.html          另四种图型模板：请求权基础图 / 结果分布图 / 时间轴 / 争点攻防图
```

## 公众号 / 法院发布这条轨怎么用

第四轨检索法院官网与公众号文章，主路径是**把检索范围限定在 `mp.weixin.qq.com`**，零配置、无需登录。

若想要「一次拿到带公众号名和精确发布日期的清单」，可以额外装一个小工具。**默认不装，会先问你**；不同意或装不上，就走域名限定检索，功能不受影响。

本技能包**不内置任何第三方脚本**，安装方式见 `references/wechat-sources.md` 第三节。

## 三条硬规则

1. 无来源不入正文（缺永久标识只能进"待核验附录"）。
2. 无数据不生成条文（库不可达就停止，不用记忆补）。
3. 不承诺穷举（库有返回上限，超出时提示收窄）。

## 责任

- 输出是检索底稿，不是法律意见。
- 对非法律专业用户，不给胜诉概率、策略建议或结论性判断。
- 引用前请以官方发布文本为准；链接会过期、也可能需要登录，永久标识不会。

## 版本

见 `CHANGELOG.md`。技能版本与法律数据截止日是两件事，后者在每次检索声明里单独注明。
