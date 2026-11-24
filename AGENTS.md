# AGENTS.md

## Project Overview

本仓库维护个人自用的 **Legado（开源阅读）书源** 与 **单文件直链 RSS 订阅源**，托管于公开 GitHub 仓库 `https://github.com/entr0pia/legado-source`。

- **分发方式**：不启用 GitHub Pages，不生成静态网页；通过单个 `bookSources.json` 直链（优先 `fastly.jsdelivr.net` 国内直连，备用 `cdn.jsdelivr.net` 与 `raw.githubusercontent.com`）完成书源合集网络导入与 Legado RSS 订阅源解析。
- **核心工具**：
  - 本地零依赖构建脚本：[scripts/build.py](scripts/build.py)（Python 3 + `uv`）
  - 实机调试脚本：[legado-debug.py](.agents/skills/legado-book-source-skill/scripts/legado-debug.py)
  - 书源开发技能：[SKILL.md](.agents/skills/legado-book-source-skill/SKILL.md)

## Repository Structure & Git Tracking Boundary

**强约束：GitHub 远端严格只保留以下追踪项，由本地白名单 `.gitignore` 控制（其余本地工具/备份文件均不提交）。**

```text
legado-source/
├── src/                        # [追踪] 单条书源 JSON（严禁新建二级子目录！）
│   └── 轻小说文库.json          # 文件名必须与 bookSourceName 一致
├── bookSources.json            # [追踪] 由 scripts/build.py 自动合并的书源数组
├── rssSource.json              # [追踪] 由 scripts/build.py 自动生成的单文件直链 RSS 源
├── AGENTS.md                   # [追踪] 本指南文件
├── scripts/build.py            # [仅本地] 合并书源并生成 rssSource.json
├── backups/                    # [仅本地] 书架与旧书源备份（含隐私数据，严禁提交）
├── .agents/                    # [仅本地] Agent 技能与调试脚本
└── .gitignore                  # [仅本地] 白名单忽略规则（连自身也不提交）
```

- **单书源命名约定**：`src/` 下禁止建二级目录；文件名必须严格等于书源的 `<bookSourceName>.json`。原因：`rssSource.json` 的「一键导入此书源」按钮会根据当前 `bookSources.json` 的直链自动推算 `.../src/<encodeURIComponent(bookSourceName + '.json')>`。
- **禁止手改生成产物**：新增或修改书源一律在 `src/*.json` 中完成，再运行构建脚本更新 `bookSources.json` 和 `rssSource.json`。

## Setup & Build Commands

运行环境为 Windows 11 + PowerShell，Python 统一通过 `uv` 调用，务必加 `$env:PYTHONUTF8=1` 避免中文乱码：

```powershell
# 1. 校验 src/*.json 去重并重新生成 bookSources.json 与 rssSource.json
$env:PYTHONUTF8=1; uv run python scripts\build.py

# 2. 自定义直链、订阅源名称或分组时（可选）
$env:PYTHONUTF8=1; uv run python scripts\build.py --raw-url <JSON直链> --name "我的书源仓库" --group "自建"
```

## Development & Testing Workflow (TDD)

创建、修复或调试任何书源/订阅源时，须遵循 `.agents/skills/legado-book-source-skill/SKILL.md` 的 TDD 循环：

