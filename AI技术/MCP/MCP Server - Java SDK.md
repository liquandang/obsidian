## 如何搭建MCP Server
```java
package com.qunar.dzs.qta.qhotel.mcpservers.config;  
  
import com.fasterxml.jackson.databind.ObjectMapper;  
import com.qunar.dzs.qta.qhotel.mcpservers.service.biz.HotelMcpService;  
import io.modelcontextprotocol.server.McpServer;  
import io.modelcontextprotocol.server.McpServerFeatures;  
import io.modelcontextprotocol.server.McpSyncServer;  
import io.modelcontextprotocol.spec.McpSchema;  
import org.apache.commons.lang3.StringUtils;  
import org.springframework.boot.web.servlet.ServletRegistrationBean;  
import org.springframework.context.annotation.Bean;  
import org.springframework.context.annotation.Configuration;  
import org.springframework.web.servlet.config.annotation.EnableWebMvc;  
  
import javax.annotation.Resource;  
  
@EnableWebMvc  
@Configuration  
public class McpServerConfig {  
  
    @Resource  
    private PromoptConfig promoptConfig;  
  
	// 因为公司环境的问题，所以重写HttpServletSseServerTransportProvider
    @Bean  
    public ServletRegistrationBean<JavaxServletHttpServletSseServerTransportProviderJavaxServletHttpServletSseServerTransportProvider> customServletBean(  
            JavaxServletHttpServletSseServerTransportProvider transportProvider) {  
        return new ServletRegistrationBean<>(transportProvider);  
    }  

	// 设置messageEndpoint为/mcp/hotel
    @Bean  
    public JavaxServletHttpServletSseServerTransportProvider servletSseServerTransportProvider() {  
        return new JavaxServletHttpServletSseServerTransportProvider(new ObjectMapper(), "/mcp/hotel");  
    }  

	// 构建mcpserver
    @Bean  
    public McpSyncServer mcpAsyncServer(JavaxServletHttpServletSseServerTransportProvider transportProvider,  
                                        HotelMcpService hotelMcpService) {  
        // 设置mcpserver的能力，支持tools、resources、prompts
        McpSyncServer server = McpServer.sync(transportProvider)  
                .serverInfo("qunar hotel mcp server", "1.0.0")  
                .capabilities(McpSchema.ServerCapabilities.builder()  
                        .resources(true, true)  
                        .tools(true)  
                        .prompts(true)  
                        .logging()  
                        .build())  
                .build();  
  
        // 这里使用了动态替换，忽略
        String createReserveBasePriceTaskSchema = """  
                  {                  "type": "object",                  "properties": {                    "hotelSeq": {                      "type": "string",                      "description": %s                    },                    "checkIn": {                      "type": "string",                      "description": %s                    },                    "checkOut": {                      "type": "string",                      "description": %s                    },                    "crawlRole": {                      "type": "string",                      "description": %s                    }                  },                  "required": [                    "hotelSeq",                    "checkIn",                    "checkOut",                    "crawlRole"                  ],                  "additionalProperties": false                }""";  
        // 设置tool的描述
        String createReserveBasePriceTaskDesc = StringUtils.defaultString(promoptConfig.getItem("createReserveBasePriceTask"), "创建QC酒店价格产品力归因分析任务");  
        // 添加tool到server，包括tool的name、描述、参数描述
        server.addTool(new McpServerFeatures.SyncToolSpecification(  
                new McpSchema.Tool("createReserveBasePriceTask", createReserveBasePriceTaskDesc, String.format(createReserveBasePriceTaskSchema, hotelSeqDesc, checkInDesc, checkOutDesc, crawlRoleDesc)),  
                // 真正执行tool的时候，会执行如何逻辑
                (exchange, arguments) -> {  
                    String spaResult = hotelMcpService.createReserveBasePriceTask(String.valueOf(arguments.get("hotelSeq")), String.valueOf(arguments.get("checkIn")), String.valueOf(arguments.get("checkOut")), String.valueOf(arguments.get("crawlRole")));  
                    return new McpSchema.CallToolResult(spaResult, false);  
                }));  
        return server;  
    }  
}
```

