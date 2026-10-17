# cloudflare-zone-guardrails-audit

**Cloudflare「AI 爬虫被拦」隐形故障体检：让 AI 搜索读得到你的网站。**

[![GitHub stars](https://img.shields.io/github/stars/yueliangjiejie/cloudflare-zone-guardrails-audit?style=social&label=Star)](https://github.com/yueliangjiejie/cloudflare-zone-guardrails-audit/stargazers)
[![GitHub followers](https://img.shields.io/github/followers/yueliangjiejie?style=social&label=Follow)](https://github.com/yueliangjiejie)
[![Agent Skill](https://img.shields.io/badge/Agent-SKILL.md-7c3aed)](https://github.com/yueliangjiejie/cloudflare-zone-guardrails-audit)
[![License](https://img.shields.io/badge/%E8%AE%B8%E5%8F%AF-%E4%B8%AA%E4%BA%BA%E5%85%8D%E8%B4%B9%C2%B7%E5%95%86%E7%94%A8%E6%8E%88%E6%9D%83-orange)](LICENSE)

一个即装即用的 Agent Skill。你的网站浏览器访问一切正常，但 ChatGPT、豆包、元宝、Kimi 对你的品牌一无所知？大概率不是内容问题，是 Cloudflare 在 zone 层把 AI 爬虫拦了，而这种拦截用浏览器永远测不出来。

## 症状自查（中一条就要查）

- 浏览器访问正常，但 AI 助手完全不知道你的网站/品牌
- `robots.txt` 返回 200，`llms.txt`、`sitemap.xml` 却是 403
- 403 返回体是一段约 25 字节的 `text/plain`：`Your request was blocked.`
- 搜索引擎收录正常，AI 检索为 0
- 网页源码里的邮箱变成了 `[email protected]`

## 30 秒自检（不装任何东西）

```bash
for ua in GPTBot ClaudeBot PerplexityBot; do
  for p in / /llms.txt /sitemap.xml; do
    code=$(curl -s -o /tmp/body.txt -w "%{http_code}" -A "$ua" --max-time 20 "https://你的域名$p")
    hit=$(grep -c "Your request was blocked" /tmp/body.txt 2>/dev/null || echo 0)
    printf "%-14s %-14s %s  blocked=%s\n" "$ua" "$p" "$code" "$hit"
  done
done
```

出现 `403 + blocked=1` 即命中。修复方法见 SKILL.md（核心是三套互相独立的开关：Bot Fight Mode / AI Labyrinth / AI Crawl Control，以及一条会显式排除 `/robots.txt` 的 WAF 自定义规则）。

## 安装

**Claude Code（macOS / Linux）**

```bash
git clone https://github.com/yueliangjiejie/cloudflare-zone-guardrails-audit.git \
  ~/.claude/skills/cloudflare-zone-guardrails-audit
```

Windows 放入 `%USERPROFILE%\.claude\skills\`。其他支持 SKILL.md 约定的 Agent（如 OpenClaw）放入对应 skills 目录即可。

## 使用

装好后对你的 Agent 说：

> 帮我查一下 tanlulu.com 的 AI 爬虫可达性
> AI 搜不到我们，帮我体检是不是被 Cloudflare 拦了

Agent 会按 SKILL.md 的流程执行：带 AI 爬虫 UA 实测取证 → 区分两类拦截指纹 → 排除出口 IP 信誉干扰 → 给出逐开关的放行步骤 → 验收复测。

## 这个 skill 沉淀了什么（都是实测踩出来的）

| 模块 | 内容 |
| --- | --- |
| 取证方法 | 6 个 AI 爬虫 UA × 关键路径批量实测，附对照组设计 |
| 指纹区分 | 25 字节 text/plain（AI Crawl Control）vs `cf-mitigated: challenge`（Bot Fight），处理入口完全不同 |
| 混淆变量 | 出口 IP 信誉排除法 4 条，前三条不需要任何 zone 凭据，避免把自己的代理漂移误判成站点配置变化 |
| 逐 zone 独立 | 同一账号 N 个域名 = N 份配置，实测 4 站只有 1 站被拦 |
| 假报错识别 | 控制台 `API Request Failed: PATCH` 多半已生效；列表显示「活动」也未必没关掉，唯一权威判据是实测 |
| 致命默认项 | zone 级策略选朴素 Block 会连 Googlebot / BingBot 一起掐死，直接伤普通搜索排名 |
| 版本回灌 | 带语义化版本号与「反例保留」机制：被推翻的结论不删，保留反例警示后来者 |

## 实测效果

2026-10 用于我的个人网站 tanlulu.com（Cloudflare Pages）及另外三个站点体检：

- 1 个站命中「WAF 规则拦截 AI 爬虫」，按本流程放行后，GPTBot 抓取请求 +728.6%
- 另一个站命中「邮箱混淆」导致对外邮箱对 AI 不可读，已修复
- 其余站点确认全绿（逐 zone 独立，必须逐站实测）

完整复盘见我的行动营分享：从 AI 可见度诊断 35 分（20 问命中 2 次）到 47 分，GPTBot 抓取 +728.6%。

## 版本

当前 v1.2.0。变更记录与被推翻结论的反例均保留在 SKILL.md 文末。

## 作者与许可

**月亮姐姐（谭璐璐）** ｜ [tanlulu.com](https://tanlulu.com) ｜ 微信 moonai6

个人与学习用途可免费使用，须保留署名。商业用途（含付费服务、二次分发）需获得作者授权，见 [LICENSE](LICENSE)。

---

## ⭐ 觉得有用？

如果这个 skill 帮你查出了「AI 搜不到」的真凶，欢迎三件事：

1. **点个 Star**，让更多被同样问题困扰的站长能搜到它
2. **关注 [@yueliangjiejie](https://github.com/yueliangjiejie)**，后面还会开源 GEO 诊断、站点全维度体检等一整套 skill
3. **排查中遇到新情况？[提个 Issue](https://github.com/yueliangjiejie/cloudflare-zone-guardrails-audit/issues)**，你的坑会被回灌进版本记录，帮到下一个人

> 这个 skill 出自我个人网站的 GEO 改造复盘：6 天从「AI 查无此人」到 GPTBot 抓取 +728.6%，完整过程见 [tanlulu.com](https://tanlulu.com)。
