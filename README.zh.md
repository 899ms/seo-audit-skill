# seo-audit-skill

[English](README.md) · **中文**

可复用的单页面 SEO 审计 Agent Skill。给一个 URL，输出结构化 HTML 审计报告，包含可执行的修复建议。

基于 **Script + LLM 双层架构**：Python 脚本处理确定性检查（HTTP 状态码、XML 解析、字符串匹配），LLM 处理语义判断（关键词意图、内容质量、页面类型推断）。支持 Claude Code、Cursor 及任何兼容 SKILL.md 的 Agent 运行时。

## 最佳实践

1. 运行 `npx skills add JeffLi1993/seo-audit-skill`，然后发起审计，例如：`audit this page: https://example.com`。根据生成的报告（`reports/<hostname>-audit.html`），自行过一遍：哪些问题与你的业务目标相关、哪些可以忽略。
2. 把报告（或报告中的关键段落）交给 Cursor 或 Claude Code，让 AI 根据报告一项一项协助修复即可。

---

## 报告产出

Basic 审计生成独立 HTML 报告，保存至 `reports/<hostname>-audit.html`。
Full 审计保存至 `reports/<hostname>-full-audit.html`。

| 报告章节 | 内容 |
|---|---|
| **Audit Summary** | 一句话总结 + 关键问题 / 警告 / 通过项一览 |
| **Site Checks** | Sitemap URL Inventory · 可抓取性 · URL 规范化 · i18n · Schema · E-E-A-T |
| **Page Speed** | Full only — Lighthouse 分数、实验室指标、最终 URL、截图可用性 |
| **Page Checks** | TDK · H1 · 标题层级 · 字数 · 内链 · 社交标签 |
| **Priority Actions** | 影响最大的 3 项修复，按优先级排序 |
| **Insight Walkthrough** | 每个重要发现的 Evidence → Impact → Fix |

```
audit this page: https://openclaw.ai
→ ✅ Report saved → reports/openclaw-ai-audit.html
```

| 站点 | Audit Summary | Site Checks | Page Checks & insights |
|---|---|---|---|
| colaos.ai <br><small><code>reports/colaos-ai-audit.html</code></small> | <img src="assets/0-0.png" alt="审计摘要" width="240" /> | <img src="assets/0-1.png" alt="站点检查" width="240" /> | <img src="assets/0-2.png" alt="页面检查与洞察" width="240" /> |

---

## 架构：Script + LLM 双层设计

```
URL
 │
 ▼
┌──────────────────────────────────────────────────┐
│  Layer 1 · Python 脚本                            │
│  确定性检查 → 结构化 JSON                          │
│                                                  │
│  check-site.py      sitemap 目录盘点、测试子域名、robots │
│  check-page.py      H1 / title / meta / canonical│
│  check-schema.py    JSON-LD 字段 + 多语言对齐校验   │
│  fetch-page.py      原始 HTML + SSRF 防护          │
└───────────────────────┬──────────────────────────┘
                        │ JSON + llm_review_required 标志
                        ▼
┌──────────────────────────────────────────────────┐
│  Layer 2 · LLM Agent                             │
│  仅对标记字段进行语义判断                           │
│                                                  │
│  · 关键词意图对齐（H1 / Title）                     │
│  · Meta Description 质量与具体性评分                │
│  · 页面类型 → 期望 Schema @type 映射               │
│  · E-E-A-T 信任页面可达性（footer/nav 链接）         │
│  · 内容分析（字数、标题层级、内链）                   │
└───────────────────────┬──────────────────────────┘
                        │
                        ▼
              report-template.html
              → reports/<hostname>-audit.html
```

**为什么分两层？** 脚本处理 80% 的确定性检查——robots.txt 是否存在？Title 是否 55 个字符？LLM 处理 20% 需要理解力的判断——这个 H1 在语义上是否覆盖了"AI workflow automation"的搜索意图？`llm_review_required` 标志确保 LLM 仅在脚本明确无法判断时才介入——事实性检查不会产生幻觉，语义性检查不会有盲区。

---

## Skill 说明

| Skill | 层级 | 适用场景 |
|---|---|---|
| `seo-audit` | Basic | 默认入口 — 给一个 URL，输出结构化首轮检查 |
| `seo-audit-full` | Full | 深度审计：PageSpeed、Sitemap 页面资产盘点、社交标签、内容质量评分、GSC 数据、竞品差距分析 |

> `seo-audit-full` 的 PageSpeed 检查必须配置 Google PageSpeed Insights API key。可以设置 `PAGESPEED_API_KEY` / `GOOGLE_PAGESPEED_API_KEY`，或通过 `--api-key` 传入；没有 key 时，full audit 会停止并提示先配置。

---

## 审计覆盖范围

### 站点级检查

| 检查项 | 检查内容 | Basic | Full |
|---|---|:---:|:---:|
| Sitemap URL Inventory | 独立表格展示 sitemap 里的主要目录：URL 数量、页面类型、代表性示例 URL | — | ✅ |
| 测试子域名收录风险 | 检测公开的 `test.`、`staging.`、`dev.`、`preview.`、`beta.`、`uat.` 是否镜像正式站并可能被索引 | — | ✅ |
| robots.txt | RFC 9309 指令组解析、Allow/Disallow 逻辑、Googlebot 状态、Sitemap 指令 | ✅ | ✅ |
| sitemap.xml | 有效 XML、URL 数量、追踪 robots.txt 声明的 Sitemap 路径，必要时抽样展开子 sitemap | ✅ | ✅ |
| 404 处理 | 真 404 vs 软 404（200）vs 跳转首页（301） | ✅ | ✅ |
| URL 规范化 | HTTP→HTTPS 重定向、www 一致性、尾斜杠、Canonical 标签匹配 | ✅ | ✅ |
| i18n / hreflang | 互相引用对称、BCP 47 语言码、x-default、默认语言 URL 重复、canonical/hreflang 对齐 | ✅ | ✅ |
| Schema（JSON-LD） | JSON 可解析性、@type、必填/推荐字段、嵌套字段、类型冲突、多语言 schema 的语言和 URL 对齐 | ✅ | ✅ |
| E-E-A-T 信任页面 | About / Contact / Privacy / Terms — 页面存在（HTTP 200）+ footer/nav 可达 | ✅ | ✅ |
| GSC 抓取状态 | 索引覆盖、抓取错误、被屏蔽资源 | — | ✅ |
| PageSpeed / Lighthouse | Performance · Accessibility · Best Practices · SEO 分数、实验室指标、最终 URL、截图可用性 | — | ✅ |

