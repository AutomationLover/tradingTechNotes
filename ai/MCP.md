#udemy
#AI 
#MCP
#python

source https://www.udemy.com/course/the-complete-agentic-ai-engineering-course/learn/lecture/50767593#overview

---

# **MCP** 概念 

- MCP 的全称是 **Model Context Protocol**，关键字是 “Protocol（协议/标准）”。
- 它的目标是：提供一个 **统一、简单的方式**，让你的 Agent 能够和别人实现的 工具、资源、提示词进行集成。
- 重点是 **“****共享工具****”**：一个人写了有用的工具，其他人可以非常容易地拿来用到自己的 Agent 里。
- 因为这种“即插即用、互通性”，Anthropic 把 MCP 比喻成 **“Agentic AI** **的** **USB‑C”**：统一接口、随插即用。


**MCP** **三件套：** **Host / Client / Server**
**1. MCP Host****（宿主）** 

Host 就是你运行 Agent 的那整个应用：

- 可以是 Claude Desktop（桌面端 Claude 客户端）
- 也可以是你自己用 OpenAI Agents SDK 写的 agent 应用

简单说：**Host = “****跑** **Agent** **的主程序****”**，里面会加载 MCP client，把外部 MCP server 提供的工具挂给 Agent 用。

**2. MCP Client****（客户端）** 

MCP client 是跑在 Host 里面的“一小块插件”：

- 每个 MCP client **1:1** **连接一个** **MCP server**
- 如果在 Claude Desktop 里接了 3 个 MCP server，你的 Host 里就会有 3 个 MCP client
- Client 在 Host 进程内，负责跟外部的 MCP server 建立连接、通讯

**3. MCP Server****（服务器）** 

MCP server 是真正**提供工具** **/** **上下文** **/ prompt** **模板**的那段代码，运行在 Host 外面：

- 重点还是 tools（最火的用法）
- 也可以暴露额外 context（比如查询数据源）、prompt 模板等

示例：**fetch MCP server**

- 功能：给 Agent 提供“上网抓网页”的工具
- 实现方式：在本机起一个 headless 浏览器（比如 headless Chrome），用 Playwright 控制，抓网页内容返回
- 使用方式：你在 Claude Desktop 里配置一个 MCP client 连到本机的 fetch MCP server，这样 Claude 聊天时就能“看网页”
- 课程里之前用 Autogen 时已经用过同一个 fetch MCP server——server 跑在你机器上，Autogen 里那段代码相当于 MCP client。

**4.** **常见拓扑：本地** **vs** **远端** **server** 

讲师画了几种典型拓扑，并特别澄清一个容易误解的点：

1）**最常见：****server** **跑在本机**

- 你从公共 repo（例如 Anthropic 官方 repo、社区市场）下载 MCP server 代码
- 在你自己的机器上启动 MCP server 进程
- Host（Claude Desktop / 你写的 Agent app）里跑 MCP client，连到本机这个 server
- Server 可以：

- 只操作本地（比如读写本地文件系统），或者
- 再自己调用外部 Web API（查天气、查网页等）

→ 也就是说：**虽然** **server** **会****“****上网****”****，但它这个进程本身一般还是跑在你本机。**

2）**较少见：真正的远程** **MCP server**

- MCP server 部署在另一台机器上（托管 / managed / hosted MCP server）
- 你的 MCP client 通过网络连过去

讲师强调：这是少数情况，**大部分** **MCP server** **都是装在你本地跑的** (如下图)，只是你从远程仓库拉下来而已。很多人听到 “server” 本能以为一定是远程服务，这是他反复纠正的误区。

```
                   ┌───────────────────────────────────────────┐
                   │                 YOUR PC                   │
                   │                                           │
                   │  ┌─────────────────────────────────────┐  │
                   │  │               HOST                  │  │
                   │  │ (Claude Desktop / Agents SDK App)   │  │
                   │  │                                     │  │
                   │  │   ┌─────────────────────────────┐   │  │
                   │  │   │        MCP CLIENT          │   │  │
                   │  │   │  (plugin inside Host)      │   │  │
                   │  │   └─────────────┬──────────────┘   │  │
                   │  └─────────────────│───────────────────┘  │
                   │                    │ stdio / SSE           │
                   │                    ▼                       │
                   │  ┌─────────────────────────────────────┐  │
                   │  │             MCP SERVER              │  │
                   │  │   (runs OUTSIDE Host, on your PC)   │  │
                   │  │                                     │  │
                   │  │  - exposes TOOLS / CONTEXT / PROMPTS│  │
                   │  └─────────────┬───────────────────────┘  │
                   │                │                          │
                   └────────────────│──────────────────────────┘
                                    │
                                    │ HTTP / other network calls
                                    ▼
                         ┌──────────────────────────────┐
                         │      REMOTE WEB SERVICES     │
                         │  (weather API, websites, …)  │
                         └──────────────────────────────┘

```

