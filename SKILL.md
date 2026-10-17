---
name: cloudflare-zone-guardrails-audit
author: 月亮姐姐
description: Cloudflare zone 级「AI 爬虫拦截」类隐形故障的体检与放行。当用户说"AI 搜不到我们/豆包元宝说不知道这个品牌/llms.txt 或 sitemap 报 403/只有 AI 爬虫访问失败而浏览器正常/站点在控制台找不到对应开关/网页源码里邮箱被替换成 [email protected]"时使用。覆盖三套互相独立的开关（Bot Fight Mode、AI Labyrinth、AI Crawl Control 与 zone 级 AI 自动程序策略）、后台入口映射、两类拦截的指纹区分、出口 IP 信誉这个混淆变量，以及「只有带 AI 爬虫 UA 的 curl 才看得见」的实测判定法。
agent_created: true
version: 1.2.0
---

# Cloudflare zone 级拦截体检（AI 爬虫可达性）

## 为什么需要独立一套检查

这类故障的特征是**只对特定 UA 生效**：

- 普通浏览器访问一切正常
- `robots.txt` 返回 200，但 `llms.txt`、`sitemap.xml` 返回 **403**
- 返回体是一段 **`text/plain`、约 25 字节、内容 `Your request was blocked.`** 的裸页
- 搜索引擎收录正常下滑、AI 助手完全不知道你是谁

**只用浏览器或用普通 `curl -A curl` 测试，永远发现不了。** 必须带 AI 爬虫 UA 实测。

⚠️ 这是**逐 zone 独立**的：同一账号下 N 个域名 = N 份配置。**一个站被拦不代表别的站也被拦，必须逐站实测**（实测样例：4 个站里恰好只有 1 个被拦，另 3 个一直是 200）。

## 第 0 步：实测（只读，先取证再改设置）

```bash
# 6 个 AI 爬虫 UA × 关键路径，一次跑完
# <代理端口> 换成你本机代理软件的实际端口（如 7890/7897）；
# 网络可直连 Cloudflare 系站点时，可去掉 -x 参数。
for ua in GPTBot ClaudeBot PerplexityBot OAI-SearchBot CCBot Bytespider; do
  for p in / /llms.txt /sitemap.xml /robots.txt; do
    code=$(curl -s -o /tmp/body.txt -w "%{http_code}" -A "$ua" --max-time 20 \
      -x http://127.0.0.1:<代理端口> "https://example.com$p")
    size=$(wc -c < /tmp/body.txt | tr -d ' ')
    hit=$(grep -c "Your request was blocked" /tmp/body.txt 2>/dev/null || echo 0)
    printf "%-14s %-14s %s  %sB  blocked=%s\n" "$ua" "$p" "$code" "$size" "$hit"
  done
done
```

**一定要带对照组**：除了 AI 爬虫 UA，再跑一个**不在阻止名单里的 bot UA**（如 `Google-Extended`、`meta-externalagent`）。实测会出现「它们 200、AI 爬虫 403」——这一组差异本身就是证据。

判定：

| 观察 | 结论 |
|---|---|
| 全部 200，且 `/llms.txt` 体量与预期相符（非 25B） | ✅ 无拦截 |
| 出现 403 + 25B `text/plain` + `Your request was blocked.` | ❌ 被 **AI Crawl Control** 拦截，走第 1 步 |
| 出现 HTML 挑战页 + 响应头 `cf-mitigated: challenge` | ❌ 被 **Bot Fight / 托管挑战**拦住，见下「两类拦截的指纹区分」 |
| 只有 `/` 是 302 | 站点自身行为（常见于 `/` 跳子路径），加 `-L` 跟随后再判 |