`Sitemap URL Inventory` 会在 Site Checks 里作为独立表格展示，位置在 Crawlability 之前：

| 目录 | URL 数量 | 页面类型判断 | 示例页面 |
|---|---:|---|---|
| `/blog/` | 300 | Blog / 内容页 | `https://xx.ai/blog/best-ai-video-tools` |
| `/tools/` | 80 | 工具页 | `https://xx.ai/tools/image-generator` |
| `/alternatives/` | 40 | 竞品替代页 | `https://xx.ai/alternatives/synthesia` |

它是站点级页面资产地图，不是单页 full audit 结论；后续可以从重要目录各选一个代表 URL 继续深度审计。

### 页面级检查

| 检查项 | 检查内容 | Basic | Full |
|---|---|:---:|:---:|
| URL Slug | 小写、连字符、含关键词、停用词 & 关键词堆砌检测 | ✅ | ✅ |
| Title 标题 | 50–60 字符、关键词位置、首页 vs 内页差异化规则 | ✅ | ✅ |
| Meta Description | 120–160 字符、关键词匹配、具体价值主张（非空泛描述） | ✅ | ✅ |
| H1 标签 | 唯一 H1、关键词匹配（full / partial / none）、语义意图复审 | ✅ | ✅ |
| Canonical 标签 | 自引用、与重定向后最终 URL 一致 | ✅ | ✅ |
| 图片 Alt 文本 | 所有 `<img>` 的 alt 属性检查、JS 渲染检测 | ✅ | ✅ |
| 字数统计 | 正文 ≥ 500 词、薄内容标记 | ✅ | ✅ |
| 关键词位置 | 主关键词出现在正文前 100 词内 | ✅ | ✅ |
| 标题层级结构 | H2 数量（目标 5–7）、H3/H2 比例、关键词在 H2 中的分布 | ✅ | ✅ |
| 内部链接 | 同源链接数（排除 nav/footer）、权重分配 | ✅ | ✅ |
| OG / 社交标签 | og:image、twitter:card、社交预览完整性 | — | ✅ |
| 内容质量 | E-E-A-T 深度、可读性、与竞品的具体程度对比 | — | ✅ |
| Robots Meta | noindex、nofollow、max-snippet 指令 | — | ✅ |

---

## 目录结构

```
seo-audit-skill/
├── seo-audit/
│   ├── SKILL.md                       # Skill 定义 + Agent 工作流
│   ├── references/REFERENCE.md        # 字段定义、边界情况
│   ├── assets/report-template.html    # HTML 报告模板
│   └── scripts/
│       ├── check-site.py              # robots.txt + sitemap → JSON
│       ├── check-page.py              # TDK + H1 + canonical + slug → JSON
│       ├── check-schema.py            # JSON-LD 提取 + 校验 → JSON
│       └── fetch-page.py              # 原始 HTML 抓取，SSRF 防护
└── seo-audit-full/
    ├── SKILL.md
    ├── references/REFERENCE.md
    ├── assets/report-template.html
    └── scripts/
        ├── check-site.py              # 测试子域名 + sitemap inventory + robots/sitemap → JSON
        ├── check-pagespeed.py         # PageSpeed Insights / Lighthouse → JSON
        ├── check-social.py            # OG + Twitter Card 校验 → JSON
        └── check-schema.py            # JSON-LD 质量 + 多语言 schema 校验 → JSON
```

---

## 安装

**方式一：CLI 安装（推荐）**

```bash
npx skills add JeffLi1993/seo-audit-skill

# 安装指定 Skill
npx skills add JeffLi1993/seo-audit-skill --skill seo-audit
npx skills add JeffLi1993/seo-audit-skill --skill seo-audit-full
```

**方式二：Claude Code Plugin**

```bash
/plugin marketplace add JeffLi1993/seo-audit-skill
/plugin install seo-audit-skill
```

## 使用

```
audit this page: https://example.com
```

```
deep audit: https://example.com
```

---

## 内置脚本

所有脚本均输出结构化 JSON 到 stdout。退出码 `0` = 通过/警告，`1` = 存在失败项。

| 脚本 | 功能 |
|---|---|
| `check-site.py` | robots.txt + sitemap + sitemap inventory — RFC 9309 解析、测试子域名检测、主要目录聚合 |
| `check-page.py` | H1 / title / meta / canonical / URL slug — 停用词感知的关键词匹配 |
| `check-schema.py` | JSON-LD 提取、@graph 展平、@type + 必填字段校验、多语言 schema 对齐 |
| `fetch-page.py` | 原始 HTML 抓取 — SSRF 防护、重定向链追踪、Googlebot UA 选项 |
| `check-pagespeed.py` | Full only — PageSpeed Insights / Lighthouse 分数和实验室指标；必须配置 API key |
| `check-social.py` | Full only — OG 标签 + Twitter Card 校验 |

**依赖：** `pip install requests`

---

## License

MIT