接下来我们看看mcpserver的build方法：
```java
McpSyncServer server = McpServer.sync(transportProvider)  
                .serverInfo("qunar hotel mcp server", "1.0.0")  
                .capabilities(McpSchema.ServerCapabilities.builder()  
                        .resources(true, true)  
                        .tools(true)  
                        .prompts(true)  
                        .logging()  
                        .build())  
                .build();  // 这个的build方法，如下。

public McpSyncServer build() {  
   // 默认tools等都是空
   McpServerFeatures.Sync syncFeatures = new McpServerFeatures.Sync(this.serverInfo, this.serverCapabilities,  
         this.tools, this.resources, this.resourceTemplates, this.prompts, this.rootsChangeHandlers,  
         this.instructions);  
   McpServerFeatures.Async asyncFeatures = McpServerFeatures.Async.fromSync(syncFeatures);  
   var mapper = this.objectMapper != null ? this.objectMapper : new ObjectMapper();  
   // 这个是JavaxServletHttpServletSseServerTransportProvider
   var asyncServer = new McpAsyncServer(this.transportProvider, mapper, asyncFeatures);  
   // 最后将异步的server包装为同步的server
   return new McpSyncServer(asyncServer);  
}
```

那么McpAsyncServer的构造方法做了什么呢？
```java
private static class AsyncServerImpl extends McpAsyncServer {  
  
   private final McpServerTransportProvider mcpTransportProvider;  
  
   private final ObjectMapper objectMapper;  
  
   private final McpSchema.ServerCapabilities serverCapabilities;  
  
   private final McpSchema.Implementation serverInfo;  
  
   private final String instructions;  
  
   private final CopyOnWriteArrayList<McpServerFeatures.AsyncToolSpecification> tools = new CopyOnWriteArrayList<>();  
  
   private final CopyOnWriteArrayList<McpSchema.ResourceTemplate> resourceTemplates = new CopyOnWriteArrayList<>();  
  
   private final ConcurrentHashMap<String, McpServerFeatures.AsyncResourceSpecification> resources = new ConcurrentHashMap<>();  
  
   private final ConcurrentHashMap<String, McpServerFeatures.AsyncPromptSpecification> prompts = new ConcurrentHashMap<>();  
  
   // FIXME: this field is deprecated and should be remvoed together with the  
   // broadcasting loggingNotification.   private LoggingLevel minLoggingLevel = LoggingLevel.DEBUG;  
  
   private List<String> protocolVersions = List.of(McpSchema.LATEST_PROTOCOL_VERSION);  
  
   AsyncServerImpl(McpServerTransportProvider mcpTransportProvider, ObjectMapper objectMapper,  
         McpServerFeatures.Async features) {  
      this.mcpTransportProvider = mcpTransportProvider;  
      this.objectMapper = objectMapper;  
      this.serverInfo = features.serverInfo();  
      this.serverCapabilities = features.serverCapabilities();  
      this.instructions = features.instructions();  
      this.tools.addAll(features.tools());  
      this.resources.putAll(features.resources());  
      this.resourceTemplates.addAll(features.resourceTemplates());  
      this.prompts.putAll(features.prompts());  
	  // 这里初始化了很多的请求handler
      Map<String, McpServerSession.RequestHandler<?>> requestHandlers = new HashMap<>();  
  
      // Initialize request handlers for standard MCP methods  
  
      // Ping MUST respond with an empty data, but not NULL response.      requestHandlers.put(McpSchema.METHOD_PING, (exchange, params) -> Mono.just(Map.of()));  
  
      // Add tools API handlers if the tool capability is enabled  
      if (this.serverCapabilities.tools() != null) {  
         requestHandlers.put(McpSchema.METHOD_TOOLS_LIST, toolsListRequestHandler());  
         requestHandlers.put(McpSchema.METHOD_TOOLS_CALL, toolsCallRequestHandler());  
      }  
  
      // Add resources API handlers if provided  
      if (this.serverCapabilities.resources() != null) {  
         requestHandlers.put(McpSchema.METHOD_RESOURCES_LIST, resourcesListRequestHandler());  
         requestHandlers.put(McpSchema.METHOD_RESOURCES_READ, resourcesReadRequestHandler());  
         requestHandlers.put(McpSchema.METHOD_RESOURCES_TEMPLATES_LIST, resourceTemplateListRequestHandler());  
      }  
  
      // Add prompts API handlers if provider exists  
      if (this.serverCapabilities.prompts() != null) {  
         requestHandlers.put(McpSchema.METHOD_PROMPT_LIST, promptsListRequestHandler());  
         requestHandlers.put(McpSchema.METHOD_PROMPT_GET, promptsGetRequestHandler());  
      }  
  
      // Add logging API handlers if the logging capability is enabled  
      if (this.serverCapabilities.logging() != null) {  
         requestHandlers.put(McpSchema.METHOD_LOGGING_SET_LEVEL, setLoggerRequestHandler());  
      }  
  
      Map<String, McpServerSession.NotificationHandler> notificationHandlers = new HashMap<>();  
  
      notificationHandlers.put(McpSchema.METHOD_NOTIFICATION_INITIALIZED, (exchange, params) -> Mono.empty());  
  
      List<BiFunction<McpAsyncServerExchange, List<McpSchema.Root>, Mono<Void>>> rootsChangeConsumers = features  
         .rootsChangeConsumers();  
  
      if (Utils.isEmpty(rootsChangeConsumers)) {  
         rootsChangeConsumers = List.of((exchange,  
               roots) -> Mono.fromRunnable(() -> logger.warn(  
                     "Roots list changed notification, but no consumers provided. Roots list changed: {}",  
                     roots)));  
      }  
  
      notificationHandlers.put(McpSchema.METHOD_NOTIFICATION_ROOTS_LIST_CHANGED,  
            asyncRootsListChangedNotificationHandler(rootsChangeConsumers));  
  
      mcpTransportProvider  
         .setSessionFactory(transport -> new McpServerSession(UUID.randomUUID().toString(), transport,  
               this::asyncInitializeRequestHandler, Mono::empty, requestHandlers, notificationHandlers));  
   }  
  
   // ---------------------------------------  
   // Lifecycle Management   // ---------------------------------------   private Mono<McpSchema.InitializeResult> asyncInitializeRequestHandler(  
         McpSchema.InitializeRequest initializeRequest) {  
      return Mono.defer(() -> {  
         logger.info("Client initialize request - Protocol: {}, Capabilities: {}, Info: {}",  
               initializeRequest.protocolVersion(), initializeRequest.capabilities(),  
               initializeRequest.clientInfo());  
  
         // The server MUST respond with the highest protocol version it supports  
         // if         // it does not support the requested (e.g. Client) version.         String serverProtocolVersion = this.protocolVersions.get(this.protocolVersions.size() - 1);  
  
         if (this.protocolVersions.contains(initializeRequest.protocolVersion())) {  
            // If the server supports the requested protocol version, it MUST  
            // respond            // with the same version.            serverProtocolVersion = initializeRequest.protocolVersion();  
         }  
         else {  
            logger.warn(  
                  "Client requested unsupported protocol version: {}, so the server will sugggest the {} version instead",  
                  initializeRequest.protocolVersion(), serverProtocolVersion);  
         }  
  
         return Mono.just(new McpSchema.InitializeResult(serverProtocolVersion, this.serverCapabilities,  
               this.serverInfo, this.instructions));  
      });  
   }  
  
   public McpSchema.ServerCapabilities getServerCapabilities() {  
      return this.serverCapabilities;  
   }  
  
   public McpSchema.Implementation getServerInfo() {  
      return this.serverInfo;  
   }  
  
   @Override  
   public Mono<Void> closeGracefully() {  
      return this.mcpTransportProvider.closeGracefully();  
   }  
  
   @Override  
   public void close() {  
      this.mcpTransportProvider.close();  
   }  
  
   private McpServerSession.NotificationHandler asyncRootsListChangedNotificationHandler(  
         List<BiFunction<McpAsyncServerExchange, List<McpSchema.Root>, Mono<Void>>> rootsChangeConsumers) {  
      return (exchange, params) -> exchange.listRoots()  
         .flatMap(listRootsResult -> Flux.fromIterable(rootsChangeConsumers)  
            .flatMap(consumer -> consumer.apply(exchange, listRootsResult.roots()))  
            .onErrorResume(error -> {  
               logger.error("Error handling roots list change notification", error);  
               return Mono.empty();  
            })  
            .then());  
   }  
  
   // ---------------------------------------  
   // Tool Management   // ---------------------------------------  
   @Override  
   public Mono<Void> addTool(McpServerFeatures.AsyncToolSpecification toolSpecification) {  
      if (toolSpecification == null) {  
         return Mono.error(new McpError("Tool specification must not be null"));  
      }  
      if (toolSpecification.tool() == null) {  
         return Mono.error(new McpError("Tool must not be null"));  
      }  
      if (toolSpecification.call() == null) {  
         return Mono.error(new McpError("Tool call handler must not be null"));  
      }  
      if (this.serverCapabilities.tools() == null) {  
         return Mono.error(new McpError("Server must be configured with tool capabilities"));  
      }  
  
      return Mono.defer(() -> {  
         // Check for duplicate tool names  
         if (this.tools.stream().anyMatch(th -> th.tool().name().equals(toolSpecification.tool().name()))) {  
            return Mono  
               .error(new McpError("Tool with name '" + toolSpecification.tool().name() + "' already exists"));  
         }  
  
         this.tools.add(toolSpecification);  
         logger.debug("Added tool handler: {}", toolSpecification.tool().name());  
  
         if (this.serverCapabilities.tools().listChanged()) {  
            return notifyToolsListChanged();  
         }  
         return Mono.empty();  
      });  
   }  
  
   @Override  
   public Mono<Void> removeTool(String toolName) {  
      if (toolName == null) {  
         return Mono.error(new McpError("Tool name must not be null"));  
      }  
      if (this.serverCapabilities.tools() == null) {  
         return Mono.error(new McpError("Server must be configured with tool capabilities"));  
      }  
  
      return Mono.defer(() -> {  
         boolean removed = this.tools  
            .removeIf(toolSpecification -> toolSpecification.tool().name().equals(toolName));  
         if (removed) {  
            logger.debug("Removed tool handler: {}", toolName);  
            if (this.serverCapabilities.tools().listChanged()) {  
               return notifyToolsListChanged();  
            }  
            return Mono.empty();  
         }  
         return Mono.error(new McpError("Tool with name '" + toolName + "' not found"));  
      });  
   }  
  
   @Override  
   public Mono<Void> notifyToolsListChanged() {  
      return this.mcpTransportProvider.notifyClients(McpSchema.METHOD_NOTIFICATION_TOOLS_LIST_CHANGED, null);  
   }  
  
   private McpServerSession.RequestHandler<McpSchema.ListToolsResult> toolsListRequestHandler() {  
      return (exchange, params) -> {  
         List<Tool> tools = this.tools.stream().map(McpServerFeatures.AsyncToolSpecification::tool).toList();  
  
         return Mono.just(new McpSchema.ListToolsResult(tools, null));  
      };  
   }  
  
   private McpServerSession.RequestHandler<CallToolResult> toolsCallRequestHandler() {  
      return (exchange, params) -> {  
         McpSchema.CallToolRequest callToolRequest = objectMapper.convertValue(params,  
               new TypeReference<McpSchema.CallToolRequest>() {  
               });  
  
         Optional<McpServerFeatures.AsyncToolSpecification> toolSpecification = this.tools.stream()  
            .filter(tr -> callToolRequest.name().equals(tr.tool().name()))  
            .findAny();  
  
         if (toolSpecification.isEmpty()) {  
            return Mono.error(new McpError("Tool not found: " + callToolRequest.name()));  
         }  
  
         return toolSpecification.map(tool -> tool.call().apply(exchange, callToolRequest.arguments()))  
            .orElse(Mono.error(new McpError("Tool not found: " + callToolRequest.name())));  
      };  
   }  
  
   // ---------------------------------------  
   // Resource Management   // ---------------------------------------  
   @Override  
   public Mono<Void> addResource(McpServerFeatures.AsyncResourceSpecification resourceSpecification) {  
      if (resourceSpecification == null || resourceSpecification.resource() == null) {  
         return Mono.error(new McpError("Resource must not be null"));  
      }  
  
      if (this.serverCapabilities.resources() == null) {  
         return Mono.error(new McpError("Server must be configured with resource capabilities"));  
      }  
  
      return Mono.defer(() -> {  
         if (this.resources.putIfAbsent(resourceSpecification.resource().uri(), resourceSpecification) != null) {  
            return Mono.error(new McpError(  
                  "Resource with URI '" + resourceSpecification.resource().uri() + "' already exists"));  
         }  
         logger.debug("Added resource handler: {}", resourceSpecification.resource().uri());  
         if (this.serverCapabilities.resources().listChanged()) {  
            return notifyResourcesListChanged();  
         }  
         return Mono.empty();  
      });  
   }  
  
   @Override  
   public Mono<Void> removeResource(String resourceUri) {  
      if (resourceUri == null) {  
         return Mono.error(new McpError("Resource URI must not be null"));  
      }  
      if (this.serverCapabilities.resources() == null) {  
         return Mono.error(new McpError("Server must be configured with resource capabilities"));  
      }  
  
      return Mono.defer(() -> {  
         McpServerFeatures.AsyncResourceSpecification removed = this.resources.remove(resourceUri);  
         if (removed != null) {  
            logger.debug("Removed resource handler: {}", resourceUri);  
            if (this.serverCapabilities.resources().listChanged()) {  
               return notifyResourcesListChanged();  
            }  
            return Mono.empty();  
         }  
         return Mono.error(new McpError("Resource with URI '" + resourceUri + "' not found"));  
      });  
   }  
  
   @Override  
   public Mono<Void> notifyResourcesListChanged() {  
      return this.mcpTransportProvider.notifyClients(McpSchema.METHOD_NOTIFICATION_RESOURCES_LIST_CHANGED, null);  
   }  
  
   private McpServerSession.RequestHandler<McpSchema.ListResourcesResult> resourcesListRequestHandler() {  
      return (exchange, params) -> {  
         var resourceList = this.resources.values()  
            .stream()  
            .map(McpServerFeatures.AsyncResourceSpecification::resource)  
            .toList();  
         return Mono.just(new McpSchema.ListResourcesResult(resourceList, null));  
      };  
   }  
  
   private McpServerSession.RequestHandler<McpSchema.ListResourceTemplatesResult> resourceTemplateListRequestHandler() {  
      return (exchange, params) -> Mono  
         .just(new McpSchema.ListResourceTemplatesResult(this.resourceTemplates, null));  
  
   }  
  
   private McpServerSession.RequestHandler<McpSchema.ReadResourceResult> resourcesReadRequestHandler() {  
      return (exchange, params) -> {  
         McpSchema.ReadResourceRequest resourceRequest = objectMapper.convertValue(params,  
               new TypeReference<McpSchema.ReadResourceRequest>() {  
               });  
         var resourceUri = resourceRequest.uri();  
         McpServerFeatures.AsyncResourceSpecification specification = this.resources.get(resourceUri);  
         if (specification != null) {  
            return specification.readHandler().apply(exchange, resourceRequest);  
         }  
         return Mono.error(new McpError("Resource not found: " + resourceUri));  
      };  
   }  
  
   // ---------------------------------------  
   // Prompt Management   // ---------------------------------------  
   @Override  
   public Mono<Void> addPrompt(McpServerFeatures.AsyncPromptSpecification promptSpecification) {  
      if (promptSpecification == null) {  
         return Mono.error(new McpError("Prompt specification must not be null"));  
      }  
      if (this.serverCapabilities.prompts() == null) {  
         return Mono.error(new McpError("Server must be configured with prompt capabilities"));  
      }  
  
      return Mono.defer(() -> {  
         McpServerFeatures.AsyncPromptSpecification specification = this.prompts  
            .putIfAbsent(promptSpecification.prompt().name(), promptSpecification);  
         if (specification != null) {  
            return Mono.error(new McpError(  
                  "Prompt with name '" + promptSpecification.prompt().name() + "' already exists"));  
         }  
  
         logger.debug("Added prompt handler: {}", promptSpecification.prompt().name());  
  
         // Servers that declared the listChanged capability SHOULD send a  
         // notification,         // when the list of available prompts changes         if (this.serverCapabilities.prompts().listChanged()) {  
            return notifyPromptsListChanged();  
         }  
         return Mono.empty();  
      });  
   }  
  
   @Override  
   public Mono<Void> removePrompt(String promptName) {  
      if (promptName == null) {  
         return Mono.error(new McpError("Prompt name must not be null"));  
      }  
      if (this.serverCapabilities.prompts() == null) {  
         return Mono.error(new McpError("Server must be configured with prompt capabilities"));  
      }  
  
      return Mono.defer(() -> {  
         McpServerFeatures.AsyncPromptSpecification removed = this.prompts.remove(promptName);  
  
         if (removed != null) {  
            logger.debug("Removed prompt handler: {}", promptName);  
            // Servers that declared the listChanged capability SHOULD send a  
            // notification, when the list of available prompts changes            if (this.serverCapabilities.prompts().listChanged()) {  
               return this.notifyPromptsListChanged();  
            }  
            return Mono.empty();  
         }  
         return Mono.error(new McpError("Prompt with name '" + promptName + "' not found"));  
      });  
   }  
  
   @Override  
   public Mono<Void> notifyPromptsListChanged() {  
      return this.mcpTransportProvider.notifyClients(McpSchema.METHOD_NOTIFICATION_PROMPTS_LIST_CHANGED, null);  
   }  
  
   private McpServerSession.RequestHandler<McpSchema.ListPromptsResult> promptsListRequestHandler() {  
      return (exchange, params) -> {  
         // TODO: Implement pagination  
         // McpSchema.PaginatedRequest request = objectMapper.convertValue(params,         // new TypeReference<McpSchema.PaginatedRequest>() {         // });  
         var promptList = this.prompts.values()  
            .stream()  
            .map(McpServerFeatures.AsyncPromptSpecification::prompt)  
            .toList();  
  
         return Mono.just(new McpSchema.ListPromptsResult(promptList, null));  
      };  
   }  
  
   private McpServerSession.RequestHandler<McpSchema.GetPromptResult> promptsGetRequestHandler() {  
      return (exchange, params) -> {  
         McpSchema.GetPromptRequest promptRequest = objectMapper.convertValue(params,  
               new TypeReference<McpSchema.GetPromptRequest>() {  
               });  
  
         // Implement prompt retrieval logic here  
         McpServerFeatures.AsyncPromptSpecification specification = this.prompts.get(promptRequest.name());  
         if (specification == null) {  
            return Mono.error(new McpError("Prompt not found: " + promptRequest.name()));  
         }  
  
         return specification.promptHandler().apply(exchange, promptRequest);  
      };  
   }  
  
   // ---------------------------------------  
   // Logging Management   // ---------------------------------------  
   @Override  
   public Mono<Void> loggingNotification(LoggingMessageNotification loggingMessageNotification) {  
  
      if (loggingMessageNotification == null) {  
         return Mono.error(new McpError("Logging message must not be null"));  
      }  
  
      if (loggingMessageNotification.level().level() < minLoggingLevel.level()) {  
         return Mono.empty();  
      }  
  
      return this.mcpTransportProvider.notifyClients(McpSchema.METHOD_NOTIFICATION_MESSAGE,  
            loggingMessageNotification);  
   }  
  
   private McpServerSession.RequestHandler<Object> setLoggerRequestHandler() {  
      return (exchange, params) -> {  
         return Mono.defer(() -> {  
  
            SetLevelRequest newMinLoggingLevel = objectMapper.convertValue(params,  
                  new TypeReference<SetLevelRequest>() {  
                  });  
  
            exchange.setMinLoggingLevel(newMinLoggingLevel.level());  
  
            // FIXME: this field is deprecated and should be removed together  
            // with the broadcasting loggingNotification.            this.minLoggingLevel = newMinLoggingLevel.level();  
  
            return Mono.just(Map.of());  
         });  
      };  
   }  
  
   // ---------------------------------------  
   // Sampling   // ---------------------------------------  
   @Override  
   void setProtocolVersions(List<String> protocolVersions) {  
      this.protocolVersions = protocolVersions;  
   }  
  
}
```