“多个 server、多 client”的变体，方便你脑补后面 trading floor 的结构
```
┌───────────────────────────────────────────┐
│                   HOST                    │
│  (Agents SDK App with your Agent)         │
│                                           │
│   ┌────────────┐    ┌────────────┐        │
│   │MCP CLIENT A│    │MCP CLIENT B│  ...   │
│   └─────┬──────┘    └─────┬──────┘        │
└─────────│─────────────────│───────────────┘
          │                 │   (each 1:1)
          ▼                 ▼
   ┌───────────────┐  ┌───────────────┐
   │MCP SERVER A   │  │MCP SERVER B   │   ...
   │(filesystem)   │  │(fetch web)    │
   └───────────────┘  └───────────────┘

```

本地/Remote MCP server
1. 本地 server，只做本机事情（比如写文件的 file-writer server）。
2. 本地 server，但会自己调用远程服务（如 fetch、Playwright：本地起进程，进程里去上网）。
3. 比较少见的远程/托管 MCP server：client 直接连另一台机器上的 MCP server（这时必须用 SSE 传输）。

**5.** **两种传输机制（****transport****）** 

Anthropic MCP 目前规范了两种传输方式：

1）**stdio****（****Stdio****）** – 最常用

- MCP client 在本机 **spawn** **一个子进程** 作为 MCP server
- 通过进程的标准输入 / 输出（stdin / stdout）通信
- 这是课程中构建自定义 MCP server 时会用的方式，也是大多数本地 server 的默认模式

2）**SSE****（****Server-Sent Events****）**

- 基于 HTTPS 的长连接，像 LLM 流式返回 token 那种方式
- 如果要连**远程托管的** **MCP server**，就必须用 SSE，不能用 stdio
- 对于本地 server，两种都可以用，但现实中还是 stdio 最普遍

**6.** **小结（帮你记住的心智模型）** 

- **Host**：Agent 主应用（Claude Desktop / 你写的 Agents SDK app）
- **MCP Client**：Host 里的“小插件”，每个 client 连一个 server
- **MCP Server**：实际提供工具 / 上下文 / prompt 的进程，一般运行在你本机，常用 stdio 作为进程间通信，有时再去访问互联网 API
- **远程** **server + SSE**：存在，但比较少见，多数 MCP server 是“装在你机器上的插件进程”，而不是你远程调用的 SaaS 服务


---
# 使用MCP server

```python
# 下面代码 用 MCP stdio transport，在本机拉起一个第三方 MCP server（fetch），并查询它暴露的 tools。

fetch_params = {"command": "uvx", "args": ["mcp-server-fetch"]} # 定义 MCP server 的参数， 

# 启动 MCP server(stdio 类型) + 建立 client
async with MCPServerStdio(params=fetch_params, lient_session_timeout_seconds=60) as server:
    fetch_tools = await server.list_tools() # list_tools 向 MCP server 询问“你有什么工具”

fetch_tools
```

`uvx` is just shortcut for “grab this package, run its main entrypoint”.

```sh
uvx some-tool arg1 arg2
# or
pip install some-tool
some-tool arg1 arg2
# or
python -m some_tool arg1 arg2  # depends on actual module name

```

```
1. `async with ... as server:`

- 进入 `with` 时：
	- 启动子进程（MCP server）
	- 建好到它的连接（MCP client）

- 退出 `with` 时：
	- 关闭连接
	- 杀掉子进程，做清理

- `server` 这个变量，其实就是“ 已经连上的 MCP client 句柄”，对它调用方法，就是在跟 MCP server 说话。
```

```
- Host：OpenAI Agents SDK / 你的 Python Notebook
- MCP client：`MCPServerStdio(...)` 里创建的那段逻辑
- MCP server：通过 `uvx mcp-server-fetch` 启动的子进程
```