> 代理相关：中国大陆网络直连 Cloudflare 系站点常返回 `000`，必须走本机代理。🔴 **只用 `-x http://127.0.0.1:<port>`，不要加 `--noproxy '*'`** —— `--noproxy` 是「不走代理的主机列表」，`*` = 所有主机，它会把 `-x` **整个取消**（这是 curl 语义，不是被 env 变量盖住）→ 请求被降级为**本机 fake-ip 直连**（verbose 里可见 `Trying 198.18.0.x:443`，且**没有** `CONNECT tunnel established`），通不通取决于代理软件的 TUN/fake-ip 路由是否在跑，它一掉线就**全返 `000` + `SSL_ERROR_SYSCALL`**，看起来像"站点挂了"。shell 里已有的 `HTTPS_PROXY` 与 `-x` 同端口**并不冲突**（verbose 确认走 CONNECT 隧道）。

### 两类拦截的指纹区分（决定你去哪个页面处理）

| 判据 | AI Crawl Control（阻止 AI 爬虫） | Bot Fight Mode / 托管挑战 |
|---|---|---|
| 响应体 | **恰 ~25 字节 `text/plain`**：`Your request was blocked.` | **HTML 挑战页** |
| 响应头 | `cache-control: private, no-store` | **`cf-mitigated: challenge`** |
| 影响范围 | 按 **UA**（`用户代理 包含 <AI 爬虫列表>`） | 按**行为/指纹**，**只挑战 HTML 文档请求，放行静态非 HTML**（实测：`/`、`/pricing`、`/sitemap.xml` 挑战；`/llms.txt`、`/robots.txt`、`/assets/*.js` 放行） |
| 是否理会 `robots.txt` | **完全不理会** | **完全不理会** |
| 处理入口 | `AI Crawl Control` 页 / `安全性 › 安全规则` | `安全性 → 自动程序（Bots）` |

### 🔴 混淆变量：本机出口 IP 信誉（不下结论之前先排除）

如果本机走的是 **VPN / 代理节点**，**IP 信誉差本身就会触发托管挑战** —— 这与站点配置无关，症状却一模一样。

实测教训：观测期间出口节点反复掉线/切换，同一个站在 **40 分钟内从"干净"变成"挑战"** —— 那是**出口变了**，不是站点配置变了。

**排除法（🔑 前三条不需要任何 zone 凭据，先跑完再谈定罪）：**

1. **镜像域对照（最有力）**：同平台提供的镜像域名（静态托管的 `<project>.pages.dev` 之类）
   **不经过 zone 级安全规则**。它返回 200 → **产物与服务端都没问题**，问题只在"这个自定义域怎么放行你"。
2. **外部视角对照**：用第三方抓取服务（如 `https://r.jina.ai/<完整URL>`）从**完全不同的网络**抓一次。
   它 200 = **真实爬虫可达**，当前的挑战只针对你的出口 IP。
3. **同出口换站点**：同一出口访问**同账号下另一个 zone**（或另一个同平台站点）。
   它 200 而目标站被挑战 → 出口没被整体拉黑，**是目标 zone 的姿态更严**（这条同时验证了本 skill 开头"逐 zone 独立"的前提）。
4. **换出口（或直连）复测** + 到 zone → `安全性 → 设置` 核对 **Bot Fight Mode = OFF**、
   `安全性 → 安全规则` 自定义规则有无 `Managed Challenge`。

⇒ 🔴 **绝不要用「单次、单一出口」的挑战结果给站点配置定罪。**
⇒ 🔑 **反过来也要记住**：第 1、2 条同时 200、**唯独自定义域被挑战**，**不能**推成"站点没问题"就收工 ——
那恰好是"zone 层存在人为拦截"的指纹，仍须请用户核对第 4 条的开关。

## 三套互相独立的开关（逐 zone 各查一遍）

| 开关 | 期望状态 | 位置 |
|---|---|---|
| **Bot Fight Mode** | **关闭** | 安全性 → 自动程序（Bots） |
| **AI Labyrinth** | **关闭** | 同上 |
| **AI Crawl Control** | **关闭**（或 Crawlers 页签逐项 Allow） | 左侧菜单「AI Crawl Control」→ `Crawlers` |
| zone 级「配置 AI 自动程序策略」 | Search / Agent / Training **全部 Allow** | 安全性 → 设置（Settings） |