## MCP Client访问Server

### MCP Client和Server调用

DefaultMcpClient -> HttpMcpTransport/StreamableHttpMcpTransport -> JavaxServletHttpServletSseServerTransportProvider（server端）->  McpServerSession（执行对应的handler） -> HttpServletMcpSessionTransport（sse通道将结果发送会client端）
### MCP Client请求server的几种消息类型

![[bec5cf73-070f-424b-b1c2-bb04fa518b4c 1.png]]
``` java
public enum ClientMethod {  
    @JsonProperty("initialize")  
    INITIALIZE,  
    @JsonProperty("tools/call")  
    TOOLS_CALL,  
    @JsonProperty("tools/list")  
    TOOLS_LIST,  
    @JsonProperty("notifications/cancelled")  
    NOTIFICATION_CANCELLED,  
    @JsonProperty("notifications/initialized")  
    NOTIFICATION_INITIALIZED,  
    @JsonProperty("ping")  
    PING,  
    @JsonProperty("resources/list")  
    RESOURCES_LIST,  
    @JsonProperty("resources/read")  
    RESOURCES_READ,  
    @JsonProperty("resources/templates/list")  
    RESOURCES_TEMPLATES_LIST,  
    @JsonProperty("prompts/list")  
    PROMPTS_LIST,  
    @JsonProperty("prompts/get")  
    PROMPTS_GET  
}
```
### 建立sse通道

