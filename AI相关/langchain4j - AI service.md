## 官方教程

https://docs.langchain4j.dev/category/tutorials

## 一、MCP DEMO

在介绍之前先看一个调用例子
``` java
package com.hotel.mcp_client.service;  
  
import dev.langchain4j.service.MemoryId;  
import dev.langchain4j.service.Result;  
import dev.langchain4j.service.SystemMessage;  
import dev.langchain4j.service.TokenStream;  
import dev.langchain4j.service.UserMessage;  
import dev.langchain4j.service.V;  
  
/**  
 * @author lijie  create at 2025/4/23  
 */public interface ChatBot {  
  
    @SystemMessage("{{first_prompt}}")  
    TokenStream chatStream(@MemoryId String memoryId, @UserMessage String request, @V("first_prompt") String firstPrompt);  
  
    /**  
     * 归因结果转自然语言  
     *  
     * @param  
     * @return  
     */  
    @SystemMessage("{{sum_prompt}}")  
    @UserMessage("用户提供的数据为:```\n{{dataJson}}\n```")  
    TokenStream chatResult2Ntl(@MemoryId String memoryId, @V("sum_prompt") String systemPrompt, @V("dataJson") String dataJson);  
  
    @SystemMessage("{{sum_prompt}}")  
    @UserMessage("用户提供的数据为:```\n{{dataJson}}\n```")  
    Result<String> chatResult2NtlStr(@MemoryId String memoryId, @V("sum_prompt") String systemPrompt, @V("dataJson") String dataJson);  
  
}
```

下面是组装数据，进行模型调用：
``` JAVA
// 构建模型客户端
QunarAiChatModel gptAiChatModel = QunarAiChatModel.builder().requiredConfig(  
        ClientRequiredConfig.builder()  
            .key("xxxxx")  
            .password("111111").project("ai test")  
            .userIdentityInfo("quandang.li")  
            .build())  
    .optionalConfig(gptConf).build();  

// 构建mcp client ，封装为McpToolProvider。 client、server为1:1，演示这里只是封装一个mcpclient
final HttpMcpTransport transport = new HttpMcpTransport.Builder().sseUrl("http://10.93.125.53:8080/sse").logRequests(true).logResponses(true)  
    .build();  
  
final DefaultMcpClient client = new DefaultMcpClient.Builder().transport(transport).build();  
final McpToolProvider toolProvider = McpToolProvider.builder().mcpClients(client).build();

// 这里是重点：AiServices通过构造器模式，生成ChatBot接口的代理对象（JDK动态代理）
// 将结果类型包装为 Result<String> 以便访问工具执行信息
final ChatBot chatBot = AiServices.builder(ChatBot.class)  
    .streamingChatModel(gptAiChatModel)  
    // .chatModel(model)  
    .chatMemoryProvider(memoryId -> MessageWindowChatMemory.withMaxMessages(1))  
    .toolProvider(toolProvider).build();

// 调用生成的代理对象方法，返回流式对象
TokenStream tokenStream = chatBot.chatStream(UUID.randomUUID().toString(),  
    message, promptConfig.getItem(FIRST_PROMPT));  
StringBuilder stringBuilder = new StringBuilder();  

// 调用tokenStream的start方法开启真正的模型交互
tokenStream.onPartialResponse((String partialResponse) -> {  
        stringBuilder.append(partialResponse);  
        // System.out.println(stringBuilder.toString());  
    })  
    .onToolExecuted((ToolExecution toolExecution) -> {  
        System.out.println("\n调用工具:\n━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");  
        System.out.println("┃ " + toolExecution.request().name());  
        System.out.println("┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");  
        System.out.println("\n返回结果:\n┏━━━━━━━━━━━━━━━━━━━━━━━━━━━");  
        System.out.println("┃ " + toolExecution.result());  
        System.out.println("┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");  
    })  
    .onCompleteResponse((ChatResponse response) -> System.out.println(stringBuilder))  
    .onError((Throwable error) -> error.printStackTrace())  
    .start();
```

## 二、AIService构造器生成动态代理

```JAVA
final ChatBot chatBot = AiServices.builder(ChatBot.class)  
    .streamingChatModel(gptAiChatModel)  
    // .chatModel(model)  
    .chatMemoryProvider(memoryId -> MessageWindowChatMemory.withMaxMessages(1))  
    .toolProvider(toolProvider).build();
```

