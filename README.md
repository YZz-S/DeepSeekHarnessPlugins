# DeepSeekHarnessPlugins

> YZz-S 的 DeepSeek Harness（dsh）开源插件集总览 · A hub for YZz-S's open-source DeepSeek Harness plugins

各插件代码在各自独立仓库维护，本仓库是一页式「驾驶舱」：**装什么、怎么装、实测结果**，以及 2026-10-11 全量实测中发现的生态注记。各插件 README 里的 `Back to DeepSeekHarnessPlugins` 即指这里。

**实测环境基线**：dsh 0.2.0-rc.2 · Node v26.10.0 · pnpm 12.10.1 · Windows 11 · 2026-10-11
（原始验证记录见 [docs/test-log-2026-10-11.md](./docs/test-log-2026-10-11.md)）

## 插件总表

| 仓库 | 类型 | 一句话 | 安装命令 | 本次实测 |
| --- | --- | --- | --- | --- |
| [dsh-notifier](https://github.com/YZz-S/dsh-notifier) | Host | 任务完成/需确认时发系统原生通知（Win Toast/macOS/Linux） | `dsh plugin --profile web add github:YZz-S/dsh-notifier` | ✅ 挂载（事件驱动，需真实会话回合触发） |
| [dsh-token-cost-meter](https://github.com/YZz-S/dsh-token-cost-meter) | Host+Client | 输入框下方实时费用行（官方动态定价） | `dsh plugin --profile web add github:YZz-S/dsh-token-cost-meter` | ✅ `/api/token-cost-meter/pricing` 实拉官方定价成功 |
| [dsh-billing-balance](https://github.com/YZz-S/dsh-billing-balance) | Host+Client | DeepSeek/火山引擎余额三处展示 | `dsh plugin --profile web add github:YZz-S/dsh-billing-balance` | ✅ `/api/billing-balance/status`（余额需填 key 后出数） |
| [dsh-update-checker](https://github.com/YZz-S/dsh-update-checker) | Host+Client | 顶栏版本徽章 + npm 更新检查 | `dsh plugin --profile web add github:YZz-S/dsh-update-checker` | ✅ `/api/dsh-update-checker/check` 200（`current` 检测局限见注记 4） |
| [dsh-workspace-files-explorer](https://github.com/YZz-S/dsh-workspace-files-explorer) | Host+Client | 工作区文件浏览 + 代码/Markdown 预览 | `dsh plugin --profile web add github:YZz-S/dsh-workspace-files-explorer` | ✅ `POST /api/wsf-explorer/root` 返回目录 |
| [dsh-dshx](https://github.com/YZz-S/dsh-dshx) | Host | MCP 客户端中枢 + 扩展管理台（MCP/记忆/技能/命令/提示词） | `dsh plugin --profile web add github:YZz-S/dsh-dshx` | ✅ 6 行全挂载，`/dshext/` 控制台 200（依赖引导见注记 2） |
| [dsh-bili-summary](https://github.com/YZz-S/dsh-bili-summary) | Host（工具） | 注册 `bili_summary`：B 站元数据+字幕时间轴+切帧配图 | `dsh plugin --profile web add github:YZz-S/dsh-bili-summary` | ✅ 离线自检 23/23 过，sharp 切帧可用 |
| [dsh-modlens](https://github.com/YZz-S/dsh-modlens) | Host（工具+模型） | ModLens 分叉：纯文本大模型的视觉桥（`modlens_read_image`，多引擎） | `dsh plugin --profile web add github:YZz-S/dsh-modlens` | ✅ 挂载（TN 仓库曾坏、已修，见注记 1；真实识图需配引擎 key） |
| [awesome-dsh-plugin](https://github.com/YZz-S/awesome-dsh-plugin) | 精选列表 | dsh 社区插件精选（CC0，非插件） | — | — |

## 两种挂法

每个插件都同时支持（二选一）：

1. **安装为 dsh.bundle（推荐，随 dsh 常驻）**：见上表 `dsh plugin` 命令；内部转发 pnpm，profile 首次使用自动初始化。
2. **动态 cordis_define（临时）**：在 dsh 会话里用 Cordis 工具粘贴 `host.js`(Host 半边)，双半边插件再加 `client.js`（Client 半边）；进程级存活，dsh 重启后要重.define。

Client 半边生效需在页面批准一次；Host-only 插件（notifier、bili-summary）无需刷新页面。

## 本次实测注记（2026-10-11）

### 注记 1 · dsh-modlens：两个仓库级阻断已修复

GitHub 直装曾有两大问题（已修复，commit `f544f56`）：

- **`dsh/index.js` GBK 乱码损坏**：`—`/`…` 等字符经 GBK 往返变成 `鈥`+私用区字符，83 处注释污染 + 第 852 行模板字符串的闭合反引号被吞 → 整个模块 ESM 语法非法，`modlens` 行在**任何** GitHub 安装上都无法激活。此问题自仓库 initial commit（2026-08-15）即存在。
- **`dist/` 从未入库**：插件从包内相对路径 spawn `../dist/main.js`（无 PATH 查找、无 npx 兜底，版本锁定设计），而 dist 被 gitignore、仓库亦无 prepare 脚本 → GitHub 安装后引擎缺失。现 `dist/main.js` 已随修复提交入库。

修好后的验证：行正常挂载、`node dist/main.js --help` 正常、`node --check dsh/index.js` 通过。真实识图调用还需在 `~/.modlens/config.json` 配置引擎（Gemini/火山 Ark-Doubao/Claude，`modlens doctor` 查看就绪状态）——本机未配 key，故"看图"这一步未实测。

### 注记 2 · dsh-dshx：zero-deps 假设已失效，需依赖引导

dshx 自身零 npm 依赖、依赖宿主环境"随 dsh 自带"的包。dsh 0.2.0-rc.2 的 profile 独立安装（`nodeLinker: hoisted`）下**并不自带**这些包，需一次性补齐（精确版本，均已被 dsh 供应链策略放行）：

```sh
# 通用依赖
dsh plugin --profile web add chokidar
dsh plugin --profile web add @modelcontextprotocol/sdk
dsh plugin --profile web add js-yaml@4.3.2        # 注意 4.x：dshx 用默认导出 import 语法

# dsh 运行时套件（与 dsh 0.2.0-rc.2 严格同版）
for p in dsh-tools dsh-system-prompt dsh-scope dsh-llm dsh-agent dsh-invariants \
         dsh-ptc-runtime dsh-session dsh-user-approval dsh-sandbox dsh-sandbox-policy \
         dsh-fs; do
  dsh plugin --profile web add "@deepseek-ai/$p@0.2.0-rc.2"
done
dsh plugin --profile web add @deepseek-ai/cordis@4.0.4
```

补齐后 6 个行（`dshx-mcp/memory/skills/commands/prompts/manager`）全部激活，`/dshext/` 扩展管理台可访问。registry 上的 `@deepseek-ai/dsh-tools@0.0.1-rc.1`（旧线）与 dsh 0.2.0-rc.2 不兼容，会被 `dsh plugin` 正确拦截——必须用 `@0.2.0-rc.2` 精确版本。

### 注记 3 · dsh-dshx：README 安装命令为旧仓库名

dsh-dshx README 写的是 `github:YZz-S/dshx`——该名已 404（仓库改名 `dsh-dshx`，包名仍是 `dshx`）。**正确命令**：`dsh plugin --profile web add github:YZz-S/dsh-dshx`。互斥提醒同样有效：`dshx-*` 行 id 与旧手动安装变体重复会双重挂载。

### 注记 4 · dsh-update-checker：npm 全局安装下 `current` 扫描不到

`/api/dsh-update-checker/check` 返回 `current: null`，诊断日志为「未在任何候选路径找到 @deepseek-ai/dsh/package.json」——本机 dsh 经全局 npm 安装于 `%APPDATA%\npm\node_modules\@deepseek-ai\dsh`，不在插件扫描的候选路径内；`latest` 检测（经 npmmirror）正常。

### 注记 5 · pnpm 对 github 依赖按 `files` 白名单打包

`dsh plugin add github:...` 实际经 pnpm clone → pack，包内 `package.json` 的 `files` 白名单生效：仓库里的 `src/`、`test/`、`scripts/` 等开发目录**不随装**（如 dsh-bili-summary 的 `test/sanity.mjs` 需从仓库 clone 跑）。排障时别在装好的包里找开发文件。

### 其他实测细节

- profile 位于 `~/.dsh/profiles/web`，`cordis.patch.yml` 起初为 `[]`，各 bundle 的行经各自 `cordis.patch.yml` 插入组合；行 id 分别为 `notifier` / `token-cost-meter` / `billing-balance` / `dsh-update-checker` / `workspace-files-explorer` / `dshx-*`×6 / `bili-summary` / `modlens`，互不冲突、可并存。
- billing-balance 的 DeepSeek 余额拉取依赖配置；火山侧需 AK/SK。插件路由与结构在本环境直接验证（`/api/billing-balance/status` 返回结构化 JSON）。
- token-cost-meter 实拉官方定价样例：`deepseek-v4-pro` cacheHit 0.02 / cacheMiss 0.15 / output 0.04（元/百万 tokens，源 api-docs.deepseek.com）。
- bili-summary 的 sharp 需 pnpm 批准构建脚本后才可装；跳过批准则插件自动降级「仅封面配图」。
- Web GUI 服务本体监听 `127.0.0.1:3080`（`dsh --profile web`）。

## 相关仓库

- [awesome-dsh-plugin](https://github.com/YZz-S/awesome-dsh-plugin) — dsh 插件精选列表（社区收录入口）
- [DeepSeek Harness](https://www.deepseek.com/) — 宿主本体（`@deepseek-ai/dsh`）

## 许可证

各插件仓库自持许可证（MIT，awesome-dsh-plugin 为 CC0-1.0）。本仓库文档仅作聚合说明。