```JAVA

public class DefaultMcpClient implements McpClient {
	
	private void initialize() {  
		// 建立通道 创建EventSource、监听服务端的结果返回
	    transport.start(messageHandler);  
	    long operationId = idGenerator.getAndIncrement();  
	    McpInitializeRequest request = new McpInitializeRequest(operationId);  
	    InitializeParams params = createInitializeParams();  
	    request.setParams(params);  
	    try {  
		    // 返回server端的能力
	        JsonNode capabilities = transport.initialize(request).get();  
	        log.debug("MCP server capabilities: {}", capabilities.get("result"));  
	    } catch (Exception e) {  
	        throw new RuntimeException(e);  
	    } finally {  
	        pendingOperations.remove(operationId);  
	    }  
	}
}

public class HttpMcpTransport implements McpTransport {
	private EventSource startSseChannel(boolean logResponses) {  
	    Request request = new Request.Builder().url(sseUrl).build();  
	    CompletableFuture<String> initializationFinished = new CompletableFuture<>();  
	    // 创建服务器结果返回的监听器
	    SseEventListener listener =  
	            new SseEventListener(messageHandler, logResponses, initializationFinished, onFailure);  
	    EventSource eventSource = EventSources.createFactory(client).newEventSource(request, listener);  
	    // wait for the SSE channel to be created, receive the POST url from the server, throw an exception if that  
	    // failed    try {  
	        int timeout = client.callTimeoutMillis() > 0 ? client.callTimeoutMillis() : Integer.MAX_VALUE;  
	        String relativePostUrl = initializationFinished.get(timeout, TimeUnit.MILLISECONDS);  
	        postUrl = buildAbsolutePostUrl(relativePostUrl);  
	        log.debug("Received the server's POST URL: {}", postUrl);  
	    } catch (Exception e) {  
	        throw new RuntimeException(e);  
	    }  
	    return eventSource;  
	}
}
```