``` java
// DefaultAiServices的build方法

public T build() {  

	// 校验ChatBot方法的合法性
    performBasicValidation();  
  
    if (!context.hasChatMemory() && ChatMemoryAccess.class.isAssignableFrom(context.aiServiceClass)) {  
        throw illegalConfiguration(  
                "In order to have a service implementing ChatMemoryAccess, please configure the ChatMemoryProvider on the '%s'.",  
                context.aiServiceClass.getName());  
    }  
  
    for (Method method : context.aiServiceClass.getMethods()) {  
        if (method.isAnnotationPresent(Moderate.class) && context.moderationModel == null) {  
            throw illegalConfiguration(  
                    "The @Moderate annotation is present, but the moderationModel is not set up. "  
                            + "Please ensure a valid moderationModel is configured before using the @Moderate annotation.");  
        }  
        if (method.getReturnType() == Result.class  
                || method.getReturnType() == List.class  
                || method.getReturnType() == Set.class) {  
            TypeUtils.validateReturnTypesAreProperlyParametrized(method.getName(), method.getGenericReturnType());  
        }  
  
        if (!context.hasChatMemory()) {  
            for (Parameter parameter : method.getParameters()) {  
                if (parameter.isAnnotationPresent(MemoryId.class)) {  
                    throw illegalConfiguration(  
                            "In order to use @MemoryId, please configure the ChatMemoryProvider on the '%s'.",  
                            context.aiServiceClass.getName());  
                }  
            }  
        }  
    }  

	// 重点来了， 使用JDK动态代理，创建ChatBot的代理对象
    Object proxyInstance = Proxy.newProxyInstance(  
            context.aiServiceClass.getClassLoader(),  
            new Class<?>[] {context.aiServiceClass},  
            new InvocationHandler() {  
  
                private final ExecutorService executor = Executors.newCachedThreadPool();  
  
                @Override  
                public Object invoke(Object proxy, Method method, Object[] args) throws Exception {  
  
					// methods like equals(), hashCode() and toString() should not be handled by this proxy
                    if (method.getDeclaringClass() == Object.class) {
                        return method.invoke(this, args);  
                    }  
					// 如果方法的声明类是ChatMemoryAccess，则进行一些特殊处理返回
                    if (method.getDeclaringClass() == ChatMemoryAccess.class) {  
                        return switch (method.getName()) {  
                            case "getChatMemory" -> context.chatMemoryService.getChatMemory(args[0]);  
                            case "evictChatMemory" -> context.chatMemoryService.evictChatMemory(args[0]) != null;  
                            default -> throw new UnsupportedOperationException(  
                                    "Unknown method on ChatMemoryAccess class : " + method.getName());  
                        };  
                    }  
  
                    validateParameters(method);  
				    // 找到方法中带有MemoryId注解的参数值，获取memoryId，区分用户或新的对话
                    final Object memoryId = findMemoryId(method, args).orElse(ChatMemoryService.DEFAULT);  
                    // 为memoryId创建对应的ChatMemory对象
                    final ChatMemory chatMemory = context.hasChatMemory()  
                            ? context.chatMemoryService.getOrCreateChatMemory(memoryId)  
                            : null;  
                            
				    // 获取systemMessage，即系统级别的提示词。 主要逻辑：
				    // 1、获取方法上的SystemMessage注解对应的value，即提示词
				    // 2、解析方法参数对应name和value，最终替换提示词
                    Optional<SystemMessage> systemMessage = prepareSystemMessage(memoryId, method, args);  
                    // 获取userMessage，即用户query。主要逻辑：
                    // 1、获取方法上或参数上UserMessage注解对应的value，只允许配置1处
				    // 2、解析方法参数对应name和value，最终替换提示词
                    UserMessage userMessage = prepareUserMessage(method, args);  
                    
                    // RAG相关，暂不关注
                    AugmentationResult augmentationResult = null;  
                    if (context.retrievalAugmentor != null) {  
                        List<ChatMessage> chatMemoryMessages = chatMemory != null ? chatMemory.messages() : null;  
                        Metadata metadata = Metadata.from(userMessage, memoryId, chatMemoryMessages);  
                        AugmentationRequest augmentationRequest = new AugmentationRequest(userMessage, metadata);  
                        augmentationResult = context.retrievalAugmentor.augment(augmentationRequest);  
                        userMessage = (UserMessage) augmentationResult.chatMessage();  
                    }  
					// 判断方法返回的类型，是否支持流式输出
                    Type returnType = method.getGenericReturnType();  
                    boolean streaming = returnType == TokenStream.class || canAdaptTokenStreamTo(returnType);  
                    boolean supportsJsonSchema = supportsJsonSchema();  
                    Optional<JsonSchema> jsonSchema = Optional.empty();  
                    if (supportsJsonSchema && !streaming) {  
                        jsonSchema = serviceOutputParser.jsonSchema(returnType);  
                    }  
                    if ((!supportsJsonSchema || jsonSchema.isEmpty()) && !streaming) {  
                        // TODO append after storing in the memory?  
                        userMessage = appendOutputFormatInstructions(returnType, userMessage);  
                    }  

					// 将系统提示词、用户query添加到chatMemory
                    List<ChatMessage> messages;  
                    if (chatMemory != null) {  
                        systemMessage.ifPresent(chatMemory::add);  
                        chatMemory.add(userMessage);  
                        messages = chatMemory.messages();  
                    } else {  
                        messages = new ArrayList<>();  
                        systemMessage.ifPresent(messages::add);  
                        messages.add(userMessage);  
                    }  
                
					// 对于对话内容（移除工具请求和执行结果），通过模型校验是否存在违规内容，如果存在，则会抛出异常（因为是异步，真正的在下面）
                    Future<Moderation> moderationFuture = triggerModerationIfNeeded(method, messages);  

					// 这里很重要： 这里会为MCP server的每个tool创建一个ToolExecutor，具体见单独章节解析《为每个工具创建对应的ToolExecutor》
                    ToolServiceContext toolServiceContext =  
                            context.toolService.createContext(memoryId, userMessage);  
					// 支持流
                    if (streaming) {  
                        // 返回TokenStream对象，并设置messages、所有可用的工具列表、工具对应的executor、rag、AiServiceContext对象等 ，return退出。
                        TokenStream tokenStream = new AiServiceTokenStream(AiServiceTokenStreamParameters.builder()  
                                .messages(messages)  
                                .toolSpecifications(toolServiceContext.toolSpecifications())  
                                .toolExecutors(toolServiceContext.toolExecutors())  
                                .retrievedContents(  
                                        augmentationResult != null ? augmentationResult.contents() : null)  
                                .context(context)  
                                .memoryId(memoryId)  
                                .build());  
                        // TODO moderation  
                        // 流式目前没有支持违规内容的处理 todo
                        if (returnType == TokenStream.class) {  
                            return tokenStream;  
                        } else {  
                            return adapt(tokenStream, returnType);  
                        }  
                    }  

					// 非流式，则直接调用模型处理
                    ResponseFormat responseFormat = null;  
                    if (supportsJsonSchema && jsonSchema.isPresent()) {  
                        responseFormat = ResponseFormat.builder()  
                                .type(JSON)  
                                .jsonSchema(jsonSchema.get())  
                                .build();  
                    }  
  
                    ChatRequestParameters parameters = ChatRequestParameters.builder()  
                            .toolSpecifications(toolServiceContext.toolSpecifications())  
                            .responseFormat(responseFormat)  
                            .build();  
  
                    ChatRequest chatRequest = ChatRequest.builder()  
                            .messages(messages)  
                            .parameters(parameters)  
                            .build();  

					// 正常调用模型	
                    ChatResponse chatResponse = context.chatModel.chat(chatRequest);  
                    
				    // 如果对话内容存在违规，则抛异常
                    verifyModerationIfNeeded(moderationFuture);  
  
                    ToolServiceResult toolServiceResult = context.toolService.executeInferenceAndToolsLoop(  
                            chatResponse,  
                            parameters,  
                            messages,  
                            context.chatModel,  
                            chatMemory,  
                            memoryId,  
                            toolServiceContext.toolExecutors());  
  
                    chatResponse = toolServiceResult.chatResponse();  
                    FinishReason finishReason = chatResponse.metadata().finishReason();  
                    Response<AiMessage> response = Response.from(  
                            chatResponse.aiMessage(), toolServiceResult.tokenUsageAccumulator(), finishReason);  
  
                    Object parsedResponse = serviceOutputParser.parse(response, returnType);  
                    if (typeHasRawClass(returnType, Result.class)) {  
                        return Result.builder()  
                                .content(parsedResponse)  
                                .tokenUsage(toolServiceResult.tokenUsageAccumulator())  
                                .sources(augmentationResult == null ? null : augmentationResult.contents())  
                                .finishReason(finishReason)  
                                .toolExecutions(toolServiceResult.toolExecutions())  
                                .build();  
                    } else {  
                        return parsedResponse;  
                    }  
                }  
  
                private boolean canAdaptTokenStreamTo(Type returnType) {  
                    for (TokenStreamAdapter tokenStreamAdapter : tokenStreamAdapters) {  
                        if (tokenStreamAdapter.canAdaptTokenStreamTo(returnType)) {  
                            return true;  
                        }  
                    }  
                    return false;  
                }  
  
                private Object adapt(TokenStream tokenStream, Type returnType) {  
                    for (TokenStreamAdapter tokenStreamAdapter : tokenStreamAdapters) {  
                        if (tokenStreamAdapter.canAdaptTokenStreamTo(returnType)) {  
                            return tokenStreamAdapter.adapt(tokenStream);  
                        }  
                    }  
                    throw new IllegalStateException("Can't find suitable TokenStreamAdapter");  
                }  
  
                private boolean supportsJsonSchema() {  
                    return context.chatModel != null  
                            && context.chatModel.supportedCapabilities().contains(RESPONSE_FORMAT_JSON_SCHEMA);  
                }  
  
                private UserMessage appendOutputFormatInstructions(Type returnType, UserMessage userMessage) {  
                    String outputFormatInstructions = serviceOutputParser.outputFormatInstructions(returnType);  
                    String text = userMessage.singleText() + outputFormatInstructions;  
                    if (isNotNullOrBlank(userMessage.name())) {  
                        userMessage = UserMessage.from(userMessage.name(), text);  
                    } else {  
                        userMessage = UserMessage.from(text);  
                    }  
                    return userMessage;  
                }  
  
                private Future<Moderation> triggerModerationIfNeeded(Method method, List<ChatMessage> messages) {  
                    if (method.isAnnotationPresent(Moderate.class)) {  
                        return executor.submit(() -> {  
                            List<ChatMessage> messagesToModerate = removeToolMessages(messages);  
                            return context.moderationModel  
                                    .moderate(messagesToModerate)  
                                    .content();  
                        });  
                    }  
                    return null;  
                }  
            });  
  
    return (T) proxyInstance;  
}
```