> `AI Crawl Control` 直链格式：`dash.cloudflare.com/?to=/:account/<域名>/ai`（也可在左侧菜单或 `⌘K` 搜到）。

### 后台入口映射（记死，这两个地方极易找错）

- **「配置 AI 自动程序策略」= zone → 安全性 → 设置**。它管 Search / Agent / Training 三类**账户级默认值**。
  ❌ 不是 `/<account>/ai/ai-search` —— 那个是 **AI Search**（做 RAG 建实例的产品，与本问题无关）。
- **「需求信号 N」/ 逐爬虫明细与允许阻止 = AI Crawl Control**（原 AI Audit）。
  → `Optimization` 标签下的 *"Check what AI Assistants are trying to access, but can't"* 才是"谁被挡住了"的入口；`Crawlers` 标签里逐项改 **Allow**。
  ❌ 不在 AI Search 页面。

### 🔴 最快的定位路径：直接看 WAF 自定义规则

**AI Crawl Control 的"阻止"最终会落地成一条 WAF 自定义规则**（不是纯设置项）：

```
名称：AI Crawl Control - Block AI bots…
条件：URI 路径 不等于 /robots.txt  AND  用户代理 包含 <AI 爬虫列表>
操作：阻止        状态：活动
```

位置：**`安全性 › 安全规则`（Security Rules）→ 过滤 `自定义规则`**。

这条规则**显式排除 `/robots.txt`** —— 所以 `robots.txt` 200 而其余路径 403。这个"排除 robots.txt"的特征与实测现象**一一对应**，可以反向确认真凶。

> ⚠️ **同页常有一条 `Block scanner probes`**（拦 `/wp-admin`、`/wp-*.php` 等扫描路径，事件数往往很高）。那是**正常防护，绝对不要动**。

### 层级关系（决定你去哪儿改才有效）

- `AI Crawl Control` 里改 `Crawlers` 时若提示 **"This crawler is being blocked by an existing AI bot policy"**，说明**被 zone 级总开关压着**（总开关开着时 per-bot 开关是灰的）→ 必须回到 **安全性 → 设置 → 配置 AI 自动程序策略** 把三类全设 **Allow**，再回来改。
- 🔴 **致命默认项**：选项里的 **「阻止 Block」≠「只阻止 AI 训练」**。选朴素 Block 会**连 Googlebot / Applebot / BingBot 一起掐死**（跨类命中时取**最严**规则）→ 直接伤普通搜索排名与 AI Overviews。要么选精确的"阻止训练/爬取"项，要么全部 Allow。
- ⚠️ 平台默认会变：历史上有过「新域名默认阻止 AI 爬虫」的默认值变更，**后加的 zone 更容易吃到**（表现为别的站好好的、只有新站点 403）。所以**不能靠"我上次查过没问题"来判断新站**。

## 改设置的纪律

1. 🔴 **每个开关是一次独立的 API 调用，连点会互相覆盖**（有实测：一口气点 14 个，只生效 2 个）→ **一次只改一个，改完刷新页面确认落盘**。
2. 🔴🔴 **「报错」≠「失败」** —— 改托管的 AI Crawl Control 规则时，控制台可能弹
   `API Request Failed: PATCH /zones/…/rulesets/…/rules/… (undefined)`，
   但**改动往往已经落到边缘**。原因是 CF 前端先乐观更新、后端同步报错、列表页又没刷新。
   反过来也成立：**列表里状态仍显示「活动」不代表没关掉**（那一页没刷新）。
3. ✅ **判定"禁用是否真的生效"的权威判据**（比红条和列表都可靠）：
   打开 **`AI Crawl Control` → `安全` 标签**，若顶部出现黄色横幅
   > **「AI Crawl Control 安全规则已禁用。请重新启用它。」**

   那是 Cloudflare **亲口承认"规则已经关掉了"**（它检测到 WAF 规则与 AI Crawl Control 配置不一致）。
