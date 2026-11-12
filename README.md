# GitHub 发布器（Operit2 版）

Operit1 的 `com.operit.github_publisher`（v0.2.0，ToolPkg API 1.0.x）迁移到 Operit2 的
**ToolPkg API 2.0.0** 平行实现。功能面对齐 v1：列出已安装 Skill / MCP、校验 GitHub 账号、
一键发布到 GitHub（自动建仓、并发推文件、可选 Release + 资产），并保留侧边栏控制台。

- `toolpkg_id`: `com.operit.github_publisher`（与 v1 相同，便于沿用 `github_publisher:xxx` 工具名）
- 子包 id: `github_publisher`
- 版本: `2.0.2`
- 语言/构建: 手写 CommonJS，无打包步骤（`manifest.json` 在包根）

## 为什么必须改数据源

v1 在 Operit2 上表现为「0 个 Skill、0 个 MCP」+「未检测到 GitHub 登录」，原因不是配置问题：

| 环节 | v1 的做法 | Operit2 的实际行为 |
|---|---|---|
| Skill 列表 | 硬编码 `/sdcard/Download/Operit/skills` | 插件 `Tools.Files` 只暴露 `/operitapp`、`/virtual/android/{sdcard,root,documents}` 这几种根；`/sdcard/...` 被解析到插件私有目录，`list` 返回空数组，于是永远是 0 |
| MCP 列表 | 读 `.../mcp_plugins/mcp_config.json` | 该文件不存在；MCP 配置只存在 `data/database/operit2.sqlite` |
| GitHub 账号 | 断言「自动获取当前 APP 的 GitHub 登录」 | API 2.0.0 的 `types/*.d.ts` 里没有任何 GitHub 相关入口，App 内登录态不对外开放 |

## 2.0 版的三条实现主线

1. **Skill = 纯文件扫描**
   平台技能根目录：`/operitapp/data/extensions/device/skills`（真机绝对路径
   `/data/user/0/app.operit/files/runtime/identities/<id>/extensions/device/skills`）。
   逐个目录读 `SKILL.md` 的 front-matter 拿 name/description，再统计文件数。
   发布时用同一批路径 `walkFiles` + `readBinary` 取字节。

2. **MCP = `mcp list` / `mcp config`（只读，毫秒级）**
   这两条命令只读本地库、不发起网络握手，实测即时返回；发布时据此还原 `mcp.config.json`
   （默认对 token/key/secret/header 等值打码，`include_secrets` 可关闭打码）。
   **禁止使用 `mcp meta <id>`**：它会拉起远程 MCP 握手，是本次踩坑的直接触发点。

3. **Token = 插件自有，三级来源**
   入参 `token` > 插件内保存（`PluginConfig` / 插件配置目录 `github_token.json`）> 包环境变量 `GITHUB_TOKEN`。
   新增 `github_set_token`（先校验再落盘）与 `github_token_status`；控制台里就是「粘贴 → 校验并保存」。

## 一条踩过的坑（重要）

**不要从插件里调用 `Tools.SoftwareSettings.exec`（core command）。**
它在 QuickJS 里是阻塞实现：底层命令一旦不返回，JS 线程和 `setTimeout` 一起停住，
`Promise.race` 超时兜底根本不会触发，而且 core command 是全局串行的，会把
`operit_editor` / `package` / `mcp` 等所有 CLI 一起堵死，直到 App 重启。

实测触发点：`mcp meta <id>` 对远程 MCP 会挂住（我第一版用它补全 MCP 元数据）。
本版本已经**完全不用 core command**：Skill 走文件、MCP 走库、Token 走配置。
`shared/local.js` 里保留了 `cliExec/cliJson` 仅供人工排查，任何默认路径都不调用。

## 工具

| 工具 | 作用 |
|---|---|
| `list_installables` | 列出 Skill + MCP（含文件数 / 传输方式 / 启用状态） |
| `github_check_auth` | 校验当前可用 Token 并返回账号与来源 |
| `github_set_token` | 校验并保存 Token（masked 回显） |
| `github_token_status` | 是否已配置、来源、打码值、落盘路径 |
| `github_publish` | 发布（`type` + `source_id` 必填，可选 `repo` / `release_tag` / `asset_path` / `include_secrets` …） |
| `main` | 自检：Token、账号、可发布数量 |

## 安装

```bash
# 1. 打包（manifest.json 必须在归档根）
#    本机没有 zip 命令时可用插件侧 Tools.Files.zip，或任意 zip 工具
# 2. 同 ID 不覆盖：先删再导入
operit2 package delete github_publisher
operit2 package import /storage/emulated/0/Documents/operit2/gp2.toolpkg
operit2 package enable github_publisher
# 3. 逐工具验证（以真实返回为准，不看 package show 的 Tools 计数）
operit2 package exec github_publisher:main '{}'
operit2 package exec github_publisher:list_installables '{}'
```

> 注意：宿主会缓存已加载包的编译产物。替换压缩包后需要重新加载（App 重启或先 delete 再 import），
> 否则工具面仍是旧代码。

## 侧边栏控制台

`main.js` 注册 `UiRoute`（`toolpkg:com.operit.github_publisher:ui:github_publisher_console`）+
`main_sidebar_plugins` / `toolbox` 两个导航入口。UI 全部通过 `ctx.callTool("github_publisher:xxx")`
调本包工具，因此不依赖 ui 上下文里是否能用 `Tools`。

## 平台支持

