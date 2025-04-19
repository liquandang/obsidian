# A2A (Agent-to-Agent) 协议详解

## 目录

- [A2A (Agent-to-Agent) 协议详解](#a2a-agent-to-agent-协议详解)
  - [1. 诞生背景](#1-诞生背景)
  - [2. 核心定义](#2-核心定义)
  - [4. 核心架构](#4-核心架构)
    - [关键组件](#关键组件)
  - [5. 与MCP的关系](#5-与mcp的关系)
    - [区别](#区别)
  - [6. 实际应用示例](#6-实际应用示例)
    - [基本流程](#基本流程)
    - [支持的功能](#支持的功能)
  - [A2ADemo体验](#a2ademo体验)
  - [8. 应用场景](#8-应用场景)

## 1. 诞生背景
- 解决不同Agent之间的互操作问题
- 避免复杂的API集成工作
- 处理复杂的任务场景（长时间运行、多轮对话、流式输出等）

## 2. 核心定义
A2A是一种开放协议，旨在实现不透明Agent之间的通信和互操作，为不同Agent提供共同的"语言"标准。
![[Pasted image 20250418204832.png]]

**主要价值：**
-  降低异构Agent之间的集成复杂性：你无需了解对端Agent的细节。
- 提高Agent能力的可复用性：你可以把某个任务交给更擅长它的Agent。
- 更好的扩展性以适应变化：Agent的内部逻辑变化可以被A2A所隔离。

**A2A协议对Agent之间集成的如下方面进行标准化：**
-  集成架构与关键组件
- 消息与通信机制（JSON-RPC2.0与HTTP）
- 服务端与客户端的功能规范
- 安全验证与授权机制
## 4. 核心架构
### 关键组件
![[Pasted image 20250418205217.png]]
**几个关键的组件构成如下：**

- **Agent Card**：这是Agent的"名片"，代表"我有怎样的任务完成能力"。
- **A2A Server**：访问对端Agent的的入口，基于Web Server运行，提供端点供客户端发起各种请求并给予响应；也会通过连接发起主动通知、任务结果给客户端。
- **A2A Client**：访问A2A Server的其他Agent或定制应用。所以Client与Server是相对的，一个客户端Agent同时也可以是其他Agent的服务端。
- **Task（任务）**：这是客户端交给服务端Agent需要完成的工作任务。任务过程可能需要客户端的协作；任务结果可以同步等待也可以异步获取。
- **Artifact（工件）**：服务端Agent的最终产出被称为Artifact。比如一段文字、一个报告、一个图片或者视频；一次任务可以产生多个Artifact。
- **Messages（消息）**：是指的客户端与服务端Agent之间的多轮沟通内容，存在两个角色：user和agent，内容包括各种上下文、指令、中间状态等。客户端向Agent发送的所有内容都是消息；Agent也可以发送消息给客户端传递状态或指令（比如要求客户端补充信息），但服务端Agent的最终结果则是用Artifact来表示。
- **Notifications（通知）**：服务端Agent向客户端主动推送的信息，比如一个长期任务的中间运行状态，一个任务有了新的产出等。客户端可以设置通知处理回调。       

**所以基本的运行过程就是：**
   
- 服务端Agent启动Server，通过Agent Card展示能力
- 客户端Agent/应用连接服务端，并查看Agent Card    
- 根据Agent Card可以把后续的任务发送给服务端Agent
- 服务端Agent处理任务，必要的时候会输出中间结果，或者要求客户端补充信息
- 客户端可以等待任务完成；也可以根据任务ID异步获取任务结果
- 客户端也可以设置通知回调，及时获得任务的进展与工作成果

## 5. 与[[MCP]]的关系
### 区别

- MCP：解决Agent与外部工具/数据之间的集成（内部事务）
- A2A：解决Agent与Agent之间的集成（更高层次）
![[Pasted image 20250418211705.png]]
![[Pasted image 20250418211752.png]]
## 6. 实际应用示例
### 基本流程
1. 准备Agent（实现invoke与stream接口）
2. 启动A2A Server
3. 客户端测试

### 支持的功能
- 多轮连续对话
- 服务端通知消息
- 异步任务结果获取
- 流式处理

## A2ADemo体验

****【首先准备一个Agent】

这是一个LangGraph开发的智能体，可以独立运行。你需要实现的主要接口是invoke与stream，分别用于非流式与流式调用（在返回消息格式上有特殊处理）。其"轮廓"长这样，内部逻辑无论你是用ReAct Agent还是定义Workflow都不影响：

``` python
@tool  
def search_tavily(query: str, search_depth: str = "basic") -> Dict[str, Any]:  
    ...  
  
class ResponseFormat(BaseModel):  
    """以这种格式回应用户。"""  
    status: Literal["input_required", "completed", "error"] = "input_required"  
    message: str  
  
class SearchAgent:  
  
    SYSTEM_INSTRUCTION = (  
        "你是一个专门进行网络搜索的助手。"  
        "你的主要目的是使用'search_tavily'工具来回答用户问题，提供最新、最相关的信息。"  
        "如果需要用户提供更多信息来进行有效搜索，将响应状态设置为input_required。"  
        "如果处理请求时发生错误，将响应状态设置为error。"  
        "如果请求已完成并提供了答案，将响应状态设置为completed。"  
    )  
     
    def __init__(self):  
        ...  
    def invoke(self, query, sessionId) -> str:
```

先写个本地程序对Agent做测试，确保可用：

![[Pasted image 20250418213544.png]]

**【启动A2A Server】**

现在给这个Agent增加一个前置A2A Server，让它可以对外提供服务，主要逻辑长这样：

```python
def main(host, port):  
    
    try:  
        capabilities = AgentCapabilities(streaming=True, pushNotifications=True)  
        skill = AgentSkill(  
            id="search_web",  
            name="搜索工具",  
            description="搜索web上的相关信息",  
            tags=["Web搜索", "互联网搜索"],  
            examples=["请搜索最新的黑神话悟空的消息"],  
        )  
  
        agent_card = AgentCard(  
            name="搜索助手",  
            description="搜索Web上的相关信息",  
            url=f"http://{host}:{port}/",  
            version="1.0.0",  
            defaultInputModes=SearchAgent.SUPPORTED_CONTENT_TYPES,  
            defaultOutputModes=SearchAgent.SUPPORTED_CONTENT_TYPES,  
            capabilities=capabilities,  
            skills=[skill],  
        )  
  
        notification_sender_auth = PushNotificationSenderAuth()  
        notification_sender_auth.generate_jwk()  
        server = A2AServer(  
            agent_card=agent_card,  
            task_manager=AgentTaskManager(agent=SearchAgent(), notification_sender_auth=notification_sender_auth),  
            host=host,  
            port=port,  
        )  
  
        server.app.add_route(  
            "/.well-known/jwks.json", notification_sender_auth.handle_jwks_endpoint, methods=["GET"]  
        )  
  
        logger.info(f"正在启动服务器，地址：{host}:{port}")  
        server.start()  
    except MissingAPIKeyError as e:  
        logger.error(f"错误：{e}")  
        exit(1)  
    except Exception as e:  
        logger.error(f"服务器启动过程中发生错误：{e}")  
        exit(1)  
  
  
if __name__ == "__main__":  
    main()
```
基本过程是声明Agent的能力->发布成Agent卡片->启动A2AServer（uvicorn)。其核心是一个借助于Agent来完成任务的task_manager。

现在启动这个Server：
![[Pasted image 20250418213844.png]]

****【客户端应用测试】

用官方提供的客户端命令行来测试这个Server。可以看到，客户端启动时会读取服务端的Agent卡片，了解到Agent的能力与特性（比如Streaming代表支持流式）：
![[Pasted image 20250418213929.png]]

现在向这个Agent发送一个任务：

![[Pasted image 20250418214024.png]]
客户端采用了Streaming输出，所以这里可以看到一些中间输出。这些输出可以帮助理解A2A协议的一些标准，比如：

- 中间过程都是用message来传递，发送的角色是agent
- 最终的任务输出则是artifact，代表这是任务的最后产出
- 不管是message还是artifact，具体内容都是有多个part组成

我们来测试几个能力：
###  多轮连续对话

我们输入"我要搜索"，可以看到收到的消息状态是"input-required"，这是A2A协议规定的任务状态之一，代表需要补充信息。通过这种方式，你后续的消息会被追加进入任务输入（这也需要Agent支持），实现多轮对话：

![图片](https://mmbiz.qpic.cn/mmbiz_png/90CnTjsKiae4voyHvnrXte1lKtEKXD8OIfDpFu9EAu34NSqlB3HdGckpvyvX47LPjvLwawgVMbhicVCmia2pyfiaDQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 服务端通知消息

在客户端可以启用通知推送，这时候会在客户端启动一个Server用来等待服务端的通知消息，服务端可以会通过post把任务消息推送到客户端：

![图片](https://mmbiz.qpic.cn/mmbiz_png/90CnTjsKiae4voyHvnrXte1lKtEKXD8OIz7ica1r98CINHTRKM1SWo4Eg7310OAe1rulGEhsOeWyU0SZ4U9NdsoQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 异步任务结果获取

A2A的任务可以完全异步，即使客户端断开也不影响获取任务结果。为了证明这一点，我们另写了一个简单的根据task_id轮询任务结果的程序。然后执行如下步骤：

1. 在Agent中增加一段sleep时间，模拟很长的任务执行时间；然后启动Server；

2. 在官方客户端中启动一个新任务，然后很快强行退出客户端：

![图片](https://mmbiz.qpic.cn/mmbiz_png/90CnTjsKiae4voyHvnrXte1lKtEKXD8OIuMtjE2MShhARiaLWBpicI4Qu8IqqHlu7RnHtyjMG5Ftd2vAf0HsiaaPYw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

3. 拷贝出任务ID（任务ID是客户端生成的，可以直接在客户端打印出来）。

4. 启动我们编写的轮询客户端（带上任务ID），会看到如下结果：

![图片](https://mmbiz.qpic.cn/mmbiz_png/90CnTjsKiae4voyHvnrXte1lKtEKXD8OIfcBHtvCdYVModC4UibEVxa5OkK4C9wR0QyuTRoBx9tr0nSHd4J3SiaHw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

可以看到，尽管发起任务的客户端已经退出，但不会影响你获取任务结果。

这个Demo很好的演示了A2A的一些原理与能力，包括能力发布、流式处理、多轮对话、异步任务等。显然，你也可以把A2A Server作为企业智能体系统的前后端通信的方式（取代FastAPI），可以减少很多复杂问题的工作量。

## 8. 应用场景

- 企业智能体系统的前后端通信
- 分布式Agent协作
- 复杂任务处理
- 多Agent系统集成

这个协议的出现为AI Agent之间的协作提供了标准化的解决方案，使得不同平台、不同供应商的Agent能够更高效地进行交互和协作。

**google官方建议**：We recommend that applications model A2A agents as MCP resources (represented by their [AgentCard](https://google.github.io/A2A/#/documentation?id=agent-card)). The frameworks can then use A2A to communicate with their user, the remote agents, and other agents. （我们建议应用程序将A2A智能体建模为MCP资源（通过它们的AgentCard表示）。然后框架可以使用A2A与用户、远程智能体和其他智能体进行通信）


![[Pasted image 20250418220000.png]]
## 9.引用

微信：https://mp.weixin.qq.com/s/N6Mj0Rhnk6QsN7l32LsrKg?scene=1
Github：https://github.com/google/A2A
技术文档：https://google.github.io/A2A/#/documentation?id=agent-card

https://github.com/google/A2A/blob/main/samples/python/agents/langgraph/README.md
这里有个# LangGraph Currency Agent with A2A Protocol 的例子，有兴趣可以看下：
![[Pasted image 20250418220849.png]]