### 为每个工具创建对应的ToolExecutor

ToolExecutor会在模型返回真正调用的方法时，执行其execute方法，做真正的工具调用。
``` JAVA
public ToolServiceContext createContext(Object memoryId, UserMessage userMessage) {  
    if (this.toolProvider == null) {  
        return this.toolSpecifications.isEmpty() ?  
                new ToolServiceContext(null, null) :  
                new ToolServiceContext(this.toolSpecifications, this.toolExecutors);  
    }  
  
    List<ToolSpecification> toolsSpecs = new ArrayList<>(this.toolSpecifications);  
    Map<String, ToolExecutor> toolExecs = new HashMap<>(this.toolExecutors);  
    ToolProviderRequest toolProviderRequest = new ToolProviderRequest(memoryId, userMessage);  
    // 见如下的provideTools方法，很重要，获取mcpclient对应server的tools
    ToolProviderResult toolProviderResult = toolProvider.provideTools(toolProviderRequest);  
    if (toolProviderResult != null) {  
        for (Map.Entry<ToolSpecification, ToolExecutor> entry :  
                toolProviderResult.tools().entrySet()) {  
                // key = tool， value = executor
            if (toolExecs.putIfAbsent(entry.getKey().name(), entry.getValue()) == null) {  
                toolsSpecs.add(entry.getKey());  
            } else {  
                throw new IllegalConfigurationException(  
                        "Duplicated definition for tool: " + entry.getKey().name());  
            }  
        }  
    }  
    return new ToolServiceContext(toolsSpecs, toolExecs);  
}

// 遍历所有的mcpClients，执行mcpClient.listTools()方法，获取对应mcpserver的所有tools
// 并将tools放入map中
public ToolProviderResult provideTools(final ToolProviderRequest request) {  
    ToolProviderResult.Builder builder = ToolProviderResult.builder();  
    for (McpClient mcpClient : mcpClients) {  
        try {  
            List<ToolSpecification> toolSpecifications = mcpClient.listTools();  
            for (ToolSpecification toolSpecification : toolSpecifications) {  
	            // 重要
                builder.add(  
                        toolSpecification, (executionRequest, memoryId) -> mcpClient.executeTool(executionRequest));  
            }  
        } catch (Exception e) {  
            if (failIfOneServerFails) {  
                throw new RuntimeException("Failed to retrieve tools from MCP server", e);  
            } else {  
                log.warn("Failed to retrieve tools from MCP server", e);  
            }  
        }  
    }  
    return builder.build();  
}

// ToolProviderResult.Builder，这里有个问题，因为是静态类、属性，如果多个server对应tool方法参数完全一致，会被覆盖，不过这种概率很低
public static class Builder {  
  
    private final Map<ToolSpecification, ToolExecutor> tools = new LinkedHashMap<>();  
  
    public Builder add(ToolSpecification tool, ToolExecutor executor) {  
        tools.put(tool, executor);  
        return this;    }  
  
    public Builder addAll(Map<ToolSpecification, ToolExecutor> tools) {  
        this.tools.putAll(tools);  
        return this;    }  
  
    public ToolProviderResult build() {  
        return new ToolProviderResult(tools);  
    }  
}
```

## 三、启动模型交互并接受模型流式返回
``` JAVA
// 调用tokenStream的start方法开启真正的模型交互
tokenStream.onPartialResponse((String partialResponse) -> {  
        stringBuilder.append(partialResponse);  
        // System.out.println(stringBuilder.toString());  
    })  
    .onToolExecuted((ToolExecution toolExecution) -> {  
        System.out.println("\n调用工具:\n━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");  
        System.out.println("┃ " + toolExecution.request().name());  
        System.out.println("┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");  
        System.out.println("\n返回结果:\n┏━━━━━━━━━━━━━━━━━━━━━━━━━━━");  
        System.out.println("┃ " + toolExecution.result());  
        System.out.println("┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");  
    })  
    .onCompleteResponse((ChatResponse response) -> System.out.println(stringBuilder))  
    .onError((Throwable error) -> error.printStackTrace())  
    .start();
```

``` mermaid
sequenceDiagram
    participant User
    participant TokenStream
    participant QunarAiChatModel
    participant AiServiceStreamingResponseHandler
    participant ToolExecutor
    participant MCP工具
    participant ChatMemory

    User->>TokenStream: start()
    TokenStream->>QunarAiChatModel: streamingChatModel.chat(chatRequest, handler)
    QunarAiChatModel->>AiServiceStreamingResponseHandler: 生成流式响应
    AiServiceStreamingResponseHandler-->>TokenStream: onPartialResponse/completeResponse/error
    AiServiceStreamingResponseHandler->>ToolExecutor: (如需工具) execute(toolRequest, memoryId)
    ToolExecutor->>MCP工具: 实际工具调用
    ToolExecutor-->>AiServiceStreamingResponseHandler: 返回工具结果
    AiServiceStreamingResponseHandler->>ChatMemory: 工具结果/AI消息写入memory
    AiServiceStreamingResponseHandler->>QunarAiChatModel: (如需多轮工具) 再次chat
    AiServiceStreamingResponseHandler-->>TokenStream: 最终onCompleteResponse
```


接下来，我们重点关注start方法。
``` JAVA
public void start() {  
    validateConfiguration();  
  
    ChatRequest chatRequest = ChatRequest.builder()  
            .messages(messages)  
            .toolSpecifications(toolSpecifications)  
            .build();  

	// 这个handler很重要，会在模型真正执行过程、完成、异常回调这个类的方法，见章节“模型执行回调” 
    StreamingChatResponseHandler handler = new AiServiceStreamingResponseHandler(  
            context,  
            memoryId,  
            // 来自tokenStream的属性
            /*
            private Consumer<String> partialResponseHandler;  
			private Consumer<List<Content>> contentsHandler;  
			private Consumer<ToolExecution> toolExecutionHandler;  
			private Consumer<ChatResponse> completeResponseHandler;  
			private Consumer<Throwable> errorHandler;
            */
            partialResponseHandler,  
            toolExecutionHandler,  
            completeResponseHandler,  
            errorHandler,  
            initTemporaryMemory(context, messages),  
            new TokenUsage(),  
            toolSpecifications,  
            toolExecutors);  
  
    if (contentsHandler != null && retrievedContents != null) {  
        contentsHandler.accept(retrievedContents);  
    }  

	// 执行模型调用
    context.streamingChatModel.chat(chatRequest, handler);  
}
```

