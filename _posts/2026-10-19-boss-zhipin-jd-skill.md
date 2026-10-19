---
title: 写一个通用的 Agent Skill，让 AI 在 BOSS直聘上查 JD
description: 把 BOSS直聘的检索规则写成一份通用的 SKILL.md，Claude Code、Codex 等支持 Skill 的 Agent 都能加载：借浏览器工具在已登录的 Chrome 里翻列表、读详情、还原字体混淆的薪资，把 JD 存成 Markdown。附完整 SKILL.md 和踩过的坑。
date: 2026-10-19 19:30:00 +0800
categories: [AI, Agent]
tags: [Skill, Agent Skills, SKILL.md, Claude Code, Codex, BOSS直聘, 浏览器自动化]
---
想知道一类岗位都在要什么技能，最直接的办法是去 BOSS直聘把 JD 翻一遍。这活我交给了 Agent，让它翻完再整理成 Markdown。跑通以后，我把整套检索规则写成了一个 Skill，叫 `boss-jd-search`。现在说一句「去 BOSS 上搜上海的数据分析岗位，20K 以上」，Agent 就会在我已登录的 Chrome 里翻列表、逐条点开详情、还原被字体混淆的薪资，最后写出一个 `jobs/<日期>-<关键词>.md`。关键词、城市、薪资每次现给，Skill 不绑定任何职位方向，搜后端、产品还是设计都一样。

这个 Skill 不绑定哪家 Agent，Claude Code、Codex 都能加载。我自己是用 Claude Code 加 Claude in Chrome 跑通的，文中跟这套工具有关的限制都会单独标出来。

这套做法有三条底线：不写爬虫，不调接口，不碰「立即沟通」。Agent 只是替我在浏览器里点和读，节奏放慢，遇到验证码就停下来叫我。

## 准备工作

1. **一个支持 Skill 的 Agent。** `SKILL.md` 就是一个带 `name`、`description` front matter 的 Markdown 文件，格式是公开的 Agent Skills 标准，各家按目录找它：Claude Code 读 `~/.claude/skills/`，Codex 读 `~/.agents/skills/`。我把 Skill 实际放在 `~/.agents/skills/boss-jd-search/`，再软链一份到 `~/.claude/skills/`，一份文件两边用，改一处就行。
2. **一个能操作你那个 Chrome 的浏览器工具。** 它要能打开网页、在页面里执行 JS、读页面文本，而且操作的必须是你已经登录 BOSS 的那个浏览器。新起一个干净的浏览器实例是没有登录态的。我用的是 Claude in Chrome 扩展，靠它的 `navigate`、`javascript_tool`、`get_page_text` 干活。用别的 Agent，就接一个能连上正在运行的 Chrome 的 MCP，比如 Chrome DevTools MCP。这个组合我没实际跑过，下面那些限制要自己重新测。
3. **在 Chrome 里手动登录 BOSS直聘。** 没登录时列表页不显示薪资。

## 页面上踩过的坑

下面这些是 2026 年 10 月实测的，BOSS 改版后可能失效。列表页 URL、城市码和全部选择器都写在后面的 SKILL.md 里，这里只讲费了点工夫的几处。

**薪资是混淆字体。** `.job-salary` 的 textContent 里，数字 0–9 被换成了私有区字符 U+E031 到 U+E03A，页面靠 kanzhun 字体把它们画成数字。直接读出来是一串方块，按编码减 `0xe031` 就能还原：

```js
const decode = s => s.replace(/[\ue031-\ue03a]/g, c => c.charCodeAt(0) - 0xe031);
```

**筛选不能全信页面。** URL 里的 `salary=406` 是 20-50K 档，筛得很松，下限要自己再过一遍。本地结果不够时，BOSS 还会往列表里塞别的城市的岗位，写文件前要按 `.company-location` 再筛一次城市。

**数据怎么拿出来，这个坑最深，而且是 Claude in Chrome 特有的。** `javascript_tool` 每次调用最多跑 45 秒，返回内容超过一千五百字左右就被截断，带上职位描述，两三条 JD 就超了。我试过从页面直接往本机 localhost 发数据，被权限拦了，这条路走不通。最后用了个笨办法：

1. 每次执行 JS 只处理几张卡片，读到的数据追加进 `sessionStorage`，返回值只报个数。
2. 全部采完后，把 `sessionStorage` 里的 JSON 写进页面上的一个 `<pre>`。
3. 用 `get_page_text` 读页面文本，一次能读三万字左右，分批导出就行。

导出那一步大概是这样，下面是示意代码，没有原样跑过：

```js
const pre = document.createElement('pre');
pre.textContent = sessionStorage.getItem('boss_jobs');
document.body.replaceChildren(pre); // 页面会被清空，读完刷新再继续
```