- **已验证**：Android（Operit2 preview.31），完整链路（列出 → 发布 → Release 资产 → sha256 回读校验）。
- **未验证**：iOS / macOS / Windows / Linux。
- 设计上不依赖平台专属能力：只用 `Tools.Files` / `Tools.Net` / `Tools.SoftwareSettings`（只读命令）与 Compose DSL，
  没有 Java 桥、没有 `android.*`、没有写死 `/sdcard` 或盘符路径。
- 已知依赖：`skill list` / `mcp list` / `mcp config` 三条 core command。若某端未提供，`list_installables`
  会返回 `mcp list 未完成: …` 之类的警告，而不是静默返回空。

## 插件包发布（publish_plugin）

把本机 `.toolpkg` 发到 GitHub，并产出 Operit 市场可直接引用的 Release 资产：

```json
{ "path": "<包路径>", "repo": "仓库名", "release_tag": "v2.0.2", "dry_run": true }
```

流程：读包 → 算 sha256（宿主无 SHA256，自带纯 JS 实现，已与系统 `sha256sum` 对齐）→
推包 + README 到仓库 → 建 Release → 上传资产 → **回读资产重算 sha256 校验** →
返回 `marketHint`（拼好参数的 `market publish artifact` 命令）。

- `dry_run: true` 只算 sha256 / 读 manifest，不联网。
- `delete_release { repo, tag }` 用于清理测试版本或重发同 tag。

## 控制台（ui11）

- Compose 外壳（Token、发布设置、日志）+ 一个**可折叠**的 HTML 资源面板（`ui/list.html`）。
- 折叠是刻意的：WebView 会吞掉纵向手势，整屏 HTML 会导致下面板块滑不到。
- 面板通过 `GpHost.bootstrap/pick` 双向通信；主题按 `brightness` 推导整套 CSS token，深色下不糊。

## 已知边界

- 私有仓库、Release 资产走 GitHub REST；资产用 `Tools.Net.uploadFile`（multipart），
  上传后建议在 GitHub 上核对字节一致再拿去做市场引用。
- MCP 读取依赖 `mcp list` / `mcp config`；若某版本移除此命令，`list_installables` 会返回
  `mcp list 未完成: …` 警告而不是静默返回空。
- 单文件 40MB / 总量 300MB 上限，超出直接失败并回报，不会半个仓库。

## Token 存放与「认证失败」排查

**怎么拿 Token**：GitHub 右上头像 → Settings → 左侧最底部 Developer settings →
Personal access tokens → Generate new token。

- Classic：https://github.com/settings/tokens （勾 `repo`）
- Fine-grained：https://github.com/settings/tokens?type=beta
  （`Contents: Read and write`；要自动建仓再加 `Administration: Read and write`）
- 明文**只显示一次**，生成后立刻复制

插件内点「怎么获取令牌？」可展开同样的步骤。

- 主存：`plugins/configs/com.operit.github_publisher/github_token.json`；
  镜像：`plugins/configs/github_publisher.token.json`（**删除包会把主存目录一起清掉**，
  镜像放在根目录，重装后可自动恢复）。
- 读取优先级：入参 `token` > 主存 > 镜像 > 环境变量 `GITHUB_TOKEN`。

按 GitHub 官方文档把状态码映射成可操作提示（`shared/github.js#ghError`）：

| 状态 | 含义 | 处理 |
|---|---|---|
| 401 | 凭据无效/过期/被撤销 | 重新生成 PAT，`github_set_token` 覆盖 |
| 403 + `rate limit` | 限流；**短时间多次用错 Token 也会让所有认证被临时拒绝** | 等 `X-RateLimit-Reset`，或停几分钟 |
| 403 + `X-GitHub-SSO` | 未完成 SAML SSO 授权 | 按响应头链接授权 |
| 403 / 404（仓库相关） | 权限不足 | classic PAT 勾 `repo`；fine-grained PAT 给目标仓库 `Contents: Read and write`，自动建仓还需 `Administration: Read and write` |
| 422 | 参数被拒（仓库名非法/同名冲突） | 换仓库名 |

> 自测时不要反复用假 Token 试探，会触发上面那条「无效凭据锁」，看起来像「认证一直失败」。

## 验收记录（本机实测）

| 项 | 结果 |
|---|---|
| `package delete → import → enable` | 三步均一次成功，`Stored: …/packages/gp2.toolpkg` |
| `list_installables` | `共 4 个 Skill、1 个 MCP`，warnings 为空；skills 含 `PackageBuilder(701 文件)`、`operit2_toolpkg_dev(10)`；mcps 为 `fetch`(streamable_http) |
| `main` | `success:true`，`apiVersion:"2.0.0"`，skills=4，mcps=1 |
| `github_token_status` | `success:true`，`configured:false`，落盘路径可见 |
| `github_check_auth`（故意无效 token） | `GitHub 认证失败 (401): Bad credentials` —— REST 链路通，错误透传正确 |
| 卡顿 | 无：单步最长 <1s，未再出现「工具结果缺失」 |

过程中修掉的两个真实缺陷：
1. `mcp meta` 挂死 core command 队列（已彻底移除该调用）。
2. `Tools.Net.http` 传 `body: undefined` 会被宿主判为 `non-JSON value`，导致所有 GET 误报
   「网络请求失败」；已改成「有 body 才写字段」。

下一步（需要 Token）：`github_set_token` 保存 Personal Access Token（repo 权限）→
`github_publish {"type":"skill","source_id":"PackageBuilder"}` 即可发布。