``` JAVA
package dev.langchain4j.model.chat;  
  
import dev.langchain4j.data.message.ChatMessage;  
import dev.langchain4j.data.message.UserMessage;  
import dev.langchain4j.internal.ChatModelListenerUtils;  
import dev.langchain4j.model.ModelProvider;  
import dev.langchain4j.model.chat.listener.ChatModelListener;  
import dev.langchain4j.model.chat.request.ChatRequest;  
import dev.langchain4j.model.chat.request.ChatRequestParameters;  
import dev.langchain4j.model.chat.response.ChatResponse;  
import dev.langchain4j.model.chat.response.StreamingChatResponseHandler;  
  
import java.util.Collections;  
import java.util.List;  
import java.util.Map;  
import java.util.Set;  
import java.util.concurrent.ConcurrentHashMap;  
  
import static dev.langchain4j.internal.ChatModelListenerUtils.onRequest;  
import static dev.langchain4j.internal.ChatModelListenerUtils.onResponse;  
import static dev.langchain4j.model.ModelProvider.OTHER;  
  
/**  
 * Represents a language model that has a chat API and can stream a response one token at a time. * * @see ChatModel  
 */public interface StreamingChatModel {  
  
    /**  
     * This is the main API to interact with the chat model.     * <p>  
     * A temporary default implementation of this method is necessary  
     * until all {@link StreamingChatModel} implementations adopt it. It should be removed once that occurs.  
     *     * @param chatRequest a {@link ChatRequest}, containing all the inputs to the LLM  
     * @param handler     a {@link StreamingChatResponseHandler} that will handle streaming response from the LLM  
     */    default void chat(ChatRequest chatRequest, StreamingChatResponseHandler handler) {  
  
        ChatRequest finalChatRequest = ChatRequest.builder()  
                .messages(chatRequest.messages())  
                .parameters(defaultRequestParameters().overrideWith(chatRequest.parameters()))  
                .build();  
  
        List<ChatModelListener> listeners = listeners();  
        Map<Object, Object> attributes = new ConcurrentHashMap<>();  
  
        StreamingChatResponseHandler observingHandler = new StreamingChatResponseHandler() {  
  
            @Override  
            public void onPartialResponse(String partialResponse) {  
                handler.onPartialResponse(partialResponse);  
            }  
  
            @Override  
            public void onCompleteResponse(ChatResponse completeResponse) {  
                onResponse(completeResponse, finalChatRequest, provider(), attributes, listeners);  
                handler.onCompleteResponse(completeResponse);  
            }  
  
            @Override  
            public void onError(Throwable error) {  
                ChatModelListenerUtils.onError(error, finalChatRequest, provider(), attributes, listeners);  
                handler.onError(error);  
            }  
        };  

		// 目前QunarAiChatModel尚未实现listeners，这个主要是监听模型的请求和响应内容
        onRequest(finalChatRequest, provider(), attributes, listeners);  
        
        doChat(finalChatRequest, observingHandler);  
    }  
  
    ....
    // TODO improve javadoc  
}
```

``` JAVA
public class QunarAiChatModel implements ChatModel, StreamingChatModel {

	public void doChat(final ChatRequest chatRequest, final StreamingChatResponseHandler handler) {  
	    ChatRequestParameters parameters = chatRequest.parameters();  
	    ChatRequestValidator.validateParameters(parameters);  
	  
	    StreamingResponseHandler<AiMessage> legacyHandler = new StreamingResponseHandler<>() {  
	  
	        @Override  
	        public void onNext(String token) {  
	            handler.onPartialResponse(token);  
	        }  
	  
	        @Override  
	        public void onComplete(Response<AiMessage> response) {  
	            ChatResponse chatResponse = ChatResponse.builder()  
	                    .aiMessage(response.content())  
	                    .metadata(ChatResponseMetadata.builder()  
	                            .tokenUsage(response.tokenUsage())  
	                            .finishReason(response.finishReason())  
	                            .build())  
	                    .build();  
	            handler.onCompleteResponse(chatResponse);  
	        }  
	  
	        @Override  
	        public void onError(Throwable error) {  
	            handler.onError(error);  
	        }  
	    };  
	  
	    List<ToolSpecification> toolSpecifications = parameters.toolSpecifications();  
	    if (isNullOrEmpty(toolSpecifications)) {  
	        generate(chatRequest.messages(), legacyHandler);  
	    } else {  
	        if (parameters.toolChoice() == REQUIRED) {  
	            if (toolSpecifications.size() != 1) {  
	                throw new UnsupportedFeatureException(  
	                        String.format("%s.%s is currently supported only when there is a single tool",  
	                                ToolChoice.class.getSimpleName(), REQUIRED.name()));  
	            }  
	            generate(chatRequest.messages(), toolSpecifications.get(0), legacyHandler);  
	        } else {  
	            generate(chatRequest.messages(), toolSpecifications, legacyHandler);  
	        }  
	    }  
	}

	// 如果是glm，则特殊处理
	private void generate(List<ChatMessage> messages, List<ToolSpecification> toolSpecifications, StreamingResponseHandler<AiMessage> handler) {  
	    boolean isGLM = ChatComplexRequest.ApiType.isGLM(optionalConfig.getChatApiType());  
	    if (isGLM) {  
	        glmChatHandle(messages, null, toolSpecifications, handler);  
	    } else {  
	        defaultChatHandle(messages, null, toolSpecifications, handler);  
	    }  
	}
	
	// 真正通过post请求，跟模型交互
	private void defaultChatHandle(List<ChatMessage> messages, ToolSpecification toolSpecification, List<ToolSpecification> toolSpecifications, StreamingResponseHandler<AiMessage> handler) {  
		// getChatFuncCallOrJsonRequest方法来构建整体的prompt
	    Flux<ChatFuncCallResponseStreamItem> flux = client.chatWithFuncCallStream(getChatFuncCallOrJsonRequest(  
	            messages, toolSpecifications, toolSpecification  
	    ));  

		// 流式输出，给到handler处理
	    chatWithFuncCallStreamProcess(handler, flux);  
	}

	// 将mcpsever对应的工具描述放入function，也就是说如果模型不支持function，可能需要放到prompt中
	private ChatFuncCallRequest getChatFuncCallOrJsonRequest(List<ChatMessage> messages,  
	                                                         List<ToolSpecification> toolSpecifications,  
	                                                         ToolSpecification toolThatMustBeExecuted) {  
	  
	    FuncCallPrompt.FuncCallPromptBuilder promptBuilder = FuncCallPrompt.builder()  
	            .frequency_penalty(optionalConfig.getFrequencyPenalty())  
	            .presence_penalty(optionalConfig.getPresencePenalty())  
	            .stop(optionalConfig.getStop())  
	            .temperature(optionalConfig.getTemperature())  
	            .top_p(optionalConfig.getTopP())  
	            .messages(fromChatMessages(messages));  
		// 将tools放入模型的function
	    addFunctions(promptBuilder, toolSpecifications);  
	    if (!CollectionUtils.isEmpty(toolSpecifications)) {  
	        promptBuilder.function_call(  
	                toolThatMustBeExecuted == null ?  
	                        FunctionCallType.createAuto() :  
	                        FunctionCallType.createFunc(toolThatMustBeExecuted.name()));  
	    }  
	  
	    String formattedType = responseFormat == null ? optionalConfig.getResponseFormat() : responseFormat.getType();  
	    if (!Strings.isNullOrEmpty(formattedType)) {  
	        promptBuilder.response_format(ResponseFormat.builder()  
	                .type(formattedType)  
	                .build());  
	    }  
	  
	    return ChatFuncCallRequest.builder()  
	            .prompt(promptBuilder.build())  
	            .apiType(optionalConfig.getChatApiType())  
	            .apiVersion(optionalConfig.getChatApiVersion())  
	            .project(project)  
	            .appCode(ServerManager.getInstance().getAppConfig().getName())  
	            .traceId(QunarAiHelper.getTraceId())  
	            .userIdentityInfo(parseUsername(messages).orElse(userIdentityInfo))  
	            .externalUserIdentityInfo(optionalConfig.getExternalUserIdentityInfo())  
	            .build();  
	}
}
```