1. **前置检查**：确认手机与本机在同一局域网、手机已解锁且 Legado 已开启「设置 → Web 服务」（当前默认地址 `192.168.0.46:1122`）。
2. **连通性探活**：
   ```powershell
   curlx -s --connect-timeout 5 -o NUL -w "HTTP %{http_code}`n" 'http://192.168.0.46:1122/getBookSources'
   ```
3. **逐阶段调试顺序**：`正文 → 目录 → 详情 → 搜索 → 发现`。每个阶段先记录预期值，再改规则并立刻实机验证，禁止批量改完再测。
4. **常用实机调试命令**（用 `python -u` 无缓冲输出）：
   ```powershell
   # 调试正文页（--URL）
   $env:PYTHONUTF8=1; uv run python -u .agents\skills\legado-book-source-skill\scripts\legado-debug.py --host 192.168.0.46 --source .\src\轻小说文库.json --key='--https://www.wenku8.net/novel/4/4476/182005.htm,{"webView":true}'

   # 调试目录页（++URL）
   $env:PYTHONUTF8=1; uv run python -u .agents\skills\legado-book-source-skill\scripts\legado-debug.py --host 192.168.0.46 --source .\src\轻小说文库.json --key='++https://www.wenku8.net/novel/4/4476/index.htm,{"webView":true}'

   # 调试详情页（完整 URL）
   $env:PYTHONUTF8=1; uv run python -u .agents\skills\legado-book-source-skill\scripts\legado-debug.py --host 192.168.0.46 --source .\src\轻小说文库.json --key='https://www.wenku8.net/book/4476.htm,{"webView":true}'

   # 调试搜索页（关键字）
   $env:PYTHONUTF8=1; uv run python -u .agents\skills\legado-book-source-skill\scripts\legado-debug.py --host 192.168.0.46 --source .\src\轻小说文库.json --key="人妻"

   # 调试发现页（分类名::URL）
   $env:PYTHONUTF8=1; uv run python -u .agents\skills\legado-book-source-skill\scripts\legado-debug.py --host 192.168.0.46 --source .\src\轻小说文库.json --key='轻小说列表::/modules/article/articlelist.php?page={{page}},{"webView":true}'

   # 仅保存书源到手机（不调试）
   $env:PYTHONUTF8=1; uv run python .agents\skills\legado-book-source-skill\scripts\legado-debug.py --host 192.168.0.46 --source .\src\轻小说文库.json --save-only

   # 调试并保存个人 RSS 订阅源
   $env:PYTHONUTF8=1; uv run python -u .agents\skills\legado-book-source-skill\scripts\legado-debug.py --host 192.168.0.46 --source .\rssSource.json --rss
   ```
5. **失败排查铁律**：调试失败时**禁止盲改选择器**，先备份快照并在失败字段中加入 `<js>java.log(...)</js>` 探针确认实际返回的状态码、页面标题与 DOM 结构，定位完成后务必清理探针。

## Code Style & Legado Rule Gotchas

以下是本仓库在手机端 Legado 实测踩坑总结，编写规则时必须遵守：

- **Rhino JS 兼容性**：
  - Java 返回的字符串对象调用 `.replace()` 前必须 `String(obj).replace(...)`，避免 Rhino 方法重载歧义报错。
  - 尽量使用 ES5 语法与 `var`（Rhino 中 `const` 在循环内存在作用域问题）。
- **Cloudflare 与 WebView 策略**：
  - OkHttp 直连遇 Cloudflare 403 且系统 WebView 能直接静默放行时，在各阶段 URL 末尾统一追加 `,{"webView":true}`，并开启 `enabledCookieJar: true` 与 `header: {"User-Agent": java.getWebViewUA()}`。
  - 谨慎在 `loginCheckJs` 中调用 `java.startBrowserAwait`，否则书架旧书后台刷新触发 403 时会反复弹出界面打扰用户；优先通过 `ruleToc.preUpdateJs` 自动为旧 `tocUrl` 补齐 `,{"webView":true}`。
  - 当前手机 Legado 版本的 `java.startBrowserAwait` 仅支持 2 参数 `(url, title)`，不支持 3 参数重载。
- **GBK 站点搜索编码**：`webView: true` 模式下 URL 选项里的 `"charset": "gbk"` 不会转换 GET 查询参数，中文关键词必须显式预编码：`{{java.encodeURI(key,'gbk')}}`。
- **选择器细节**：
  - `@css:` 前缀只能放在规则最前，不可在 `&&` 后重复书写。
  - Jsoup 的 `:eq(n)` 按父节点全部子元素索引计数，取同类型第 n 个标签优先用 Legado 原生 `tag.p.0` 语法。
- **RSS 订阅源规则细节**：
  - `/rssSourceDebug` 默认请求 `sortUrl` 首行 / `sourceUrl`，不会用 `--key` 覆盖。
  - `ruleArticles` 用 `<js>` 返回数组给后续 JsonPath（`$.title` 等）消费时，必须写 `JSON.stringify(out);</js>$.[*]`。
  - 若需要进入 `ruleContent` 渲染一键导入 HTML 按钮，`ruleDescription` 必须留空 `""`（非空会直接跳过内容页解析）。
  - 国内手机直连优先使用 `https://fastly.jsdelivr.net/gh/entr0pia/legado-source@main/bookSources.json`（实测 `raw.githubusercontent.com` 与 `cdn.jsdelivr.net` 在手机直连下易 `Connection reset`）。

## Git & Security Guidelines

- **提交前核验**：每次修改书源后先执行 `uv run python scripts\build.py`，再运行 `git status --short -uall`，确认变更仅限于 `src/*.json`、`bookSources.json`、`rssSource.json`、`AGENTS.md`。
- **提交前询问**：不要自动执行 `git add`、`git commit`、`git push`，提交和推送前须先征得用户确认。
- **隐私与数据安全**：
  - 严禁将 `backups/`（含个人书架书单 `bookshelf.backup.json` 与历史备份）纳入 Git 追踪。
  - 严禁在代码、日志汇报或提交记录中输出真实 Cookie 值或账号凭据。
  - 通过 Legado `/saveBook` 批量迁移书架旧书时，**切勿修改主键 `bookUrl`**（否则会产生重复书或丢失缓存/书签），应先全量备份 `/getBookshelf`，再原地仅更新 `tocUrl`。