建立通道直接到服务端的servlet的拦截器，执行doget方法，创建sessionid，并将sessionid传回客户端：/mcp/hotel?sessionid=sxxxxxx

### 初始化

执行transport.initialize(request)，这里的request为McpInitializeRequest对象。
``` java
public class HttpMcpTransport implements McpTransport {
	@Override  
	public CompletableFuture<JsonNode> initialize(McpInitializeRequest operation) {  
	    Request httpRequest = null;  
	    Request initializationNotification = null;  
	    try {  
	        httpRequest = createRequest(operation);  
	        initializationNotification = createRequest(new InitializationNotification());  
	    } catch (JsonProcessingException e) {  
	        return CompletableFuture.failedFuture(e);  
	    }  
	    final Request finalInitializationNotification = initializationNotification;  
	    // 首先执行初始化，执行服务器端的initialize方法，然后执行InitializationNotification，服务器端是notifications/initialized方法
	    return execute(httpRequest, operation.getId())  
	            .thenCompose(originalResponse -> execute(finalInitializationNotification, null)  
	                    .thenCompose(nullNode -> CompletableFuture.completedFuture(originalResponse)));  
	}
}

public class McpInitializeRequest extends McpClientMessage {  
  
    @JsonInclude  
    public final ClientMethod method = ClientMethod.INITIALIZE;  
  
    private InitializeParams params;  
  
    public McpInitializeRequest(final Long id) {  
        super(id);  
    }  
  
    public InitializeParams getParams() {  
        return params;  
    }  
  
    public void setParams(final InitializeParams params) {  
        this.params = params;  
    }  
}

public class InitializationNotification extends McpClientMessage {  

	//notifications/initialized
    @JsonInclude  
    public final ClientMethod method = ClientMethod.NOTIFICATION_INITIALIZED;  
  
    public InitializationNotification() {  
        super(null);  
    }  
}
```

