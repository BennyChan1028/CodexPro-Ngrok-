# Windows 实战：用 CodexPro + ngrok 把 ChatGPT 接入本地项目

> 适用场景：你希望在 ChatGPT 网页版或桌面版中，通过 MCP 读取、搜索、修改并检查本地项目；同时希望使用 ngrok 的稳定 HTTPS 域名，避免每次重启都更换连接地址。

## 先澄清：这套方案到底做什么

CodexPro 不是 Codex CLI 的代理，也不会把 ChatGPT 模型“塞进”Codex CLI，更不是绕过配额的工具。它是在本机启动一个受控的 MCP 服务，让 ChatGPT 通过开发者模式调用本地项目工具。

完整链路如下：

```text
ChatGPT 对话
    │
    ▼
ChatGPT 自定义 Plugin / MCP 连接
    │  公网 HTTPS
    ▼
ngrok 稳定 Dev Domain
    │  转发至 http://127.0.0.1:8787
    ▼
CodexPro MCP Server
    │
    ▼
你明确授权的本地项目目录
```

OpenAI 官方文档要求远程 MCP 服务可通过公网 HTTPS 访问，并通常在 `/mcp` 提供 Streamable HTTP；开发者模式是否可用取决于账号和工作区策略，不应简单理解为“只要购买某个套餐就一定有”。参见 [OpenAI：连接并测试 MCP 服务](https://developers.openai.com/plugins/deploy/connect-chatgpt)。

## 一、准备条件

建议准备：

- Windows 10 或 Windows 11。
- Node.js 20 或更高版本。
- 可创建自定义 Plugin/MCP 连接的 ChatGPT 账号。
- 一个免费或付费 ngrok 账号。
- 一个用于测试的 Git 项目。第一次不要直接连接包含生产密钥或客户数据的仓库。

在 PowerShell 中检查 Node.js 和 npm：

```powershell
node --version
npm --version
```

`node --version` 应为 `v20` 或更高。

## 二、安装并检查 CodexPro

全局安装当前发布版：

```powershell
npm install -g codexpro@latest
```

检查安装：

```powershell
codexpro --version
codexpro --help
```

如果 PowerShell 提示找不到 `codexpro`，请先关闭并重新打开终端；仍无效时检查 npm 全局可执行目录是否已加入 `PATH`。

CodexPro 的当前安装要求和命令以 [CodexPro 官方仓库](https://github.com/rebel0789/codexpro) 为准。仓库明确说明它是本地 MCP 服务，不是模型代理、托管 SaaS 或配额绕过器。

## 三、安装和登录 ngrok

### 3.1 Windows 安装

官方 Windows 页面推荐 Microsoft Store，也提供 WinGet 和 Scoop。使用 WinGet：

```powershell
winget install ngrok -s msstore
```

也可以从 [ngrok Windows 下载页](https://ngrok.com/download/windows) 安装。安装后检查：

```powershell
ngrok version
```

### 3.2 写入 ngrok Authtoken

登录 ngrok Dashboard，在账号设置或入门页复制 Authtoken，然后仅在自己的终端执行：

```powershell
ngrok config add-authtoken "<YOUR_NGROK_AUTHTOKEN>"
```

不要把真实 Authtoken 写进教程、截图、聊天记录或 Git 仓库。上面的命令可能进入 PowerShell 历史记录；如设备由多人共用，应在完成后按你的终端安全策略清理或保护历史记录。

### 3.3 获取稳定 Dev Domain

打开 [ngrok Domains](https://dashboard.ngrok.com/domains)，找到系统分配的 Dev Domain，例如：

```text
example-abc-123.ngrok-free.dev
```

填写 CodexPro 配置时只填主机名，不要附加 `/mcp`，也不要把 `https://` 重复写入 hostname 参数。

ngrok 官方说明：每个账号都有一个自动分配的免费 Dev Domain；它是稳定主机名，但免费计划不能自行选择名称。自定义域名属于付费能力。参见 [ngrok Domains 文档](https://ngrok.com/docs/gateway/domains)。

## 四、在项目中配置 CodexPro + ngrok

以下示例项目位于 `D:\Code\demo-project`，请替换为你的实际目录：

```powershell
Set-Location "D:\Code\demo-project"
codexpro setup
```

交互提示的文字和顺序会随版本变化，但核心选择建议如下：

| 配置项 | 建议值 | 说明 |
|---|---|---|
| Workspace root | 当前项目目录 | 不要直接授权整个磁盘或用户主目录 |
| Local port | `8787` | 默认值；端口冲突时再修改 |
| Mode | `agent` | 允许读写和受限检查；只想规划可选 `handoff` |
| Tunnel provider | `ngrok` | 使用稳定 Dev Domain |
| Hostname | 你的 `*.ngrok-free.dev` | 不含路径和查询参数 |
| Bash mode | `safe` | 初次使用不要选 full |
| CodexPro token | 让程序生成 | 应保持私密且不少于 24 字节 |
| Save profile | `yes` | 以后可直接 `codexpro start` |
| Start now | `yes` | 立即启动本地服务和 ngrok |

成功后，CodexPro 会启动本地 MCP 服务、启动 ngrok、等待健康检查通过，并生成或复制类似下面的 Server URL：

```text
https://example-abc-123.ngrok-free.dev/mcp?codexpro_token=<PRIVATE_TOKEN>
```

这条完整 URL 就是下一步要填入 ChatGPT 的地址。`codexpro_token` 相当于访问本地项目的凭证，不能公开。

如果当前版本没有 ngrok 选项，先更新后重试：

```powershell
npm install -g codexpro@latest
codexpro --version
```

### 不走向导时的等价命令

已经完成 ngrok 登录且有安全保存的 CodexPro token 时，可以显式启动：

```powershell
codexpro ngrok `
  --root "D:\Code\demo-project" `
  --hostname "example-abc-123.ngrok-free.dev" `
  --bash safe
```

若命令要求 token，优先使用 `codexpro setup` 创建并保存配置，不要为了省事使用短密码。CodexPro 的稳定域名指南给出的完整 ngrok 工作方式见 [DOMAIN_SETUP.md](https://github.com/rebel0789/codexpro/blob/main/DOMAIN_SETUP.md)。

## 五、在 ChatGPT 中创建连接

OpenAI 当前官方界面的入口通常是：

```text
Settings → Security and login → Developer mode
```

开启 Developer mode 后，进入 Plugins 页面，点击加号创建连接。部分旧版或不同语言界面可能显示为：

```text
Settings → Apps → Advanced settings → Developer mode / Create app
```

界面名称可能变化，以当前 ChatGPT 页面和 [OpenAI 官方连接说明](https://developers.openai.com/plugins/deploy/connect-chatgpt) 为准。

填写：

| 字段 | 值 |
|---|---|
| Name | `CodexPro` |
| Description | `Local workspace bridge for coding` |
| Connection | `Server URL` |
| Server URL | CodexPro 复制的完整 `https://.../mcp?codexpro_token=...` |
| Authentication | `No Authentication` / `None` |

这里选择 `No Authentication / None` 并不等于服务没有保护：个人兼容模式的凭证已经在 URL 的 `codexpro_token` 参数中。不要再选 OAuth，否则会走错误的认证流程。保持开发者模式的 CSP enforcement 开启。

创建完成后，检查 ChatGPT 是否发现 CodexPro 的工具和描述。随后新建一个对话，并从工具或 Plugins 菜单启用刚创建的连接。

## 六、分层测试

不要仅凭“页面能打开”判断成功。推荐按以下顺序测试。

### 6.1 本机自检

保持项目目录不变，另开一个 PowerShell：

```powershell
Set-Location "D:\Code\demo-project"
codexpro doctor
codexpro connection-test
```

`doctor` 用于检查本地环境和配置，`connection-test` 用于观察 ChatGPT 的连接请求是否真正到达本机。若创建 Plugin 时失败，CodexPro 官方 README 也建议优先运行 `connection-test`。

### 6.2 MCP 协议检查

需要进一步排查传输或工具定义时，可用 OpenAI 官方文档建议的 MCP Inspector：

```powershell
npx @modelcontextprotocol/inspector@latest
```

在 Inspector 中使用完整的公网 HTTPS MCP URL，检查能否发现工具以及调用返回是否符合预期。

直接在普通浏览器中打开 MCP URL，有时会返回 `405 Method Not Allowed`。这通常只能说明域名和服务可能可达，因为浏览器发的是普通 GET，而 MCP 使用的不是普通网页 GET；它不能替代 `connection-test` 或 MCP Inspector 的协议级验证。

### 6.3 ChatGPT 只读测试

在启用了 CodexPro 的新对话中输入：

```text
请使用 CodexPro 工具查看当前工作区的一级文件结构，不要修改任何文件。
```

再输入：

```text
请使用 CodexPro 读取 README.md，概括项目用途，并明确列出你实际读取的文件路径。不要猜测不存在的内容。
```

只有返回真实目录和真实文件内容，才说明读取链路成功。

### 6.4 低风险写入测试

先确认测试仓库没有未保存的重要改动，然后输入：

```text
请使用 CodexPro 在 docs 目录创建 codexpro-smoke-test.md，内容仅为：CodexPro write test。完成后展示变更，不要修改其他文件。
```

在本地 PowerShell 验证：

```powershell
git status --short
Get-Content ".\docs\codexpro-smoke-test.md"
git diff -- ".\docs\codexpro-smoke-test.md"
```

如果内容正确，写入链路已经打通。测试完成后可删除测试文件：

```powershell
Remove-Item -LiteralPath ".\docs\codexpro-smoke-test.md"
```

## 七、日常使用

第一次 `setup` 已保存项目配置后，日常只需：

```powershell
Set-Location "D:\Code\demo-project"
codexpro start
```

保持这个终端运行，ChatGPT 才能继续访问本地 MCP 服务。ngrok Dev Domain 是稳定的，但完整 Server URL 还包含 CodexPro token：如果重新生成了 token，必须同步更新 ChatGPT 里的连接地址。

停止 CodexPro 后，本地服务和由它管理的 ngrok 进程会结束，ChatGPT 连接随之离线。

## 八、常见问题排查

### 1. ChatGPT 中没有 Developer mode 或创建 Plugin 的入口

这是账号或工作区能力问题，不一定是配置错误。OpenAI 官方文档明确说明，Developer mode 的可用性可能受账号和工作区策略影响。企业或学校工作区还可能被管理员禁用。

### 2. `codexpro` 命令不存在

重新打开终端并检查：

```powershell
npm prefix -g
npm install -g codexpro@latest
codexpro --version
```

### 3. ngrok 提示未认证

重新执行：

```powershell
ngrok config add-authtoken "<YOUR_NGROK_AUTHTOKEN>"
```

然后确认 `ngrok version` 正常，并检查 Authtoken 是否属于当前 Dashboard 账号。

### 4. 域名能打开，但 ChatGPT 连接失败

依次检查：

- CodexPro 终端是否仍在运行。
- URL 是否为 `https://`。
- URL 是否包含 `/mcp`。
- URL 中的 `codexpro_token` 是否完整且与当前配置一致。
- ChatGPT Authentication 是否选择 `None / No Authentication`。
- ngrok hostname 是否与 Dashboard 中分配的 Dev Domain 完全一致。
- 是否有另一份 ngrok 进程占用了同一 Dev Domain。
- 运行 `codexpro connection-test`，观察请求是否到达。

### 5. 出现 502 或 upstream connection error

通常表示 ngrok 已在线，但本地 `127.0.0.1:8787` 没有可用服务。检查端口冲突、CodexPro 是否启动，以及 setup 中保存的端口是否一致。

### 6. 出现 401/403

优先检查 CodexPro token。稳定 hostname 不代表 token 也自动稳定；重新 setup 或重新生成 token 后，ChatGPT 中保存的 URL 也要更新。

### 7. 能读取但不能写入

检查：

- 是否使用 `agent` / workspace write 模式，而不是 `handoff` 或 `pro`。
- 当前 Windows 用户是否有项目目录写权限。
- 目标路径是否命中 CodexPro 的保护规则。
- 文件是否被其他程序锁定。
- 请求是否明确指定了工作区内的具体路径。

### 8. ChatGPT 只给文字回答，没有调用工具

新建对话，明确启用 CodexPro，并在提示中写“请使用 CodexPro 工具”。某些模型或产品界面即使能创建连接，也未必支持直接调用自定义 MCP 工具；这是模型/界面能力边界，不应误判为 ngrok 故障。

## 九、安全建议

CodexPro 官方安全策略强调：它是能接触源码树的本地开发工具，不是操作系统沙箱。至少遵守以下规则：

- 不公开或提交包含 `codexpro_token` 的完整 Server URL。
- 不在截图、Issue、聊天记录或日志中暴露 ngrok Authtoken。
- 公网隧道绝不要使用 `--no-auth`。
- CodexPro token 使用程序生成的随机值，长度至少 24 字节。
- 只授权具体项目目录，不要授权整个磁盘、用户目录或包含多个敏感仓库的父目录。
- 初次使用保持 `--bash safe`；不可信仓库建议直接使用 `--no-bash`。
- `safe` 模式仍可能运行仓库中的 package scripts，因此不要对恶意或来源不明的项目盲目信任。
- 重要仓库先提交或备份，再允许写入；每次任务结束检查 `git status` 和 `git diff`。
- Token 一旦泄露，应立即轮换，并更新 ChatGPT 中保存的 Server URL。

更完整的威胁模型和默认保护措施见 [CodexPro SECURITY.md](https://github.com/rebel0789/codexpro/blob/main/SECURITY.md)。

## 十、最短成功路径

已经安装 Node.js 后，核心步骤可压缩为：

```powershell
npm install -g codexpro@latest
winget install ngrok -s msstore
ngrok config add-authtoken "<YOUR_NGROK_AUTHTOKEN>"

Set-Location "D:\Code\demo-project"
codexpro setup
```

在 setup 中选择 `agent`、`ngrok`、你的稳定 Dev Domain、`bash safe`、保存配置并启动；然后把 CodexPro 复制的完整 `/mcp?codexpro_token=...` URL 填入 ChatGPT Developer mode 的 Plugin 连接，Authentication 选择 `None`。最后依次完成 `doctor`、`connection-test`、只读测试和低风险写入测试。

## 参考资料

- [OpenAI 官方：连接并测试 MCP 服务](https://developers.openai.com/plugins/deploy/connect-chatgpt)
- [CodexPro GitHub 仓库](https://github.com/rebel0789/codexpro)
- [CodexPro 稳定域名与 ngrok 配置](https://github.com/rebel0789/codexpro/blob/main/DOMAIN_SETUP.md)
- [CodexPro 安全策略](https://github.com/rebel0789/codexpro/blob/main/SECURITY.md)
- [ngrok Windows 安装](https://ngrok.com/download/windows)
- [ngrok Domains 官方文档](https://ngrok.com/docs/gateway/domains)
- [参考文章：CodexPro 基础配置](https://imsuk.cn/archives/312/)
- [参考文章：CodexPro + MCP 测试流程](https://blog.csdn.net/u014451778/article/details/162704246)