换了浏览器工具，单次执行时长和返回长度的上限都不一样。要是你的工具能一次返回大段 JSON，`<pre>` 这一步可以省掉。

## 完整的 SKILL.md

`description` 决定 Agent 什么时候想起用这个 Skill，触发词要写全，中英文都放上。正文写给 Agent 看，只写它要照着做的事。为了换 Agent 不用改，正文里不写死工具名，只写「执行 JS」「读页面文本」这类动作，某个工具专有的限制单独注明。

````markdown
---
name: boss-jd-search
description: 在 BOSS直聘（zhipin.com）检索岗位 JD，通过浏览器工具使用用户已登录的 Chrome 浏览，把岗位信息整理成 jobs/ 下的 Markdown。用户提到 BOSS直聘、zhipin、查岗位、搜 JD、找工作岗位、岗位要求时使用。
---

# BOSS直聘 JD 检索

## 开始前

- 向用户确认：检索关键词、城市、薪资下限。这些只用于本次检索，不写进任何文件。
- 需要一个能操作用户已登录 Chrome 的浏览器工具（能打开网页、在页面执行 JS、读页面文本）。没有就停下，告诉用户需要接入。不要新起一个没有登录态的浏览器。
- 用浏览器工具打开新标签页。BOSS直聘 需要用户已经登录；没登录（列表不显示薪资）就停下，请用户登录。

## 硬规则

- 只用浏览器读页面。不写爬虫脚本，不调接口批量抓取。
- 只读不写：不点「立即沟通」、不发消息、不投简历、不改在线简历。用户明确要求时，每次操作前再确认一遍。
- 翻页、打开详情放慢节奏。出现验证码或登录失效，立刻停下，请用户手动处理。
- 不记录招聘人姓名。

## 页面细节（2026-10 实测，页面改版后需重新确认）

- 列表页：`https://www.zhipin.com/web/geek/jobs?query=<关键词>&city=<城市码>&salary=406`。城市码：上海 101020100，杭州 101210100，南京 101190100，苏州 101190400；其他城市在页面上选好后读 URL 的 `city` 参数。`salary=406` 是 20-50K 档，薪资下限要自己再过滤。
- 薪资用 kanzhun 字体混淆：`.job-salary` 的 textContent 里，U+E031–U+E03A 依次对应数字 0–9，按编码减 `0xe031` 还原。
- 卡片 `.job-card-wrap`；职位名和详情链接 `.job-name`；公司 `.boss-name`；地点 `.company-location`；经验和学历 `.tag-list li`。
- 点击 `.job-name` 后右侧 `.job-detail-box` 加载详情：技能标签 `.job-label-list`，职位描述 `.desc`。
- 列表滚到底每次多加载 15 条。
- 本地结果不够时会混入外地岗位，按 `.company-location` 的城市再筛一次。

## 采集与导出

- 浏览器工具单次执行 JS 的时长和返回长度通常有上限。每次只处理几张卡片，卡片之间等两三秒，结果追加到 `sessionStorage`，返回值只报条数。
- 全部采完后，把 `sessionStorage` 里的 JSON 写进页面上的 `<pre>`，再用读页面文本的工具分批读取。工具能直接返回完整 JSON 时可以跳过这一步。
- Claude in Chrome 实测：`javascript_tool` 每次最长 45 秒，返回超过约 1500 字会被截断；`get_page_text` 一次约 3 万字；从页面往本机 localhost 发数据会被权限拦截，不要尝试。

## 写文件

输出到 `jobs/<YYYY-MM-DD>-<关键词>.md`。每个岗位记：

- 职位名、公司、城市/区域、薪资
- 经验与学历要求、公司规模与融资阶段
- 岗位职责
- 技能要求，必须项和加分项分开写
- 福利标签、岗位链接

要求：

- 技能要求和薪资照抄原文，不自行概括。
- 按公司 + 职位名 + 链接去重。
- 文件开头写明检索条件、城市筛选和去重后的条数。
````

## 几条规则为什么写死

- **只读。** 「立即沟通」按钮就在详情旁边，点一下就是以我的名义给 HR 打了招呼。这种事我不想交给 Agent 判断，所以写成硬规则，要做也得我每次点头。
- **照抄原文。** 「熟悉 SQL，能用 Python 做数据清洗」这种原话，正是改简历时要对齐的关键词。缩成「有数据处理能力」，这份文件就没用了。
- **检索条件不落盘。** 城市和薪资期望属于个人信息，我宁可每次多回答一遍，也不让它写进文件。

## 跑起来是什么样

在工作区里打开 Agent，直接说：

```text
去 BOSS 上搜一下杭州的产品经理岗位，薪资下限 30K
```

Agent 会先认出这是 `boss-jd-search` 的活，读 Skill，然后：