那我们看看服务器端这2个方法都干了什么。McpInitializeRequest的请求返回服务器的所有资源、工具、prompts等，而InitializationNotification返回了空。但是比较蛋疼的是，初始化的时候虽然返回tools等给客户端，但是客户端并没有做任何处理，只是打印出来日志而已。
``` JAVA
public class McpAsyncServer {
	private Mono<McpSchema.InitializeResult> asyncInitializeRequestHandler(  
	      McpSchema.InitializeRequest initializeRequest) {  
	   return Mono.defer(() -> {  
	      logger.info("Client initialize request - Protocol: {}, Capabilities: {}, Info: {}",  
	            initializeRequest.protocolVersion(), initializeRequest.capabilities(),  
	            initializeRequest.clientInfo());  
	  
	      // The server MUST respond with the highest protocol version it supports  
	      // if      // it does not support the requested (e.g. Client) version.      String serverProtocolVersion = this.protocolVersions.get(this.protocolVersions.size() - 1);  
	  
	      if (this.protocolVersions.contains(initializeRequest.protocolVersion())) {  
	         // If the server supports the requested protocol version, it MUST  
	         // respond         // with the same version.         serverProtocolVersion = initializeRequest.protocolVersion();  
	      }  
	      else {  
	         logger.warn(  
	               "Client requested unsupported protocol version: {}, so the server will sugggest the {} version instead",  
	               initializeRequest.protocolVersion(), serverProtocolVersion);  
	      }  
		  // 返回服务端的能力（tool、prompts、resources）、服务信息等
	      return Mono.just(new McpSchema.InitializeResult(serverProtocolVersion, this.serverCapabilities,  
	            this.serverInfo, this.instructions));  
	   });  
	}
}
```

### 获取tools/prompts/resources列表