```JAVA
private void chatWithFuncCallStreamProcess(StreamingResponseHandler<AiMessage> handler, Flux<ChatFuncCallResponseStreamItem> flux) {  
    flux.doOnError(handler::onError)  
            .subscribe(item -> {  
                        if (ChatResponseStreamItem.StatusEnum.DONE.equals(item.getData().getCurrentStatus())) {  
                            FuncCallReply reply = item.getData().getReply();  
                            FuncCallReply.Choice firstChoice = reply.getChoices().get(0);  
                            if (firstChoice.getFinish_reason() == FinishReasonEnum.FUNCTION_CALL) {  
                                List<ToolExecutionRequest> tools = Lists.newArrayList();  
                                for (FuncCallReply.Choice each : reply.getChoices()) {  
                                    tools.add(ToolExecutionRequest.builder()  
                                            .name(each.getMessage().getFunction_call().getName())  
                                            .arguments(each.getMessage().getFunction_call().getArgsNameValMapJson())  
                                            .build());  
                                }  
                                handler.onComplete(new Response<>(  
                                        new AiMessage(tools),  
                                        new QunarTokenUsage(  
                                                reply.getUsage().getPrompt_tokens(),  
                                                reply.getUsage().getCompletion_tokens(),  
                                                Double.parseDouble(item.getData().getConsume()),  
                                                Double.parseDouble(item.getData().getBalance())  
                                        ),  
                                        FinishReason.TOOL_EXECUTION));  
                            } else if (firstChoice.getFinish_reason().equals(FinishReasonEnum.STOP)) {  
                                handler.onComplete(new Response<>(  
                                        new AiMessage(firstChoice.getMessage().getContent()),  
                                        new QunarTokenUsage(  
                                                reply.getUsage().getPrompt_tokens(),  
                                                reply.getUsage().getCompletion_tokens(),  
                                                Double.parseDouble(item.getData().getConsume()),  
                                                Double.parseDouble(item.getData().getBalance())  
                                        ),  
                                        FinishReason.STOP));  
                            } else if (FinishReasonEnum.CONTENT_FILTER.equals(firstChoice.getFinish_reason())) {  
                                handler.onError(new RuntimeException("content filter"));  
                            }  
                        } else if (ChatResponseStreamItem.StatusEnum.EXECUTING.equals(item.getData().getCurrentStatus())) {  
                            if (item.getData().getChunk() instanceof String) {  
                                return;  
                            }  
                            FuncCallReply chunk = JacksonSerializer.deSerialize(JacksonSerializer.serialize(item.getData().getChunk()), FuncCallReply.class);  
                            List<FuncCallReply.Choice> choices = chunk.getChoices();  
                            if (!choices.isEmpty()) {  
                                FuncCallReply.Delta delta = choices.get(0).getDelta();  
                                // non-null 有的时候会返回空字符串  
                                if (Objects.nonNull(delta.getContent())) {  
                                    //有值才给下游发,finish_reason为function_call的场景content不会有值  
                                    handler.onNext(delta.getContent());  
                                }  
                            }  
                        }  
                    }  
            );  
}
```

### 模型执行回调


``` JAVA
package dev.langchain4j.service;  
  
import dev.langchain4j.agent.tool.ToolExecutionRequest;  
import dev.langchain4j.agent.tool.ToolSpecification;  
import dev.langchain4j.data.message.AiMessage;  
import dev.langchain4j.data.message.ChatMessage;  
import dev.langchain4j.data.message.ToolExecutionResultMessage;  
import dev.langchain4j.model.chat.request.ChatRequest;  
import dev.langchain4j.model.chat.response.ChatResponse;  
import dev.langchain4j.model.chat.response.ChatResponseMetadata;  
import dev.langchain4j.model.chat.response.StreamingChatResponseHandler;  
import dev.langchain4j.model.output.TokenUsage;  
import dev.langchain4j.service.tool.ToolExecution;  
import dev.langchain4j.service.tool.ToolExecutor;  
import org.slf4j.Logger;  
import org.slf4j.LoggerFactory;  
  
import java.util.ArrayList;  
import java.util.List;  
import java.util.Map;  
import java.util.function.Consumer;  
  
import static dev.langchain4j.internal.Utils.copyIfNotNull;  
import static dev.langchain4j.internal.ValidationUtils.ensureNotNull;  
  
/**  
 * Handles response from a language model for AI Service that is streamed token-by-token. * Handles both regular (text) responses and responses with the request to execute one or multiple tools. */class AiServiceStreamingResponseHandler implements StreamingChatResponseHandler {  
  
    private final Logger log = LoggerFactory.getLogger(AiServiceStreamingResponseHandler.class);  
  
    private final AiServiceContext context;  
    private final Object memoryId;  
  
    private final Consumer<String> partialResponseHandler;  
    private final Consumer<ToolExecution> toolExecutionHandler;  
    private final Consumer<ChatResponse> completeResponseHandler;  
  
    private final Consumer<Throwable> errorHandler;  
  
    private final List<ChatMessage> temporaryMemory;  
    private final TokenUsage tokenUsage;  
  
    private final List<ToolSpecification> toolSpecifications;  
    private final Map<String, ToolExecutor> toolExecutors;  
  
    AiServiceStreamingResponseHandler(AiServiceContext context,  
                                      Object memoryId,  
                                      Consumer<String> partialResponseHandler,  
                                      Consumer<ToolExecution> toolExecutionHandler,  
                                      Consumer<ChatResponse> completeResponseHandler,  
                                      Consumer<Throwable> errorHandler,  
                                      List<ChatMessage> temporaryMemory,  
                                      TokenUsage tokenUsage,  
                                      List<ToolSpecification> toolSpecifications,  
                                      Map<String, ToolExecutor> toolExecutors) {  
        this.context = ensureNotNull(context, "context");  
        this.memoryId = ensureNotNull(memoryId, "memoryId");  
  
        this.partialResponseHandler = ensureNotNull(partialResponseHandler, "partialResponseHandler");  
        this.completeResponseHandler = completeResponseHandler;  
        this.toolExecutionHandler = toolExecutionHandler;  
        this.errorHandler = errorHandler;  
  
        this.temporaryMemory = new ArrayList<>(temporaryMemory);  
        this.tokenUsage = ensureNotNull(tokenUsage, "tokenUsage");  
  
        this.toolSpecifications = copyIfNotNull(toolSpecifications);  
        this.toolExecutors = copyIfNotNull(toolExecutors);  
    }  

	// 流式，部分响应回调，会调用TokenStream的partialResponseHandler对象的实现
    @Override  
    public void onPartialResponse(String partialResponse) {  
        partialResponseHandler.accept(partialResponse);  
    }  

	// 模型回复完成回调，这里很重要，会决定是否调用mcp tool
    @Override  
    public void onCompleteResponse(ChatResponse completeResponse) {  

		// 将模型响应结果加入到chatmemory对象
        AiMessage aiMessage = completeResponse.aiMessage();  
        addToMemory(aiMessage);  

		// 如果模型返回的结果，需要执行工具调用。这里是循环，也就是说支持模型一次返回执行多个工具的调用？
        if (aiMessage.hasToolExecutionRequests()) {  
            for (ToolExecutionRequest toolExecutionRequest : aiMessage.toolExecutionRequests()) {
	            // 获取工具执行的名称  
                String toolName = toolExecutionRequest.name();
                // 获取工具对应的执行器  
                ToolExecutor toolExecutor = toolExecutors.get(toolName);  
                // 执行工具调用
                String toolExecutionResult = toolExecutor.execute(toolExecutionRequest, memoryId);  
                // 将工具请求、工具结果封装为ToolExecutionResultMessage对象
                ToolExecutionResultMessage toolExecutionResultMessage = ToolExecutionResultMessage.from(  
                        toolExecutionRequest,  
                        toolExecutionResult  
                );  
                // 工具调用结果添加到chatmemory
                addToMemory(toolExecutionResultMessage);  

				// 如果注册了工具回调handler，则会工具调用完成后，执行工具回调逻辑。本例子中配置了。
                if (toolExecutionHandler != null) {  
                    ToolExecution toolExecution = ToolExecution.builder()  
                            .request(toolExecutionRequest)  
                            .result(toolExecutionResult)  
                            .build();  
                    toolExecutionHandler.accept(toolExecution);  
                }  
            }  

			// 根据memeoryId从chatmemory对象获取所有的对话记录，并设置所有的工具列表
            ChatRequest chatRequest = ChatRequest.builder()  
                    .messages(messagesToSend(memoryId))  
                    .toolSpecifications(toolSpecifications)  
                    .build();  

			// 再次执行模型调用
            StreamingChatResponseHandler handler = new AiServiceStreamingResponseHandler(  
                    context,  
                    memoryId,  
                    partialResponseHandler,  
                    toolExecutionHandler,  
                    completeResponseHandler,  
                    errorHandler,  
                    temporaryMemory,  
                    TokenUsage.sum(tokenUsage, completeResponse.metadata().tokenUsage()),  
                    toolSpecifications,  
                    toolExecutors  
            );  
  
            context.streamingChatModel.chat(chatRequest, handler);  
        } else {  
		    // 如果没有tool调用，则直接调用completeResponseHandler的执行
            if (completeResponseHandler != null) {  
                ChatResponse finalChatResponse = ChatResponse.builder()  
                        .aiMessage(aiMessage)  
                        .metadata(ChatResponseMetadata.builder()  
                                // TODO copy model-specific metadata  
                                .id(completeResponse.metadata().id())  
                                .modelName(completeResponse.metadata().modelName())  
                                .tokenUsage(TokenUsage.sum(tokenUsage, completeResponse.metadata().tokenUsage()))  
                                .finishReason(completeResponse.metadata().finishReason())  
                                .build())  
                        .build();  
                // TODO should completeResponseHandler accept all ChatResponses that happened?  
                completeResponseHandler.accept(finalChatResponse);  
            }  
        }  
    }  
  
    private void addToMemory(ChatMessage chatMessage) {  
        if (context.hasChatMemory()) {  
            context.chatMemoryService.getOrCreateChatMemory(memoryId).add(chatMessage);  
        } else {  
            temporaryMemory.add(chatMessage);  
        }  
    }  
  
    private List<ChatMessage> messagesToSend(Object memoryId) {  
        return context.hasChatMemory()  
                ? context.chatMemoryService.getOrCreateChatMemory(memoryId).messages()  
                : temporaryMemory;  
    }  
	// 模型调用错误，会调用TokenStream的errorHandler对象的实现
    @Override  
    public void onError(Throwable error) {  
        if (errorHandler != null) {  
            try {  
                errorHandler.accept(error);  
            } catch (Exception e) {  
                log.error("While handling the following error...", error);  
                log.error("...the following error happened", e);  
            }  
        } else {  
            log.warn("Ignored error", error);  
        }  
    }  
}
```

