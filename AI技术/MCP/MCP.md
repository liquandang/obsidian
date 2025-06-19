
## 目录

1. [什么是MCP](#1什么是mcp)
   - MCP的定义与优势
   - MCP架构核心组件
2. [MCP工作原理](#2mcp-工作原理)
   - 通信机制
   - 基本工作流程
3. [MCP的三种形式](#3mcp三种形式)
   - Prompts
   - Tools
   - Resources
4. [MCP与Function Calling的区别](#4mcp与function-calling的区别)
5. [MCP的本质与挑战](#5mcp的本质与挑战)
   - MCP遇到的问题
   - Java MCP
   - 相关资源

## 1、什么MCP
MCP（Model Context Protocol ）是一个用于模型上下文交互的协议，它允许客户端与服务器之间进行标准化的通信，以便调用各种工具和服务。MCP的火，带来了[[AI应用架构新范式]]。

MCP能帮助我们在大型语言模型（LLM）之上构建智能体和复杂工作流。由于LLM经常需要与各类数据和工具集成，MCP提供以下核心优势：
- 不断扩展的预置集成库：您的LLM可直接接入现成解决方案  
- 灵活切换能力：支持在不同LLM服务商之间自由迁移  
- 数据安全最佳实践：确保我们的数据在自有基础设施中获得保护

MCP 采用**客户端-服务器（Client-Server）架构**，其核心设计如下：
![[Pasted image 20250426164558.png]]
**MCP 架构核心组件**

**1、MCP宿主程序（MCP Hosts）**  ：如Claude桌面版、集成开发环境（IDE）等AI工具，需通过MCP协议访问数据的主体程序

**2、MCP客户端（MCP Clients）** ：协议客户端实例，与服务端保持严格1:1的持久化连接

**3、MCP服务端（MCP Servers）** ：轻量化服务程序，通过标准化《模型上下文协议》对外暴露特定能力模块

**4、本地数据源（Local Data Sources）** ：计算机本地文件系统，内嵌数据库服务，其他可被MCP服务端安全调用的本机服务

**5、远程服务（Remote Services）** ：基于互联网API调用的外部系统，需通过MCP服务端中转访问的云端资源
## 2、MCP 工作原理

MCP 协议支持两种主要的通信机制：基于标准输入输出的本地通信和基于[SSE](https://zhida.zhihu.com/search?content_id=254488153&content_type=Article&match_order=1&q=SSE&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NDU4MzcwNTEsInEiOiJTU0UiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNTQ0ODgxNTMsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.pSipZ7M_X8jwE-TOO_a-Xx9wJ5SBy424kmLX73m86E0&zhida_source=entity)（[Server-Sent Events](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/Server-sent_events)）的远程通信。

这两种机制都使用 [JSON-RPC 2.0](https://link.zhihu.com/?target=https%3A//www.jsonrpc.org/specification) 格式进行消息传输，确保了通信的标准化和可扩展性。

- 本地通信**：**通过 stdio 传输数据，适用于在同一台机器上运行的客户端和服务器之间的通信。
- 远程通信**：**利用 [[SSE（server-send-events）]] 与 HTTP 结合，实现跨网络的实时数据传输，适用于需要访问远程资源或分布式部署的场景。

MCP 协议采用了一种独特的架构设计，它将 LLM 与资源之间的通信划分为三个主要部分：客户端、服务器和资源。

客户端负责发送请求给 MCP 服务器，服务器则将这些请求转发给相应的资源。这种分层的设计使得 MCP 协议能够更好地控制访问权限，确保只有经过授权的用户才能访问特定的资源。

以下是 MCP 的基本工作流程：

- 初始化连接：客户端向服务器发送连接请求，建立通信通道。
- 发送请求：客户端根据需求构建请求消息，并发送给服务器。
- 处理请求：服务器接收到请求后，解析请求内容，执行相应的操作（如查询数据库、读取文件等）。
- 返回结果：服务器将处理结果封装成响应消息，发送回客户端。
- 断开连接：任务完成后，客户端可以主动关闭连接或等待服务器超时关闭。
![[v2-bb82edf5b8651051be151c279e7679e1_r.jpg]]
![[Pasted image 20250426200553.png]]

**服务器需提供两个端点：**
- **/sse**（GET请求）：建立长连接，接收服务器推送的事件流。
- **/messages**（POST请求）：客户端发送请求至该端点。

![[Pasted image 20250426214330.png]]

https://zhuanlan.zhihu.com/p/30515707345
## 3、MCP三种形式

### prompts

在 **MCP（Model Context Protocol）** 中，**Prompt（提示词）** 是一种预定义的交互模板，用于引导 AI 模型执行特定任务，类似于"剧本"或"指令模板"。它允许开发者将复杂的多步推理或标准化任务封装成可复用的结构，从而提高 AI 的可靠性和效率。

#### **(1) 标准化任务模板**

- 开发者可以预定义 **Prompt 模板**，例如：
    
    - **代码调试模板**：`"请分析这段代码：[代码片段]，指出潜在的错误并提供修复建议。"`
        
    - **产品描述生成模板**：`"基于以下参数生成产品描述：产品名称=[名称]，功能=[功能]，目标用户=[用户群体]。"`
        
    - **客服 SOP 模板**：`"用户反馈问题：[问题描述]，请根据企业知识库提供解决方案。"`4
        

#### **(2) 动态参数化**

- Prompt 支持 **变量替换**，允许运行时传入动态数据，例如：
 ```python   
 prompt = "请分析这段代码：{code}，错误类型可能是：{error_type}。"
 filled_prompt = prompt.format(code="def foo(x): return x+1", error_type="类型错误")
```
这样，同一个模板可以适配不同场景，而无需重新编写 Prompt。

#### **(3) 可组合性**

- Prompt 可以 **嵌套调用 Tools 和 Resources**，例如：
    - 先通过 **Resource** 读取日志文件 → 再用 **Tool** 查询数据库 → 最后用 **Prompt** 生成报告。
    - 案例：`"读取 {log_file}，查询 {db_table}，总结最近 24 小时的错误趋势。"`

#### **(4) 多模型兼容**

- 由于 MCP 是标准化协议，同一套 Prompt 可以在不同 AI 模型（如 Claude、GPT、Gemini）上运行，只需稍作调整。

#### **Prompt 的具体使用案例**

 **案例 1：自动化代码审查**

- **Prompt 模板**：
```
"请审查以下代码：[代码片段]，重点检查：
1. 潜在的安全漏洞（如 SQL 注入）
2. 性能优化点（如循环优化）
3. 是否符合团队的代码规范？
返回格式：Markdown 列表。"
```
**执行流程**：

1. 用户提交代码片段。
2. MCP Host 调用 **Code Review Prompt**。
3. AI 返回结构化审查报告

#### 总结

MCP 的 **Prompt** 让 AI 从"自由发挥"变成"按剧本执行"，特别适合 **<font color='red'>标准化、可复用</font>** 的任务。无论是代码审查、客服自动化，还是 UI 设计生成，Prompt 都能显著提升 AI 的准确性和效率。未来，随着更多企业采用 MCP，Prompt 可能会像"软件函数"一样，成为 AI 开发的基础模块。

### tools

tools是模型上下文协议（MCP）中的一个强大基础功能，它使得服务器能够向客户端暴露可执行的功能。通过工具，大语言模型（LLMs）可以与外部系统交互、执行计算任务，并在现实世界中采取行动。

tools的设计采用模型控制机制，即服务器向客户端开放工具功能的目的是让AI模型能够自动调用这些工具（同时需经人工审核批准）。

工具的核心特性包括：

- 发现机制：客户端通过tools/list接口获取可用工具列表  
- 调用方式：使用tools/call端点调用工具，服务器执行指定操作后返回结果  
- 灵活扩展：工具功能涵盖从简单计算到复杂API交互等多种场景

与resources类似，tools通过唯一名称进行标识，并可包含使用说明。但不同于静态资源，工具代表的是可修改状态或与外部系统交互的动态操作能力。

``` JAVA
package com.qunar.dzs.qta.qhotel.mcpservers.config;  
  
import com.fasterxml.jackson.databind.ObjectMapper;  
import com.qunar.dzs.qta.qhotel.mcpservers.service.biz.HotelMcpService;  
import io.modelcontextprotocol.server.McpServer;  
import io.modelcontextprotocol.server.McpServerFeatures;  
import io.modelcontextprotocol.server.McpSyncServer;  
import io.modelcontextprotocol.spec.McpSchema;  
import org.springframework.boot.web.servlet.ServletRegistrationBean;  
import org.springframework.context.annotation.Bean;  
import org.springframework.context.annotation.Configuration;  
  
@Configuration  
public class McpServerConfig {  
  
    @Bean  
    public ServletRegistrationBean<JavaxServletHttpServletSseServerTransportProvider> customServletBean(  
            JavaxServletHttpServletSseServerTransportProvider transportProvider) {  
        return new ServletRegistrationBean<>(transportProvider);  
    }  
  
    @Bean  
    public JavaxServletHttpServletSseServerTransportProvider servletSseServerTransportProvider() {  
        return new JavaxServletHttpServletSseServerTransportProvider(new ObjectMapper(), "/mcp/hotel");  
    }  
  
    @Bean  
    public McpSyncServer mcpAsyncServer(JavaxServletHttpServletSseServerTransportProvider transportProvider,  
                                        HotelMcpService hotelMcpService) {  
        McpSyncServer server = McpServer.sync(transportProvider)  
                .serverInfo("qunar hotel mcp server", "1.0.0")  
                .capabilities(McpSchema.ServerCapabilities.builder()  
                        .resources(true, true)  
                        .tools(true)  
                        .prompts(true)  
                        .logging()  
                        .build())  
                .build();  
  
        String createReserveBasePriceTaskSchema = """  
                  {                  "type": "object",                  "properties": {                    "hotelSeq": {                      "type": "string",                      "description": "酒店seq，酒店代号，比如：maanshan_3038、beijing_city_8754"  
                    },                    "checkIn": {                      "type": "string",                      "description": "入住日期，格式yyyy-MM-dd，如果没有这个字段，就取当天日期"  
                    },                    "checkOut": {                      "type": "string",                      "description": "离店日期，格式yyyy-MM-dd，如果没有这个字段，就取(checkIn+1天)"  
                    },                    "crawlRole": {                      "type": "string",                      "description": "用户身份，如果没有这个字段，默认身份：R4"  
                    }                  },                  "required": [                    "hotelSeq",                    "checkIn",                    "checkOut",                    "crawlRole"                  ],                  "additionalProperties": false                }""";  
        server.addTool(new McpServerFeatures.SyncToolSpecification(  
                new McpSchema.Tool("createReserveBasePriceTask", "创建底价归因任务", createReserveBasePriceTaskSchema),  
                (exchange, arguments) -> {  
                    String spaResult = hotelMcpService.createReserveBasePriceTask(String.valueOf(arguments.get("hotelSeq")), String.valueOf(arguments.get("checkIn")), String.valueOf(arguments.get("checkOut")), String.valueOf(arguments.get("crawlRole")));  
                    return new McpSchema.CallToolResult(spaResult, false);  
                }));  
  
  
        String getCurrentTimeSchema = """  
                  {                  "type":"object","properties":{},"required":[],"additionalProperties":false                }""";  
  
        server.addTool(new McpServerFeatures.SyncToolSpecification(  
                new McpSchema.Tool("getCurrentTime", "获取当前时间", getCurrentTimeSchema),  
                (exchange, arguments) -> {  
                    String currentTime = hotelMcpService.getCurrentTime();  
                    return new McpSchema.CallToolResult(currentTime, false);  
                }));  
  
        return server;  
    }  
}
```

```JAVA
package com.qunar.dzs.qta.qhotel.mcpservers.config;/*  
 * Copyright 2024 - 2024 the original author or authors. */  
import com.fasterxml.jackson.core.type.TypeReference;  
import com.fasterxml.jackson.databind.ObjectMapper;  
import io.modelcontextprotocol.spec.*;  
import io.modelcontextprotocol.util.Assert;  
import org.slf4j.Logger;  
import org.slf4j.LoggerFactory;  
import reactor.core.publisher.Flux;  
import reactor.core.publisher.Mono;  
  
import javax.servlet.AsyncContext;  
import javax.servlet.ServletException;  
import javax.servlet.annotation.WebServlet;  
import javax.servlet.http.HttpServlet;  
import javax.servlet.http.HttpServletRequest;  
import javax.servlet.http.HttpServletResponse;  
import java.io.BufferedReader;  
import java.io.IOException;  
import java.io.PrintWriter;  
import java.util.Map;  
import java.util.UUID;  
import java.util.concurrent.ConcurrentHashMap;  
import java.util.concurrent.atomic.AtomicBoolean;  
  
/**  
 * A Servlet-based implementation of the MCP HTTP with Server-Sent Events (SSE) transport * specification. This implementation provides similar functionality to * WebFluxSseServerTransportProvider but uses the traditional Servlet API instead of * WebFlux. * * <p>  
 * The transport handles two types of endpoints:  
 * <ul>  
 * <li>SSE endpoint (/sse) - Establishes a long-lived connection for server-to-client  
 * events</li>  
 * <li>Message endpoint (configurable) - Handles client-to-server message requests</li>  
 * </ul>  
 *  
 * <p>  
 * Features:  
 * <ul>  
 * <li>Asynchronous message handling using Servlet 6.0 async support</li>  
 * <li>Session management for multiple client connections</li>  
 * <li>Graceful shutdown support</li>  
 * <li>Error handling and response formatting</li>  
 * </ul>  
 *  
 * @author Christian Tzolov  
 * @author Alexandros Pappas  
 * @see McpServerTransportProvider  
 * @see HttpServlet  
 */  
@WebServlet(asyncSupported = true)  
public class JavaxServletHttpServletSseServerTransportProvider extends HttpServlet implements McpServerTransportProvider {  
  
   /** Logger for this class */  
   private static final Logger logger = LoggerFactory.getLogger(JavaxServletHttpServletSseServerTransportProvider.class);  
  
   public static final String UTF_8 = "UTF-8";  
  
   public static final String APPLICATION_JSON = "application/json";  
  
   public static final String FAILED_TO_SEND_ERROR_RESPONSE = "Failed to send error response: {}";  
  
   /** Default endpoint path for SSE connections */  
   public static final String DEFAULT_SSE_ENDPOINT = "/sse";  
  
   /** Event type for regular messages */  
   public static final String MESSAGE_EVENT_TYPE = "message";  
  
   /** Event type for endpoint information */  
   public static final String ENDPOINT_EVENT_TYPE = "endpoint";  
  
   public static final String DEFAULT_BASE_URL = "";  
  
   /** JSON object mapper for serialization/deserialization */  
   private final ObjectMapper objectMapper;  
  
   /** Base URL for the server transport */  
   private final String baseUrl;  
  
   /** The endpoint path for handling client messages */  
   private final String messageEndpoint;  
  
   /** The endpoint path for handling SSE connections */  
   private final String sseEndpoint;  
  
   /** Map of active client sessions, keyed by session ID */  
   private final Map<String, McpServerSession> sessions = new ConcurrentHashMap<>();  
  
   /** Flag indicating if the transport is in the process of shutting down */  
   private final AtomicBoolean isClosing = new AtomicBoolean(false);  
  
   /** Session factory for creating new sessions */  
   private McpServerSession.Factory sessionFactory;  
  
   /**  
    * Creates a new HttpServletSseServerTransportProvider instance with a custom SSE    * endpoint.    * @param objectMapper The JSON object mapper to use for message  
    * serialization/deserialization    * @param messageEndpoint The endpoint path where clients will send their messages  
    * @param sseEndpoint The endpoint path where clients will establish SSE connections  
    */   public JavaxServletHttpServletSseServerTransportProvider(ObjectMapper objectMapper, String messageEndpoint,  
                                              String sseEndpoint) {  
      this(objectMapper, DEFAULT_BASE_URL, messageEndpoint, sseEndpoint);  
   }  
  
   /**  
    * Creates a new HttpServletSseServerTransportProvider instance with a custom SSE    * endpoint.    * @param objectMapper The JSON object mapper to use for message  
    * serialization/deserialization    * @param baseUrl The base URL for the server transport  
    * @param messageEndpoint The endpoint path where clients will send their messages  
    * @param sseEndpoint The endpoint path where clients will establish SSE connections  
    */   public JavaxServletHttpServletSseServerTransportProvider(ObjectMapper objectMapper, String baseUrl, String messageEndpoint,  
                                              String sseEndpoint) {  
      this.objectMapper = objectMapper;  
      this.baseUrl = baseUrl;  
      this.messageEndpoint = messageEndpoint;  
      this.sseEndpoint = sseEndpoint;  
   }  
  
   /**  
    * Creates a new HttpServletSseServerTransportProvider instance with the default SSE    * endpoint.    * @param objectMapper The JSON object mapper to use for message  
    * serialization/deserialization    * @param messageEndpoint The endpoint path where clients will send their messages  
    */   public JavaxServletHttpServletSseServerTransportProvider(ObjectMapper objectMapper, String messageEndpoint) {  
      this(objectMapper, messageEndpoint, DEFAULT_SSE_ENDPOINT);  
   }  
  
   /**  
    * Sets the session factory for creating new sessions.    * @param sessionFactory The session factory to use  
    */   @Override  
   public void setSessionFactory(McpServerSession.Factory sessionFactory) {  
      this.sessionFactory = sessionFactory;  
   }  
  
   /**  
    * Broadcasts a notification to all connected clients.    * @param method The method name for the notification  
    * @param params The parameters for the notification  
    * @return A Mono that completes when the broadcast attempt is finished  
    */   @Override  
   public Mono<Void> notifyClients(String method, Object params) {  
      if (sessions.isEmpty()) {  
         logger.debug("No active sessions to broadcast message to");  
         return Mono.empty();  
      }  
  
      logger.debug("Attempting to broadcast message to {} active sessions", sessions.size());  
  
      return Flux.fromIterable(sessions.values())  
         .flatMap(session -> session.sendNotification(method, params)  
            .doOnError(  
                  e -> logger.error("Failed to send message to session {}: {}", session.getId(), e.getMessage()))  
               .onErrorStop())  
         .then();  
   }  
  
   /**  
    * Handles GET requests to establish SSE connections.    * <p>  
    * This method sets up a new SSE connection when a client connects to the SSE  
    * endpoint. It configures the response headers for SSE, creates a new session, and    * sends the initial endpoint information to the client.    * @param request The HTTP servlet request  
    * @param response The HTTP servlet response  
    * @throws ServletException If a servlet-specific error occurs  
    * @throws IOException If an I/O error occurs  
    */   @Override  
   protected void doGet(HttpServletRequest request, HttpServletResponse response)  
         throws ServletException, IOException {  
  
      String requestURI = request.getRequestURI();  
      // 这里只允许建立sse连接
      if (!requestURI.endsWith(sseEndpoint)) {  
         response.sendError(HttpServletResponse.SC_NOT_FOUND);  
         return;      }  
  
      if (isClosing.get()) {  
         response.sendError(HttpServletResponse.SC_SERVICE_UNAVAILABLE, "Server is shutting down");  
         return;      }  
  
      response.setContentType("text/event-stream");  
      response.setCharacterEncoding(UTF_8);  
      response.setHeader("Cache-Control", "no-cache");  
      response.setHeader("Connection", "keep-alive");  
      response.setHeader("Access-Control-Allow-Origin", "*");  
  
      String sessionId = UUID.randomUUID().toString();  
      AsyncContext asyncContext = request.startAsync();  
      asyncContext.setTimeout(0);  
  
      PrintWriter writer = response.getWriter();  
  
      // Create a new session transport  
      HttpServletMcpSessionTransport sessionTransport = new HttpServletMcpSessionTransport(sessionId, asyncContext,  
            writer);  
  
      // Create a new session using the session factory  
      McpServerSession session = sessionFactory.create(sessionTransport);  
      this.sessions.put(sessionId, session);  
  
      // Send initial endpoint event  
      this.sendEvent(writer, ENDPOINT_EVENT_TYPE, this.baseUrl + this.messageEndpoint + "?sessionId=" + sessionId);  
   }  
  
   /**  
    * Handles POST requests for client messages.    * <p>  
    * This method processes incoming messages from clients, routes them through the  
    * session handler, and sends back the appropriate response. It handles error cases    * and formats error responses according to the MCP specification.    * @param request The HTTP servlet request  
    * @param response The HTTP servlet response  
    * @throws ServletException If a servlet-specific error occurs  
    * @throws IOException If an I/O error occurs  
    */   @Override  
   protected void doPost(HttpServletRequest request, HttpServletResponse response)  
         throws ServletException, IOException {  
  
      if (isClosing.get()) {  
         response.sendError(HttpServletResponse.SC_SERVICE_UNAVAILABLE, "Server is shutting down");  
         return;      }  
  
      String requestURI = request.getRequestURI();  
      // post请求只允许建立消息类通信，比如获取工具，调用工具
      if (!requestURI.endsWith(messageEndpoint)) {  
         response.sendError(HttpServletResponse.SC_NOT_FOUND);  
         return;      }  
  
      // Get the session ID from the request parameter  
      String sessionId = request.getParameter("sessionId");  
      if (sessionId == null) {  
         response.setContentType(APPLICATION_JSON);  
         response.setCharacterEncoding(UTF_8);  
         response.setStatus(HttpServletResponse.SC_BAD_REQUEST);  
         String jsonError = objectMapper.writeValueAsString(new McpError("Session ID missing in message endpoint"));  
         PrintWriter writer = response.getWriter();  
         writer.write(jsonError);  
         writer.flush();  
         return;      }  
  
      // Get the session from the sessions map  
      McpServerSession session = sessions.get(sessionId);  
      if (session == null) {  
         response.setContentType(APPLICATION_JSON);  
         response.setCharacterEncoding(UTF_8);  
         response.setStatus(HttpServletResponse.SC_NOT_FOUND);  
         String jsonError = objectMapper.writeValueAsString(new McpError("Session not found: " + sessionId));  
         PrintWriter writer = response.getWriter();  
         writer.write(jsonError);  
         writer.flush();  
         return;      }  
  
      try {  
         BufferedReader reader = request.getReader();  
         StringBuilder body = new StringBuilder();  
         String line;  
         while ((line = reader.readLine()) != null) {  
            body.append(line);  
         }  
  
         McpSchema.JSONRPCMessage message = McpSchema.deserializeJsonRpcMessage(objectMapper, body.toString());  
  
         // Process the message through the session's handle method  
         session.handle(message).block(); // Block for Servlet compatibility  
  
         response.setStatus(HttpServletResponse.SC_OK);  
      }  
      catch (Exception e) {  
         logger.error("Error processing message: {}", e.getMessage());  
         try {  
            McpError mcpError = new McpError(e.getMessage());  
            response.setContentType(APPLICATION_JSON);  
            response.setCharacterEncoding(UTF_8);  
            response.setStatus(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);  
            String jsonError = objectMapper.writeValueAsString(mcpError);  
            PrintWriter writer = response.getWriter();  
            writer.write(jsonError);  
            writer.flush();  
         }  
         catch (IOException ex) {  
            logger.error(FAILED_TO_SEND_ERROR_RESPONSE, ex.getMessage());  
            response.sendError(HttpServletResponse.SC_INTERNAL_SERVER_ERROR, "Error processing message");  
         }  
      }  
   }  
  
   /**  
    * Initiates a graceful shutdown of the transport.    * <p>  
    * This method marks the transport as closing and closes all active client sessions.  
    * New connection attempts will be rejected during shutdown.    * @return A Mono that completes when all sessions have been closed  
    */   @Override  
   public Mono<Void> closeGracefully() {  
      isClosing.set(true);  
      logger.debug("Initiating graceful shutdown with {} active sessions", sessions.size());  
  
      return Flux.fromIterable(sessions.values()).flatMap(McpServerSession::closeGracefully).then();  
   }  
  
   /**  
    * Sends an SSE event to a client.    * @param writer The writer to send the event through  
    * @param eventType The type of event (message or endpoint)  
    * @param data The event data  
    * @throws IOException If an error occurs while writing the event  
    */   private void sendEvent(PrintWriter writer, String eventType, String data) throws IOException {  
      writer.write("event: " + eventType + "\n");  
      writer.write("data: " + data + "\n\n");  
      writer.flush();  
  
      if (writer.checkError()) {  
         throw new IOException("Client disconnected");  
      }  
   }  
  
   /**  
    * Cleans up resources when the servlet is being destroyed.    * <p>  
    * This method ensures a graceful shutdown by closing all client connections before  
    * calling the parent's destroy method.    */   @Override  
   public void destroy() {  
      closeGracefully().block();  
      super.destroy();  
   }  
  
   /**  
    * Implementation of McpServerTransport for HttpServlet SSE sessions. This class    * handles the transport-level communication for a specific client session.    */   private class HttpServletMcpSessionTransport implements McpServerTransport {  
  
      private final String sessionId;  
  
      private final AsyncContext asyncContext;  
  
      private final PrintWriter writer;  
  
      /**  
       * Creates a new session transport with the specified ID and SSE writer.       * @param sessionId The unique identifier for this session  
       * @param asyncContext The async context for the session  
       * @param writer The writer for sending server events to the client  
       */      HttpServletMcpSessionTransport(String sessionId, AsyncContext asyncContext, PrintWriter writer) {  
         this.sessionId = sessionId;  
         this.asyncContext = asyncContext;  
         this.writer = writer;  
         logger.debug("Session transport {} initialized with SSE writer", sessionId);  
      }  
  
      /**  
       * Sends a JSON-RPC message to the client through the SSE connection.       * @param message The JSON-RPC message to send  
       * @return A Mono that completes when the message has been sent  
       */      @Override  
      public Mono<Void> sendMessage(McpSchema.JSONRPCMessage message) {  
         return Mono.fromRunnable(() -> {  
            try {  
               String jsonText = objectMapper.writeValueAsString(message);  
               sendEvent(writer, MESSAGE_EVENT_TYPE, jsonText);  
               logger.debug("Message sent to session {}", sessionId);  
            }  
            catch (Exception e) {  
               logger.error("Failed to send message to session {}: {}", sessionId, e.getMessage());  
               sessions.remove(sessionId);  
               asyncContext.complete();  
            }  
         });  
      }  
  
      /**  
       * Converts data from one type to another using the configured ObjectMapper.       * @param data The source data object to convert  
       * @param typeRef The target type reference  
       * @return The converted object of type T  
       * @param <T> The target type  
       */      @Override  
      public <T> T unmarshalFrom(Object data, TypeReference<T> typeRef) {  
         return objectMapper.convertValue(data, typeRef);  
      }  
  
      /**  
       * Initiates a graceful shutdown of the transport.       * @return A Mono that completes when the shutdown is complete  
       */      @Override  
      public Mono<Void> closeGracefully() {  
         return Mono.fromRunnable(() -> {  
            logger.debug("Closing session transport: {}", sessionId);  
            try {  
               sessions.remove(sessionId);  
               asyncContext.complete();  
               logger.debug("Successfully completed async context for session {}", sessionId);  
            }  
            catch (Exception e) {  
               logger.warn("Failed to complete async context for session {}: {}", sessionId, e.getMessage());  
            }  
         });  
      }  
  
      /**  
       * Closes the transport immediately.       */      @Override  
      public void close() {  
         try {  
            sessions.remove(sessionId);  
            asyncContext.complete();  
            logger.debug("Successfully completed async context for session {}", sessionId);  
         }  
         catch (Exception e) {  
            logger.warn("Failed to complete async context for session {}: {}", sessionId, e.getMessage());  
         }  
      }  
  
   }  
  
   /**  
    * Creates a new Builder instance for configuring and creating instances of    * HttpServletSseServerTransportProvider.    * @return A new Builder instance  
    */   public static Builder builder() {  
      return new Builder();  
   }  
  
   /**  
    * Builder for creating instances of HttpServletSseServerTransportProvider.    * <p>  
    * This builder provides a fluent API for configuring and creating instances of  
    * HttpServletSseServerTransportProvider with custom settings.    */   public static class Builder {  
  
      private ObjectMapper objectMapper = new ObjectMapper();  
  
      private String baseUrl = DEFAULT_BASE_URL;  
  
      private String messageEndpoint;  
  
      private String sseEndpoint = DEFAULT_SSE_ENDPOINT;  
  
      /**  
       * Sets the JSON object mapper to use for message serialization/deserialization.       * @param objectMapper The object mapper to use  
       * @return This builder instance for method chaining  
       */      public Builder objectMapper(ObjectMapper objectMapper) {  
         Assert.notNull(objectMapper, "ObjectMapper must not be null");  
         this.objectMapper = objectMapper;  
         return this;      }  
  
      /**  
       * Sets the base URL for the server transport.       * @param baseUrl The base URL to use  
       * @return This builder instance for method chaining  
       */      public Builder baseUrl(String baseUrl) {  
         Assert.notNull(baseUrl, "Base URL must not be null");  
         this.baseUrl = baseUrl;  
         return this;      }  
  
      /**  
       * Sets the endpoint path where clients will send their messages.       * @param messageEndpoint The message endpoint path  
       * @return This builder instance for method chaining  
       */      public Builder messageEndpoint(String messageEndpoint) {  
         Assert.hasText(messageEndpoint, "Message endpoint must not be empty");  
         this.messageEndpoint = messageEndpoint;  
         return this;      }  
  
      /**  
       * Sets the endpoint path where clients will establish SSE connections.       * <p>  
       * If not specified, the default value of {@link #DEFAULT_SSE_ENDPOINT} will be  
       * used.       * @param sseEndpoint The SSE endpoint path  
       * @return This builder instance for method chaining  
       */      public Builder sseEndpoint(String sseEndpoint) {  
         Assert.hasText(sseEndpoint, "SSE endpoint must not be empty");  
         this.sseEndpoint = sseEndpoint;  
         return this;      }  
  
      /**  
       * Builds a new instance of HttpServletSseServerTransportProvider with the       * configured settings.       * @return A new HttpServletSseServerTransportProvider instance  
       * @throws IllegalStateException if objectMapper or messageEndpoint is not set  
       */      public JavaxServletHttpServletSseServerTransportProvider build() {  
         if (objectMapper == null) {  
            throw new IllegalStateException("ObjectMapper must be set");  
         }  
         if (messageEndpoint == null) {  
            throw new IllegalStateException("MessageEndpoint must be set");  
         }  
         return new JavaxServletHttpServletSseServerTransportProvider(objectMapper, baseUrl, messageEndpoint, sseEndpoint);  
      }  
  
   }  
  
}
```
### resources

// 待完善，用的少
 
## 4、MCP与Function Calling的区别

- MCP是通用协议层的标准，类似于"AI领域的USB-C 接口"，定义了 LLM 与外部工具/数据源的通信格式，但不绑定任何特定模型或厂商，将复杂的函数调用抽象为客户端-服务器架构。

	- **<font color='red'>优点</font>**：统一 MCP 客户端和服务器的运行规范，并且要求 MCP 客户端和服务器之间，也统一按照某个既定的提示词模板进行通信，这样就能通过 MCP Server 加强全球开发者的协作，复用全球的开发成果。

- Function Caling 是大模型厂商提供的专有能力，由大模型厂商定义，不同大模型厂商之间在接口定义和开发文档上存在差异；允许模型直接生成调用函数，触发外部API，依赖模型自身的上下文理解和结构化输出能力。
	
	-  **<font color='red'>缺点</font>**：需要为每个外部函数编写一个 JSON Schema 格式的功能说明，精心设计一个提示词模版，才能提高 Function Calling 响应的准确率，如果一个需求涉及到几十个外部系统，那设计成本是巨大，产品化成本极高（<font color='red'>每个 API 都需要硬编码，为不同平台反复适配，开发和维护成本极高</font>）。
![[Pasted image 20250426172418.png]]

## 5、MCP的本质与挑战

MCP才刚刚开始，很多的问题需要一一被解决。
![[Pasted image 20250426173306.png]]


MCP哪些坑：[https://baijiahao.baidu.com/s?id=1829472267514879056&wfr=spider&for=pc](https://baijiahao.baidu.com/s?id=1829472267514879056&wfr=spider&for=pc)

======================================

官网
https://modelcontextprotocol.io/introduction

https://modelcontextprotocol.io/sdk/java/mcp-overview
https://github.com/modelcontextprotocol/python-sdk