1. 确认关键词、城市、薪资下限，没说全的先问我。
2. 新开标签页打开列表页，滚动加载，把卡片读进 `sessionStorage`。
3. 逐条点开详情，补上技能标签和职位描述。
4. 把数据导出来，按薪资下限和城市过滤、去重。
5. 写出 `jobs/2026-10-19-产品经理.md` 这样的文件。

单条岗位的格式是这样：

```markdown
### 1. <职位名> · <公司> · <薪资>

- 地点：<城市/区域>｜经验：<经验>｜学历：<学历>
- 技能标签：<.job-label-list 原文，没有就写（无）>
- 链接：https://www.zhipin.com/job_detail/<编号>.html
- 职位描述：<.desc 原文>
```

分析是另一步。JD 落成文件以后，我再让 Agent 读 `jobs/` 下的文件，统计技能出现频次，跟自己的技能清单对一遍，看缺什么。这一步跟浏览器无关，所以没放进这个 Skill。

## 让 Agent 自己装上

把下面这句复制给你的 Agent，它会自己下载并装好这个 Skill：

```text
请读取 https://raw.githubusercontent.com/redwolf2019/redwolf2019.github.io/main/_posts/2026-10-19-boss-zhipin-jd-skill.md ，把「完整的 SKILL.md」一节里 markdown 代码块的内容原样保存为你的 Skill 目录下的 boss-jd-search/SKILL.md（Claude Code 是 ~/.claude/skills/，Codex 是 ~/.agents/skills/，其他 Agent 用你自己的 Skill 目录），保存后告诉我文件路径，再检查你有没有能操作我已登录 Chrome 的浏览器工具。
```

## 推荐用法：让 Agent 每天查一次新岗位

Skill 装好以后，可以让 Agent 隔一段时间自动跑一遍，比如每天一次，只记新出现的岗位。别设得太勤，BOSS 有反爬，一天一次足够了。

在 Claude Code 里用 `/loop`：

```text
/loop 1d 用 boss-jd-search 搜杭州的产品经理岗位，薪资下限 30K，只记 jobs/ 里以前没出现过的新岗位
```

间隔支持 `m`、`h`、`d` 几种单位。`/loop` 只在当前会话里有效，Claude Code 关掉就停了；循环任务创建 7 天后自动过期，想继续就重新建一个。

其他 Agent 的做法不一样。不管用哪个，有一条前提不变：任务得跑在你自己的电脑上，Chrome 开着并保持登录。跑在云端的定时任务碰不到你本机的浏览器，这个场景用不了。

| Agent | 定时方式 | 能不能用在这里 |
|---|---|---|
| Claude Code | 会话里用 `/loop 1d <提示词>` | 能。会话内有效，7 天过期 |
| Codex | 桌面 app 里的定时任务（Scheduled tasks），可选每天、每周或自定义；Codex CLI 没有这个管理界面 | 能。任务要选本地项目目录，电脑开着、app 运行着；定时任务可以调用 Skill |
| Cursor | Automations，预设间隔或 cron 表达式 | 不合适。Automations 跑的是云端 agent，碰不到本机已登录的 Chrome |
| Gemini CLI 等没有内置定时的 | 用系统的 cron、launchd 或 Windows 任务计划程序，定时调用非交互模式，比如 `gemini -p "<提示词>"`；`claude -p`、`codex exec` 同理 | 可以试。非交互模式下浏览器工具也要配好，这条路我没实际跑过 |

## 相关引用

- [Equipping agents for the real world with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)：Anthropic 讲 Skill 原理的工程博客。核心是渐进式披露：Agent 平时只把每个 Skill 的 `name` 和 `description` 放进上下文，用得上时才读 `SKILL.md` 正文，正文里引用的其他文件再按需读。
- [Agent Skills 规范](https://agentskills.io/specification)：Skill 的开放标准，规定了 `SKILL.md` 的 front matter 字段和目录结构。
- [Claude Code 的 Skill 文档](https://code.claude.com/docs/en/skills)：Skill 放在哪、怎么触发，以及 Claude Code 在标准之外加的功能。
- [Codex 的 Skill 文档](https://developers.openai.com/codex/skills)：Codex 扫描 Skill 的目录，包括用户级的 `~/.agents/skills/` 和仓库里的 `.agents/skills/`。
- [Claude Code 的定时任务文档](https://code.claude.com/docs/en/scheduled-tasks)：`/loop` 的写法、间隔单位和 7 天过期规则。
- [Codex 定时任务文档](https://developers.openai.com/codex/app/automations)：在桌面 app 里建定时任务，本地任务需要电脑开着、app 运行。
- [Cursor Automations 文档](https://cursor.com/docs/cloud-agent/automations)：定时触发的写法，以及它跑在云端 agent 上这一点。
- [anthropics/skills](https://github.com/anthropics/skills)：Anthropic 官方的 Skill 示例仓库，写自己的 Skill 前可以照着看。