## 四、MCP 工具调用

``` JAVA

public ToolProviderResult provideTools(final ToolProviderRequest request) {  
    ToolProviderResult.Builder builder = ToolProviderResult.builder();  
    for (McpClient mcpClient : mcpClients) {  
        try {  
            List<ToolSpecification> toolSpecifications = mcpClient.listTools();  
            for (ToolSpecification toolSpecification : toolSpecifications) { 
		        // 这里是用的是lambda表达式，工具调用会执行 mcpClient.executeTool
                builder.add(  
                        toolSpecification, (executionRequest, memoryId) -> mcpClient.executeTool(executionRequest));  
            }  
        } catch (Exception e) {  
            if (failIfOneServerFails) {  
                throw new RuntimeException("Failed to retrieve tools from MCP server", e);  
            } else {  
                log.warn("Failed to retrieve tools from MCP server", e);  
            }  
        }  
    }  
    return builder.build();  
}
```

``` JAVA
package dev.langchain4j.mcp.client;  
  
...
  
public class DefaultMcpClient implements McpClient {  
  
    private static final Logger log = LoggerFactory.getLogger(DefaultMcpClient.class);  
    private final AtomicLong idGenerator = new AtomicLong(0);  
    private final McpTransport transport;  
    static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();  
    private final String clientName;  
    private final String clientVersion;  
    private final String protocolVersion;  
    private final Duration toolExecutionTimeout;  
    private final Duration resourcesTimeout;  
    private final Duration promptsTimeout;  
    private final Duration pingTimeout;  
    private final JsonNode RESULT_TIMEOUT;  
    private final String toolExecutionTimeoutErrorMessage;  
    private final Map<Long, CompletableFuture<JsonNode>> pendingOperations = new ConcurrentHashMap<>();  
    private final McpOperationHandler messageHandler;  
    private final McpLogMessageHandler logHandler;  
    private final AtomicReference<List<McpResource>> resourceRefs = new AtomicReference<>();  
    private final AtomicReference<List<McpResourceTemplate>> resourceTemplateRefs = new AtomicReference<>();  
    private final AtomicReference<List<McpPrompt>> promptRefs = new AtomicReference<>();  
    private final AtomicReference<List<ToolSpecification>> toolListRefs = new AtomicReference<>();  
    private final AtomicBoolean toolListOutOfDate = new AtomicBoolean(true);  
    private final AtomicReference<CompletableFuture<Void>> toolListUpdateInProgress = new AtomicReference<>(null);  
    private final Duration reconnectInterval;  
  
    public DefaultMcpClient(Builder builder) {  
        transport = ensureNotNull(builder.transport, "transport");  
        clientName = getOrDefault(builder.clientName, "langchain4j");  
        clientVersion = getOrDefault(builder.clientVersion, "1.0");  
        protocolVersion = getOrDefault(builder.protocolVersion, "2024-11-05");  
        toolExecutionTimeout = getOrDefault(builder.toolExecutionTimeout, Duration.ofSeconds(60));  
        resourcesTimeout = getOrDefault(builder.resourcesTimeout, Duration.ofSeconds(60));  
        promptsTimeout = getOrDefault(builder.promptsTimeout, Duration.ofSeconds(60));  
        logHandler = getOrDefault(builder.logHandler, new DefaultMcpLogMessageHandler());  
        pingTimeout = getOrDefault(builder.pingTimeout, Duration.ofSeconds(10));  
        reconnectInterval = getOrDefault(builder.reconnectInterval, Duration.ofSeconds(5));  
        toolExecutionTimeoutErrorMessage =  
                getOrDefault(builder.toolExecutionTimeoutErrorMessage, "There was a timeout executing the tool");  
        RESULT_TIMEOUT = JsonNodeFactory.instance.objectNode();  
        messageHandler = new McpOperationHandler(  
                pendingOperations, transport, logHandler::handleLogMessage, () -> toolListOutOfDate.set(true));  
        ((ObjectNode) RESULT_TIMEOUT)  
                .putObject("result")  
                .putArray("content")  
                .addObject()  
                .put("type", "text")  
                .put("text", toolExecutionTimeoutErrorMessage);  
        transport.onFailure(() -> {  
            try {  
                TimeUnit.MILLISECONDS.sleep(reconnectInterval.toMillis());  
            } catch (InterruptedException e) {  
                throw new RuntimeException(e);  
            }  
            log.info("Trying to reconnect...");  
            initialize();  
        });  
        initialize();  
    }  
  
    private void initialize() {  
        transport.start(messageHandler);  
        long operationId = idGenerator.getAndIncrement();  
        McpInitializeRequest request = new McpInitializeRequest(operationId);  
        InitializeParams params = createInitializeParams();  
        request.setParams(params);  
        try {  
            JsonNode capabilities = transport.initialize(request).get();  
            log.debug("MCP server capabilities: {}", capabilities.get("result"));  
        } catch (Exception e) {  
            throw new RuntimeException(e);  
        } finally {  
            pendingOperations.remove(operationId);  
        }  
    }  
  
    private InitializeParams createInitializeParams() {  
        InitializeParams params = new InitializeParams();  
        params.setProtocolVersion(protocolVersion);  
  
        InitializeParams.ClientInfo clientInfo = new InitializeParams.ClientInfo();  
        clientInfo.setName(clientName);  
        clientInfo.setVersion(clientVersion);  
        params.setClientInfo(clientInfo);  
  
        InitializeParams.Capabilities capabilities = new InitializeParams.Capabilities();  
        InitializeParams.Capabilities.Roots roots = new InitializeParams.Capabilities.Roots();  
        roots.setListChanged(false); // TODO: listChanged is not supported yet  
        capabilities.setRoots(roots);  
        params.setCapabilities(capabilities);  
  
        return params;  
    }  
  
    @Override  
    public List<ToolSpecification> listTools() {  
        if (toolListOutOfDate.get()) {  
            CompletableFuture<Void> updateInProgress = this.toolListUpdateInProgress.get();  
            if (updateInProgress != null) {  
                // if an update is already in progress, wait for it to finish  
                toolListUpdateInProgress.get();  
                return toolListRefs.get();  
            } else {  
                // if no update is in progress, start one  
                CompletableFuture<Void> update = new CompletableFuture<>();  
                this.toolListUpdateInProgress.set(update);  
                try {  
                    obtainToolList();  
                } finally {  
                    update.complete(null);  
                    toolListOutOfDate.set(false);  
                    toolListUpdateInProgress.set(null);  
                }  
                return toolListRefs.get();  
            }  
        } else {  
            return toolListRefs.get();  
        }  
    }  
  
    @Override  
    public String executeTool(ToolExecutionRequest executionRequest) {  
        ObjectNode arguments = null;  
        try {  
            String args = executionRequest.arguments();  
            if (isNullOrBlank(args)) {  
                args = "{}";  
            }  
            arguments = OBJECT_MAPPER.readValue(args, ObjectNode.class);  
        } catch (JsonProcessingException e) {  
            throw new RuntimeException(e);  
        }  
        long operationId = idGenerator.getAndIncrement();  
        McpCallToolRequest operation = new McpCallToolRequest(operationId, executionRequest.name(), arguments);  
        long timeoutMillis = toolExecutionTimeout.toMillis() == 0 ? Integer.MAX_VALUE : toolExecutionTimeout.toMillis();  
        CompletableFuture<JsonNode> resultFuture = null;  
        JsonNode result = null;  
        try {  
	        // 执行工具调用
            resultFuture = transport.executeOperationWithResponse(operation);  
            result = resultFuture.get(timeoutMillis, TimeUnit.MILLISECONDS);  
        } catch (TimeoutException timeout) {  
            transport.executeOperationWithoutResponse(new CancellationNotification(operationId, "Timeout"));  
            return ToolExecutionHelper.extractResult(RESULT_TIMEOUT);  
        } catch (ExecutionException | InterruptedException e) {  
            throw new RuntimeException(e);  
        } finally {  
            pendingOperations.remove(operationId);  
        }  
        return ToolExecutionHelper.extractResult(result);  
    }  
  
    @Override  
    public List<McpResource> listResources() {  
        if (resourceRefs.get() == null) {  
            obtainResourceList();  
        }  
        return resourceRefs.get();  
    }  
  
    @Override  
    public McpReadResourceResult readResource(String uri) {  
        final long operationId = idGenerator.getAndIncrement();  
        McpReadResourceRequest operation = new McpReadResourceRequest(operationId, uri);  
        long timeoutMillis = resourcesTimeout.toMillis() == 0 ? Integer.MAX_VALUE : resourcesTimeout.toMillis();  
        JsonNode result = null;  
        CompletableFuture<JsonNode> resultFuture = null;  
        try {  
            resultFuture = transport.executeOperationWithResponse(operation);  
            result = resultFuture.get(timeoutMillis, TimeUnit.MILLISECONDS);  
            return ResourcesHelper.parseResourceContents(result);  
        } catch (ExecutionException | InterruptedException | TimeoutException e) {  
            throw new RuntimeException(e);  
        } finally {  
            pendingOperations.remove(operationId);  
        }  
    }  
  
    @Override  
    public List<McpPrompt> listPrompts() {  
        if (promptRefs.get() == null) {  
            obtainPromptList();  
        }  
        return promptRefs.get();  
    }  
  
    @Override  
    public McpGetPromptResult getPrompt(String name, Map<String, Object> arguments) {  
        long operationId = idGenerator.getAndIncrement();  
        McpGetPromptRequest operation = new McpGetPromptRequest(operationId, name, arguments);  
        long timeoutMillis = promptsTimeout.toMillis() == 0 ? Integer.MAX_VALUE : promptsTimeout.toMillis();  
        JsonNode result = null;  
        CompletableFuture<JsonNode> resultFuture = null;  
        try {  
            resultFuture = transport.executeOperationWithResponse(operation);  
            result = resultFuture.get(timeoutMillis, TimeUnit.MILLISECONDS);  
            return PromptsHelper.parsePromptContents(result);  
        } catch (ExecutionException | InterruptedException | TimeoutException e) {  
            throw new RuntimeException(e);  
        } finally {  
            pendingOperations.remove(operationId);  
        }  
    }  
  
    @Override  
    public void checkHealth() {  
        transport.checkHealth();  
        long operationId = idGenerator.getAndIncrement();  
        McpPingRequest ping = new McpPingRequest(operationId);  
        try {  
            CompletableFuture<JsonNode> resultFuture = transport.executeOperationWithResponse(ping);  
            resultFuture.get(pingTimeout.toMillis(), TimeUnit.MILLISECONDS);  
        } catch (ExecutionException | InterruptedException | TimeoutException e) {  
            throw new RuntimeException(e);  
        } finally {  
            pendingOperations.remove(operationId);  
        }  
    }  
  
    @Override  
    public List<McpResourceTemplate> listResourceTemplates() {  
        if (resourceTemplateRefs.get() == null) {  
            obtainResourceTemplateList();  
        }  
        return resourceTemplateRefs.get();  
    }  
  
    private synchronized void obtainToolList() {  
        McpListToolsRequest operation = new McpListToolsRequest(idGenerator.getAndIncrement());  
        CompletableFuture<JsonNode> resultFuture = transport.executeOperationWithResponse(operation);  
        JsonNode result = null;  
        try {  
            result = resultFuture.get();  
        } catch (InterruptedException | ExecutionException e) {  
            throw new RuntimeException(e);  
        } finally {  
            pendingOperations.remove(operation.getId());  
        }  
  
        final List<ToolSpecification> toolList = ToolSpecificationHelper.toolSpecificationListFromMcpResponse(  
                (ArrayNode) result.get("result").get("tools"));  
        toolListRefs.set(toolList);  
    }  
  
    private synchronized void obtainResourceList() {  
        if (resourceRefs.get() != null) {  
            return;  
        }  
        McpListResourcesRequest operation = new McpListResourcesRequest(idGenerator.getAndIncrement());  
        long timeoutMillis = resourcesTimeout.toMillis() == 0 ? Integer.MAX_VALUE : resourcesTimeout.toMillis();  
        JsonNode result = null;  
        CompletableFuture<JsonNode> resultFuture = null;  
        try {  
            resultFuture = transport.executeOperationWithResponse(operation);  
            result = resultFuture.get(timeoutMillis, TimeUnit.MILLISECONDS);  
            resourceRefs.set(ResourcesHelper.parseResourceRefs(result));  
        } catch (ExecutionException | InterruptedException | TimeoutException e) {  
            throw new RuntimeException(e);  
        } finally {  
            pendingOperations.remove(operation.getId());  
        }  
    }  
  
    private synchronized void obtainResourceTemplateList() {  
        if (resourceTemplateRefs.get() != null) {  
            return;  
        }  
        McpListResourceTemplatesRequest operation = new McpListResourceTemplatesRequest(idGenerator.getAndIncrement());  
        long timeoutMillis = toolExecutionTimeout.toMillis() == 0 ? Integer.MAX_VALUE : toolExecutionTimeout.toMillis();  
        JsonNode result = null;  
        CompletableFuture<JsonNode> resultFuture = null;  
        try {  
            resultFuture = transport.executeOperationWithResponse(operation);  
            result = resultFuture.get(timeoutMillis, TimeUnit.MILLISECONDS);  
            resourceTemplateRefs.set(ResourcesHelper.parseResourceTemplateRefs(result));  
        } catch (ExecutionException | InterruptedException | TimeoutException e) {  
            throw new RuntimeException(e);  
        } finally {  
            pendingOperations.remove(operation.getId());  
        }  
    }  
  
    private synchronized void obtainPromptList() {  
        if (promptRefs.get() != null) {  
            return;  
        }  
        McpListPromptsRequest operation = new McpListPromptsRequest(idGenerator.getAndIncrement());  
        long timeoutMillis = promptsTimeout.toMillis() == 0 ? Integer.MAX_VALUE : promptsTimeout.toMillis();  
        JsonNode result = null;  
        CompletableFuture<JsonNode> resultFuture = null;  
        try {  
            resultFuture = transport.executeOperationWithResponse(operation);  
            result = resultFuture.get(timeoutMillis, TimeUnit.MILLISECONDS);  
            promptRefs.set(PromptsHelper.parsePromptRefs(result));  
        } catch (ExecutionException | InterruptedException | TimeoutException e) {  
            throw new RuntimeException(e);  
        } finally {  
            pendingOperations.remove(operation.getId());  
        }  
    }  
  
    @Override  
    public void close() {  
        try {  
            transport.close();  
        } catch (Exception e) {  
            log.warn("Cannot close MCP transport", e);  
        }  
    }  
  
    public static class Builder {  
  
        private String toolExecutionTimeoutErrorMessage;  
        private McpTransport transport;  
        private String clientName;  
        private String clientVersion;  
        private String protocolVersion;  
        private Duration toolExecutionTimeout;  
        private Duration resourcesTimeout;  
        private Duration pingTimeout;  
        private Duration promptsTimeout;  
        private McpLogMessageHandler logHandler;  
        private Duration reconnectInterval;  
  
        public Builder transport(McpTransport transport) {  
            this.transport = transport;  
            return this;        }  
  
        /**  
         * Sets the name that the client will use to identify itself to the         * MCP server in the initialization message. The default value is         * "langchain4j".         */        public Builder clientName(String clientName) {  
            this.clientName = clientName;  
            return this;        }  
  
        /**  
         * Sets the version string that the client will use to identify         * itself to the MCP server in the initialization message. The         * default value is "1.0".         */        public Builder clientVersion(String clientVersion) {  
            this.clientVersion = clientVersion;  
            return this;        }  
  
        /**  
         * Sets the protocol version that the client will advertise in the         * initialization message. The default value right now is         * "2024-11-05", but will change over time in later langchain4j         * versions.         */        public Builder protocolVersion(String protocolVersion) {  
            this.protocolVersion = protocolVersion;  
            return this;        }  
  
        /**  
         * Sets the timeout for tool execution.         * This value applies to each tool execution individually.         * The default value is 60 seconds.         * A value of zero means no timeout.         */        public Builder toolExecutionTimeout(Duration toolExecutionTimeout) {  
            this.toolExecutionTimeout = toolExecutionTimeout;  
            return this;        }  
  
        /**  
         * Sets the timeout for resource-related operations (listing resources as well as reading the contents of a resource).         * The default value is 60 seconds.         * A value of zero means no timeout.         */        public Builder resourcesTimeout(Duration resourcesTimeout) {  
            this.resourcesTimeout = resourcesTimeout;  
            return this;        }  
  
        /**  
         * Sets the timeout for prompt-related operations (listing prompts as well as rendering the contents of a prompt).         * The default value is 60 seconds.         * A value of zero means no timeout.         */        public Builder promptsTimeout(Duration promptsTimeout) {  
            this.promptsTimeout = promptsTimeout;  
            return this;        }  
  
        /**  
         * Sets the error message to return when a tool execution times out.         * The default value is "There was a timeout executing the tool".         */        public Builder toolExecutionTimeoutErrorMessage(String toolExecutionTimeoutErrorMessage) {  
            this.toolExecutionTimeoutErrorMessage = toolExecutionTimeoutErrorMessage;  
            return this;        }  
  
        /**  
         * Sets the log message handler for the client.         */        public Builder logHandler(McpLogMessageHandler logHandler) {  
            this.logHandler = logHandler;  
            return this;        }  
  
        /**  
         * The timeout to apply when waiting for a ping response.         * Currently, this is only used in the health check - if the         * server does not send a pong within this timeframe, the health         * check will fail. The timeout is 10 seconds.         */        public Builder pingTimeout(Duration pingTimeout) {  
            this.pingTimeout = pingTimeout;  
            return this;        }  
  
        /**  
         * The delay before attempting to reconnect after a failed connection.         * The default is 5 seconds.         */        public Builder reconnectInterval(Duration reconnectInterval) {  
            this.reconnectInterval = reconnectInterval;  
            return this;        }  
  
        public DefaultMcpClient build() {  
            return new DefaultMcpClient(this);  
        }  
    }  
}
```