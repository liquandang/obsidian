
[MCP 协议：为什么 Streamable HTTP 是最佳选择？](# https://developer.aliyun.com/article/1661971)

``` python

MCP（Model Context Protocol）官方关于支持 **Streamable HTTP** 的公告和变更主要出现在以下几个来源：

1. **GitHub PR 合并**
    
    - 2025年3月，MCP 官方 GitHub 仓库合并了 [#206 PR](https://github.com/modelcontextprotocol/specification/pull/206)，正式引入 **Streamable HTTP** 作为新的传输方式，取代原有的 **HTTP+SSE** 方案156。
        
2. **MCP 规范更新**
    
    - 2025年3月26日，MCP 官方更新了协议规范文档，详细描述了 **Streamable HTTP** 的设计和优势，包括移除 `/sse` 端点、统一使用 `/message` 端点、支持按需升级 SSE 流等56。
        
3. **开源社区报道**
    
    - OSC开源社区、CSDN、博客园等技术媒体均报道了 MCP 的这一重大更新，并引用了官方 GitHub 的变更说明124。
        
4. **Spring AI Alibaba 实现**
    
    - 2025年4月，Spring AI Alibaba 联合 Higress 发布了业界首个 **Streamable HTTP** 实现方案，进一步验证了 MCP 官方的这一变更23。
        

### 总结

MCP 官方并未单独发布公告，但通过 **GitHub PR 合并** 和 **协议规范更新** 正式宣布了对 **Streamable HTTP** 的支持。相关技术社区和开源项目（如 Spring AI Alibaba）也基于此进行了适配和报道。如需查看官方原始说明，可参考：

- [MCP GitHub PR #206](https://github.com/modelcontextprotocol/specification/pull/206)
    
- [MCP 2025-03-26 规范更新](https://github.com/modelcontextprotocol/specification/tree/main/docs/specification/2025-03-26) 
```

## 为什么选择 Streamable HTTP?

![[Pasted image 20250426224235.png]]

### HTTP + [[SSE（server-send-events）]] 存在的问题


HTTP+SSE 的传输过程实现中，客户端和服务器通过两个主要渠道进行通信：
（1）HTTP 请求/响应：客户端通过标准的 HTTP 请求向服务器发送消息。
（2）服务器发送事件（SSE）：服务器通过专门的 /sse 端点向客户端推送消息，这就导致存在下面三个问题：

- **服务器必须维护长连接**，在高并发情况下会导致显著的资源消耗。
- **服务器消息只能通过 SSE 传递**，造成了不必要的复杂性和开销。
- **基础架构兼容性**，许多现有的网络基础架构可能无法正确处理长期的 SSE 连接。企业防火墙可能会强制终止超时连接，导致服务不可靠。

### Streamable HTTP 的改进

Streamable HTTP 是 MCP 协议的一次重要升级，通过下面的改进解决了原有 HTTP + SSE 传输方式的多个关键问题：

- **统一端点**：移除了专门建立连接的 /sse 端点，将所有通信整合到统一的端点。
- **按需流式传输**：服务器可以灵活选择返回标准 HTTP 响应或通过 SSE 流式返回。
- **状态管理**：引入 session 机制以支持状态管理和恢复。

## HTTP + SSE vs Streamable HTTP

下面通过实际应用场景中稳定性，性能和客户端复杂度三个角度对比说明 Streamable HTTP 相比 HTTP + SSE 的优势，**AI 网关 Higress 目前已经支持了 Streamable HTTP 协议**.
- 通过 MCP 官方 Python SDK 的样例 Server 部署了一个 HTTP + SSE 协议的 MCP Server
- 通过 Higress 部署了一个 Streamable HTTP 协议的 MCP Server。

## 稳定性对比

### TCP 连接数对比

利用 Python 程序模拟 1000 个用户同时并发访问远程的 MCP Server 并调用获取工具列表，图中可以看出 SSE Server 的 SSE 连接无法复用且需要长期维护，高并发的需求也会带来 TCP 连接数的突增，而 Streamable HTTP 协议则可以直接返回响应，多个请求可以复用同一个 TCP 连接，TCP 连接数最高只到几十条，并且整体执行时间也只有 SSE Server 的四分之一。

![image](https://ucc.alicdn.com/pic/developer-ecology/pawmkwdq37c7s_544ca42f2dba464b85ca42258562d558.png "image")

在 1000 个并发用户的测试场景下，Higress 部署的 Streamable HTTP 方案的 TCP 连接数明显低于 HTTP + SSE 方案：

- HTTP + SSE：需要维持大量长连接，TCP连接数随时间持续增长
- Streamable HTTP：按需建立连接，TCP连接数维持在较低水平

### 请求成功率对比

实际应用场景中进程级别通常会限制最大连接数，linux 默认通常是 1024。利用 Python 程序模拟不同数量的用户访问远程的 MCP Server 并调用获取工具列表，SSE Server 在并发请求数到达最大连接数限制后，成功率会极速下降，大量的并发请求无法建立新的 SSE 连接而访问失败。

![image](https://ucc.alicdn.com/pic/developer-ecology/pawmkwdq37c7s_8e02d1ca9ad04b238563a1e1db6369bd.png "image")

在不同并发用户数下的请求成功率测试中，Higress 部署的 Streamable HTTP 的成功率显著高于 HTTP + SSE 方案：

- HTTP + SSE：随着并发用户数增加，成功率显著下降
- Streamable HTTP：即使在高并发场景下仍能保持较高的请求成功率

## 性能对比

_这里对比的是社区 Python 版本的_ _GitHUB MCP Server__【2】_ _和 Higress MCP 市场的_ _GitHUB MCP Server_

利用 Python 程序模拟不同数量的用户同时并发访问远程的 MCP Server 并调用获取工具列表，并统计调用返回响应的时间，图中给出的响应时间对比为对数刻度，SSE Server 在并发用户数量较多时平均响应时间会从 0.0018s 显著增加到 1.5112s，而 Higress 部署的 Streamable HTTP Server 则依然维持在 0.0075s 的响应时间，也得益于 Higress 生产级的性能相比于 Python Starlette 框架。

![image](https://ucc.alicdn.com/pic/developer-ecology/pawmkwdq37c7s_3a8c3ff20cc54fb4b38e2204c7bd5d9e.png "image")

性能测试结果显示，Higress 部署的 Streamable HTTP 在响应时间方面具有明显优势：

- Streamable HTTP 的平均响应时间更短，响应时间波动较小，随并发用户数增加，响应时间增长更平
- HTTP + SSE 的平均响应时间更长，在高并发场景下响应时间波动较大

## 客户端复杂度对比


Streamable HTTP 支持无状态的服务和有状态的服务，目前的大部分场景无状态的 Streamable HTTP 的可以解决，通过对比两种传输方案的客户端实现代码，可以直观地看到无状态的 Streamable HTTP 的客户端实现简洁性。 

#### **HTTP + SSE 客户端样例代码**

```python
class SSEClient:
    def __init__(self, url: str, headers: dict = None):        
	    self.url = url        
	    self.headers = headers or {}        
	    self.event_source = None        
	    self.endpoint = None
        async def connect(self):            
        # 1. 建立 SSE 连接            
		    async with aiohttp.ClientSession(headers=self.headers) as session:               self.event_source = await session.get(self.url)
        # 2. 处理连接事件                
	        print('SSE connection established')
        # 3. 处理消息事件                
        async for line in self.event_source.content:                    
	        if line:                        
		        message = json.loads(line)                        
		        await self.handle_message(message)
        # 4. 处理错误和重连                        
        if self.event_source.status != 200:
		    print(f'SSE error: {self.event_source.status}')                              await self.reconnect()
		    
        async def send(self, message: dict):            
        # 需要额外的 POST 请求发送消息            
        async with aiohttp.ClientSession(headers=self.headers) as session:               async with session.post(self.endpoint, json=message) as response:                return await response.json()
        async def handle_message(self, message: dict):                    
        # 处理接收到的消息                    
        print(f'Received message: {message}')
	    async def reconnect(self):        
	    # 实现重连逻辑        
	    print('Attempting to reconnect...')        
	    await self.connect()
```

#### **Streamable HTTP 客户端样例代码**

```python
class StreamableHTTPClient:
    def __init__(self, url: str, headers: dict = None):        self.url = url        self.headers = headers or {}        
    async def send(self, message: dict):        
    # 1. 发送 POST 请求        
		async with aiohttp.ClientSession(headers=self.headers) as session:               async with session.post( self.url, json=message, headers={'Content-Type': 'application/json'}) as response:                
    # 2. 处理响应                
	    if response.status == 200:                    
		    return await response.json()                
		else:                    
			raise Exception(f'HTTP error: {response.status}')
```

  

从代码对比可以看出：

1. 复杂度：Streamable HTTP 无需处理连接维护、重连等复杂逻辑

2. 可维护性：Streamable HTTP 代码结构更清晰，更易于维护和调试

3. 错误处理：Streamable HTTP 的错误处理更直接，无需考虑连接状态

  

【1】PR #206[https://github.com/modelcontextprotocol/modelcontextprotocol/pull/206](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/206?spm=a2c6h.13046898.publish-article.15.23166ffaSTSAgH)

  

【2】GitHUB MCP Server[https://github.com/modelcontextprotocol/servers/tree/main/src/github](https://github.com/modelcontextprotocol/servers/tree/main/src/github?spm=a2c6h.13046898.publish-article.16.23166ffaSTSAgH)