另外一个实例
- **一个** **Agent** **同时用浏览器** **+** **文件系统两个** **MCP server** **完成搜索菜谱任务** 

```python
instructions = """
You browse the internet to accomplish your instructions.
You are highly capable at browsing the internet independently to accomplish your task, including accepting all cookies and clicking 'not now' as appropriate to get to the content you need. If one website isn't fruitful, try another.
Be persistent until you have solved your assignment,trying different options and sites as needed.
"""
# mcp_server_files: File system MCP server：在指定 `sandbox` 目录里读写本地文件。
async with MCPServerStdio(params=files_params, client_session_timeout_seconds=60) as mcp_server_files:
	
	# Playwright 浏览器 MCP server：细粒度控制浏览器（打开页面、点击、滚动、截图等）
	async with MCPServerStdio(params=playwright_params, client_session_timeout_seconds=60) as mcp_server_browser:
	
		agent = Agent(
			name="investigator",
			instructions=instructions,
			model="gpt-4.1-mini",
			mcp_servers=[mcp_server_files, mcp_server_browser]	
			)
		# trace（方便之后在 OpenAI UI 里看调用轨迹）
		with trace("investigate"):
			# `runner.run()`任务: “找到一个 banoffee pie 的好食谱，并用 Markdown 总结。”
			result = await Runner.run(agent, "Find a great recipe for Banoffee Pie, then summarize it in markdown to banoffee.md")
			print(result.final_output)
```

运行时你会看到：
- 本机弹出浏览器窗口，自动打开（例如）BBC Good Food 之类的网站。
- Agent 通过 Playwright MCP server 自动导航、浏览网页。
- 任务完成后，它调用 File system MCP server，把总结好的 Markdown 写进 sandbox 目录的某个文件中。

检查：
- 打开 sandbox 目录里的文件，确认确实写入了 banoffee pie 配方的 Markdown。
- 在 OpenAI traces 里查看调用过程：
	- 先对两个 MCP server 做 `list_tools`。
	- 使用浏览器 navigate 类工具访问网页。
	- 使用文件工具（read / write file）把结果写到本地。

https://mcp.so/ MCP marketplace 找 MCP servers

---
# 编写 MCP server

**适合做** **MCP server** **的情况：**
- 你希望：
	- 把自己写的工具 **分享给别人** 使用，让别人的 Agent 可以直接接入。
	- 你的工具不仅是代码，还有配套的 **资源（类似** **RAG** **的** **context****）** 和 **精心设计的** **prompt** **模板**，希望通过标准协议暴露出去。

- 你在搭建一个系统，里头已经大量使用 MCP servers，希望把自己的工具也包装成 **MCP server**，以统一管理/调用方式。
- 你想借这个过程，**真正理解** **MCP** **的底层** **plumbing**（协议、stdio/SSE 通信等），而不是只做使用者。


**什么时候不要做** **MCP server**
如果你只是想给“自己当前的 Agent”增加一个工具函数，**完全没必要**做 MCP server。
- 在 OpenAI Agents SDK 里，你已经可以：
	- 写一个普通的 Python 函数。
	- 用 `@app.tool` / `@function_tool` 这样的 decorator 一装。
	- 把它直接放到 `tools=[...]` 里，LLM 就能在同一进程内直接调用。

- 这样函数调用就是“本进程内普通函数调用”，简单、高效，没有额外进程，没有 IPC。


做 MCP server意味着什么？
- 你要写一堆 protocol + plumbing：
	- 实现 MCP 协议
	- 起一个单独进程
	- 和 client 通过 stdio/SSE 通信

MCP 的真正价值在于：让工具可以被分享和复用，而不是让你本地函数变成工具。


```python
# MCP server code
from mcp.server.fastmcp import FastMCP
from accounts import Account # 业务逻辑模块

mcp = FastMCP("accounts_server")

@mcp.tool()
async def get_balance(name: str) -> float:
	"""Get the cash balance of the given account name.
	Args:
	name: The name of the account holder
	"""
	return Account.get(name).balance
```


```python
# using MCP server code
params = {"command": "uvx", "args": ["your-mcp-server-entrypoint"]}
async with MCPServerStdio(params=params) as server:
    tools = await server.list_tools()

```