4. 🔴🔴 **那条黄条里的「重新启用它」链接千万不要点** —— 一点就把 AI 爬虫拦截**原样恢复**。让它挂着，不影响任何东西。
5. 🔴 **最终判据永远是第 0 步的实测**，不是控制台回显、不是红条、不是列表状态。改完立刻重跑。
6. ⚠️ 也有**真的改不动**的情形：官方文档明确写着
   > "Changes made directly to the WAF rule are **not reflected back** in the AI Crawl Control dashboard."

   ⇒ 若直改 WAF 确实不生效，**正规入口是 `AI Crawl Control` 页的 `Crawlers` 标签**（它才是主界面）；另一条兜底是**新建一条优先级更高的放行规则**（新建规则不会撞上这个 PATCH 限制）。

> **顺序建议**：先在 WAF 规则页试一次禁用（最直接）→ 无论报什么错，**都去 AI Crawl Control › 安全 页看有没有那条黄条** → 再跑第 0 步实测确认 403 变 200。三步都做完才下结论。

## 顺带必查的三项

### 邮箱混淆（Scrape Shield / Email Obfuscation）

站点可见邮箱会被 CF 改写成 `[email protected]` —— 这会让"站上承诺的对外邮箱"在 AI 与用户眼里**完全不可读**，直接伤实体确权。

- 期望状态：**关闭**
- 验法（要数**两个方向**）：
  ```bash
  curl -s https://example.com/ > /tmp/p.html
  grep -c 'cdn-cgi/l/email-protection' /tmp/p.html   # 期望 0
  grep -o 'hello@example.com' /tmp/p.html | wc -l     # 期望 >0（真实地址仍在）
  ```
  实测样例：`cdn-cgi/l/email-protection` 命中 **0** 次、页面里两处真实邮箱分别明文出现 **6 处 / 4 处** → 未生效 ✅。

### `robots.txt` 是否被 CF 动态改写

CF 的托管 robots 功能会在你的 `robots.txt` 里**插入一段自己的内容块**。

```bash
curl -s https://example.com/robots.txt | grep -n "BEGIN Cloudflare Managed content"
```

命中 = 被托管改写（多源自托管方案会出现，与"是否拦 AI 爬虫"是两件事）。**注意：这条来自公开资料，本项目未亲自复现过，请以实测结果为准。**

### zone 级「配置缓存」

**不要动。** 与本问题无关，误改会引入新的缓存问题（例如打乱静态宿主自身的缓存策略）。

## ⚠️ 控制台的「快速入门指南」不是待办清单

CF 首页的快速入门 / Security Insights 建议列表是**功能入口目录**，不是"都要完成"。逐项结论：

| 项目 | 建议 | 原因 |
|---|---|---|
| **Bot Fight 模式** | 🔴 **绝不能开** | 按行为指纹判定、**不遵守 robots 的 Allow**，会把 GPTBot/ClaudeBot/Bytespider 一起挑战掉 → 掐死 GEO 与 AI 可见度 |
| **Under Attack 模式** | 🔴 **仅被 DDoS 时临时开** | 给每个访客加等待页，人和爬虫一起劝退 |
| 开发模式 | 仅调试时开 | 绕过缓存、不计统计 |
| Page Shield | ⚪ 无对象可开 | 它监控第三方脚本；纯静态站没有 |
| 泄露凭据检测 | ⚪ 无对象可开 | 它给"用泄露密码登录"的 IP 限速；无登录表单的站没作用对象 |
| **配置缓存** | ⚠️ **不要动** | 静态宿主已自管缓存，再加 zone 级规则会打架 |
| 域名转移 / 区域 ID / 速度测试 | 非待办 | 有的是展示信息，转移还会强制多续一年 |

## 权限现实（别浪费时间）

