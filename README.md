Windows 实战：用 CodexPro + ngrok 把 ChatGPT 接入本地项目
适用场景：你希望在 ChatGPT 网页版或桌面版中，通过 MCP 读取、搜索、修改并检查本地项目；同时希望使用 ngrok 的稳定 HTTPS 域名，避免每次重启都更换连接地址。

先澄清：这套方案到底做什么
CodexPro 不是 Codex CLI 的代理，也不会把 ChatGPT 模型“塞进”Codex CLI，更不是绕过配额的工具。它是在本机启动一个受控的 MCP 服务，让 ChatGPT 通过开发者模式调用本地项目工具。

完整链路如下：

TEXT
复制
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
OpenAI 官方文档要求远程 MCP 服务可通过公网 HTTPS 访问，并通常在 /mcp 提供 Streamable HTTP；开发者模式是否可用取决于账号和工作区策略，不应简单理解为“只要购买某个套餐就一定有”。参见 OpenAI：连接并测试 MCP 服务。

一、准备条件
建议准备：

Windows 10 或 Windows 11。
Node.js 20 或更高版本。
可创建自定义 Plugin/MCP 连接的 ChatGPT 账号。
一个免费或付费 ngrok 账号。
一个用于测试的 Git 项目。第一次不要直接连接包含生产密钥或客户数据的仓库。
在 PowerShell 中检查 Node.js 和 npm：

POWERSHELL
复制
node --version
npm --version
node --version 应为 v20 或更高。

二、安装并检查 CodexPro
全局安装当前发布版：

POWERSHELL
复制
npm install -g codexpro@latest
检查安装：

POWERSHELL
复制
codexpro --version
codexpro --help
如果 PowerShell 提示找不到 codexpro，请先关闭并重新打开终端；仍无效时检查 npm 全局可执行目录是否已加入 PATH。

CodexPro 的当前安装要求和命令以 CodexPro 官方仓库 为准。仓库明确说明它是本地 MCP 服务，不是模型代理、托管 SaaS 或配额绕过器。

三、安装和登录 ngrok
3.1 Windows 安装
官方 Windows 页面推荐 Microsoft Store，也提供 WinGet 和 Scoop。使用 WinGet：

POWERSHELL
复制
winget install ngrok -s msstore
也可以从 ngrok Windows 下载页 安装。安装后检查：

POWERSHELL
复制
ngrok version
3.2 写入 ngrok Authtoken
登录 ngrok Dashboard，在账号设置或入门页复制 Authtoken，然后仅在自己的终端执行：

POWERSHELL
复制
ngrok config add-authtoken "<YOUR_NGROK_AUTHTOKEN>"
不要把真实 Authtoken 写进教程、截图、聊天记录或 Git 仓库。上面的命令可能进入 PowerShell 历史记录；如设备由多人共用，应在完成后按你的终端安全策略清理或保护历史记录。

3.3 获取稳定 Dev Domain
打开 ngrok Domains，找到系统分配的 Dev Domain，例如：

TEXT
复制
example-abc-123.ngrok-free.dev
填写 CodexPro 配置时只填主机名，不要附加 /mcp，也不要把 https:// 重复写入 hostname 参数。

ngrok 官方说明：每个账号都有一个自动分配的免费 Dev Domain；它是稳定主机名，但免费计划不能自行选择名称。自定义域名属于付费能力。参见 ngrok Domains 文档。

四、在项目中配置 CodexPro + ngrok
以下示例项目位于 D:\Code\demo-project，请替换为你的实际目录：

POWERSHELL
复制
Set-Location "D:\Code\demo-project"
codexpro setup
交互提示的文字和顺序会随版本变化，但核心选择建议如下：

配置项	建议值	说明
Workspace root	当前项目目录	不要直接授权整个磁盘或用户主目录
Local port	8787	默认值；端口冲突时再修改
Mode	agent	允许读写和受限检查；只想规划可选 handoff
Tunnel provider	ngrok	使用稳定 Dev Domain
Hostname	你的 *.ngrok-free.dev	不含路径和查询参数
Bash mode	safe	初次使用不要选 full
CodexPro token	让程序生成	应保持私密且不少于 24 字节
Save profile	yes	以后可直接 codexpro start
Start now	yes	立即启动本地服务和 ngrok
成功后，CodexPro 会启动本地 MCP 服务、启动 ngrok、等待健康检查通过，并生成或复制类似下面的 Server URL：

TEXT
复制
https://example-abc-123.ngrok-free.dev/mcp?codexpro_token=<PRIVATE_TOKEN>
这条完整 URL 就是下一步要填入 ChatGPT 的地址。codexpro_token 相当于访问本地项目的凭证，不能公开。

如果当前版本没有 ngrok 选项，先更新后重试：

POWERSHELL
复制
npm install -g codexpro@latest
codexpro --version
不走向导时的等价命令
已经完成 ngrok 登录且有安全保存的 CodexPro token 时，可以显式启动：

POWERSHELL
复制
codexpro ngrok `
  --root "D:\Code\demo-project" `
  --hostname "example-abc-123.ngrok-free.dev" `
  --bash safe
若命令要求 token，优先使用 codexpro setup 创建并保存配置，不要为了省事使用短密码。CodexPro 的稳定域名指南给出的完整 ngrok 工作方式见 DOMAIN_SETUP.md。

五、在 ChatGPT 中创建连接
OpenAI 当前官方界面的入口通常是：

TEXT
复制
Settings → Security and login → Developer mode
开启 Developer mode 后，进入 Plugins 页面，点击加号创建连接。部分旧版或不同语言界面可能显示为：

TEXT
复制
Settings → Apps → Advanced settings → Developer mode / Create app
界面名称可能变化，以当前 ChatGPT 页面和 OpenAI 官方连接说明 为准。

填写：

字段	值
Name	CodexPro
Description	Local workspace bridge for coding
Connection	Server URL
Server URL	CodexPro 复制的完整 https://.../mcp?codexpro_token=...
Authentication	No Authentication / None