`wrangler` 的 OAuth 凭据通常只有 **Workers / Pages / DNS** 权限，读 zone 级数据一律 **`9109 Unauthorized`**
（实测：`zones/<id>/settings/security_level` 与 `bot_management` 都读不到）。

**推论：AI Crawl Control / 安全性设置 / 逐爬虫明细这类 zone 级内容，AI 无法代查代改，只能请用户自己在控制台确认。** 不要反复尝试 API。

> 例外：同一套凭据**能直接调 Pages 接口**（读项目配置、列删部署、建 Deploy Hook）。所以"平台层查不到"只限 zone 级安全数据，别一并放弃。

## 验收口径

- 6 个 AI 爬虫 UA × 关键路径**全部 200**，且无 `Your request was blocked.`
- 对照组（普通浏览器 UA / 无 UA）同样 200
- `robots.txt` **不是**唯一 200 的路径（这条本身就是"排除 robots 的拦截规则"的特征，修好后应全绿）
- 页面可见邮箱为**真实地址**，无 `[email protected]`
- `AI Crawl Control › 概览` 页的「失败/被拦」计数不再增长（实测样例：修复后概览显示 AI 请求 114 次、允许 71 / 失败 43，且热门路径已收到 200）
- **T+1 复测**仍成立（历史上出现过状态回去的情况）

## 坑

- 🔴 用普通 UA 测"站点通不通"，得到 200 就以为没事 —— **AI 爬虫专用拦截完全测不出来**。
- 🔴 **看到 `API Request Failed: PATCH` 就以为改不动** → 其实多半已生效；**反过来，看到列表仍标"活动"就以为没改掉** → 其实只是页面没刷新。**只能以实测为准。**
- 🔴 **手贱点了黄条里的「重新启用它」** → 拦截原样恢复，前功尽弃。
- 🔴 **顺手禁用了同页的 `Block scanner probes`** → 拆掉了正常防护。
- 🔴 把 zone 级默认策略选成朴素 **Block** → 连正统搜索引擎一起封。
- 🔴 **只在其中一个 zone 上验证就下全站结论** → 每个域名是独立配置，必须逐站测。
- 🔴 **用单次 VPN 出口的结果给站点定罪** → 出口 IP 信誉会自造一模一样的挑战现象。
- ⚠️ 大陆网络下不挂代理直连 CF 站点会返回 `000`，会被误读成"站点已下线"。
- 🔴 **在 `curl -x` 后面多写了 `--noproxy '*'`** → `-x` 被取消 → 请求降级为 fake-ip 直连 → 代理软件的 TUN 一掉线就**全返 `000` + `SSL_ERROR_SYSCALL`**，看起来像站点不通。（旧版本这条写反了，见「反例保留」）

- 🔴 **用第三方抓取服务验收「线上是不是新版」却不加 cache-buster** → 拿到的是**抓取服务自己的缓存**（旧版本），会把"部署成功"误判成"部署没生效"。加 `?cb=$(date +%s)` 再比。**凡"看起来还是旧版"，先排除中间层缓存（CDN / 抓取服务 / 浏览器），再怀疑部署。**
- 🔴 **拿单个出口的 `cf-mitigated: challenge` 直接判「站点被拦」** → 先跑本节排除法的第 1、2 条（镜像域 + 第三方抓取）。实测：一个站对本机出口全返挑战，而**镜像域 200、第三方抓取也 200** —— 真实爬虫根本没被拦，此前差点据此误报一次 P0。
- 🔴 **只拿"另一个 zone 正常"就推翻全部怀疑** → 反向也成立：同出口下**别的 zone 正常、目标 zone 被挑战**，正是"目标 zone 自己更严"的证据，不能用来证明它没问题。
## 变更记录

| 版本 | 日期 | 变更 |
|---|---|---|
| 1.0.0 | 2026-10-14 | 首次创建。来自三站点（含自建服务器站与两个 CF Pages 站）的 AI 爬虫拦截排查：定型"带 AI UA 的 curl"取证法、三套独立开关与两处易错的入口映射、以及「Block 会误伤 Googlebot」这条致命默认项 |
| 1.1.0 | 2026-10-14 | 回灌原始取证补全。①**订正**第 2 条纪律：`API Request Failed: PATCH` 是**假报错**，改动多半已落边缘；并补上权威判据（`AI Crawl Control › 安全` 的黄条）与「黄条里『重新启用它』绝不能点」；②新增「两类拦截的指纹区分」表（25B text/plain vs `cf-mitigated: challenge`，及 Bot Fight 只挑战 HTML 文档）；③新增**出口 IP 信誉**这个混淆变量与排除法（曾把出口漂移误判成站点配置变化）；④新增「WAF 自定义规则里最快定位真凶」（规则名 + 排除 `/robots.txt` 的对应特征）+ 官方"直改不回传"原文 + 新建高优先放行规则这条兜底；⑤新增「快速入门指南不是待办清单」逐项表；⑥新增逐 zone 独立性的实测反例与 AI Crawl Control 直链 |
| 1.1.1 | 2026-10-14 | **推翻一条硬规则**：`curl -x` 与 `--noproxy '*'` **不能同时用** —— 后者会把 `-x` 整个取消，探测降级为 fake-ip 直连（verbose 实证：无 `CONNECT tunnel established`），成败取决于代理软件 TUN 路由是否在线，掉线即全返 `000` + `SSL_ERROR_SYSCALL`，伪装成"站点不通"。实证：同一批 39 条带 AI 爬虫 UA 的请求，旧形式（带 `--noproxy`）首轮全 `000`，去掉后同批命令全 `200`。第 0 步示例命令已同步去掉该参数；旧写法作为反例保留 |
| 1.2.0 | 2026-10-16 | 把「出口 IP 信誉」的排除法从 **2 条扩到 4 条**，并指出前三条**不需要任何 zone 凭据**即可完成：新增①**镜像域对照**（`*.pages.dev` 不经 zone 级规则 → 200 即证明产物与服务端无恙）、②**外部视角对照**（第三方抓取服务 200 = 真实爬虫可达）、③**同出口换站点**（别的 zone 正常、目标 zone 被挑战 = 目标 zone 自己更严）；④保留原「换出口 + 核对后台开关」。并补一条**反向判据**：1、2 条同时 200 而唯独自定义域被挑战，**不能**据此收工，那正是 zone 层人为拦截的指纹。坑新增「第三方抓取服务自带缓存 → 不加 cache-buster 会把「部署成功」误判成「没生效」」 |

## 反例保留（不许静默删）

- **1.0.0–1.1.0 的写法**：「`curl -x` 必须**同时**加 `--noproxy '*'`，否则 shell 里已有的 `HTTPS_PROXY` 会盖住 `-x`。」
  这句**方向是反的**：`--noproxy '*'` 不是"压住 env"，而是**直接禁用代理**（`*` = 所有主机都绕过）→ `-x` 形同不存在。
  ⇒ 保留此反例，提醒后来者：**"多写一个看起来更保险的参数"可能正好是故障源**；遇到「全 `000` / `SSL_ERROR_SYSCALL`」，先看 verbose 里有没有 `CONNECT tunnel established` —— 没有就说明根本没走代理，别急着给站点"定罪"。

- **1.0.0 的写法**：「托管 WAF 规则不能在 WAF 页面删（会报 `API Request Failed: PATCH …`，这是假报错）。只能从上面的正规入口关。」
  这句话**本身没错但不完整**，会让人以为"点了没效果、白费劲"。实际是**可能已生效**。
  ⇒ 保留此反例，提醒后来者：**不要把"报错"直接等同于"失败"**，两者都要用实测裁决。

---

© 月亮姐姐 ｜ https://tanlulu.com ｜ 微信 moonai6 ｜ GitHub @yueliangjiejie

个人与学习用途可免费使用，须保留本署名。商业用途（含付费服务、二次分发）需获得作者授权。
