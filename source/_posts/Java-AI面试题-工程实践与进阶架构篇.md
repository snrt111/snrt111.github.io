---
title: Java AI面试题：工程实践与进阶架构篇（8道）
date: 2026-05-18 17:20:00
tags: [Java, AI, 面试题, Function Calling, Agent, 多模态, 限流熔断, 多租户]
categories:
  - 面试题
---

# Java AI面试题：工程实践与进阶架构篇（8道）

本文整理了工程实践和进阶架构相关的8道面试题，涵盖限流熔断、多租户、对话记忆、安全过滤、Function Calling、Agent、多模态等高级主题，帮助你掌握生产级AI应用的开发和架构设计。

---

## 一、工程实践篇

### 1. 如何实现AI应用的限流和熔断？

**答案：**

```java
@Component
public class AIClientWrapper {
    
    private final ChatClient chatClient;
    private final RateLimiter rateLimiter;
    private final CircuitBreaker circuitBreaker;
    
    public AIClientWrapper(ChatClient chatClient) {
        this.chatClient = chatClient;
        // 令牌桶限流：每秒10个请求
        this.rateLimiter = RateLimiter.create(10.0);
        // 熔断器配置
        this.circuitBreaker = CircuitBreaker.ofDefaults("ai-client");
    }
    
    public String safeCall(String message) {
        // 限流检查
        if (!rateLimiter.tryAcquire()) {
            throw new RateLimitException("请求过于频繁，请稍后重试");
        }
        
        // 熔断保护
        return circuitBreaker.executeSupplier(() -> {
            try {
                return chatClient.call(message);
            } catch (Exception e) {
                throw new AIClientException("AI服务调用失败", e);
            }
        });
    }
}
```

**Resilience4j配置：**
```yaml
resilience4j:
  circuitbreaker:
    instances:
      ai-client:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
```

---

### 2. 如何设计一个支持多租户的AI知识库系统？

**答案：**

```java
@Service
public class MultiTenantKnowledgeService {
    
    @Autowired
    private VectorStore vectorStore;
    
    // 添加文档（带租户隔离）
    public void addDocument(String tenantId, String content) {
        Document doc = new Document(content);
        // 添加租户ID到元数据
        doc.getMetadata().put("tenant_id", tenantId);
        vectorStore.add(List.of(doc));
    }
    
    // 租户隔离的搜索
    public List<Document> search(String tenantId, String query, int topK) {
        // 使用filter表达式实现租户隔离
        String filterExpr = String.format("tenant_id == '%s'", tenantId);
        
        return vectorStore.similaritySearch(
            SearchRequest.query(query)
                .withTopK(topK)
                .withFilterExpression(filterExpr)
        );
    }
}
```

**数据隔离方案对比：**

| 方案 | 实现方式 | 适用场景 |
|------|----------|----------|
| 索引隔离 | 每个租户独立索引 | 数据量大、安全要求高 |
| 命名空间隔离 | 同一索引不同前缀 | 中等规模 |
| 元数据过滤 | 统一索引+过滤条件 | 小规模、成本敏感 |

---

### 3. 如何实现AI对话的历史记忆功能？

**答案：**

```java
@Service
public class ChatMemoryService {
    
    @Autowired
    private ChatClient chatClient;
    
    // 使用Redis存储对话历史
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    private static final int MAX_HISTORY = 10;
    private static final String KEY_PREFIX = "chat:history:";
    
    public String chatWithMemory(String sessionId, String message) {
        String key = KEY_PREFIX + sessionId;
        
        // 获取历史对话
        List<String> history = redisTemplate.opsForList().range(key, 0, -1);
        
        // 构建带上下文的提示词
        Prompt prompt = buildPromptWithHistory(history, message);
        
        // 调用模型
        String response = chatClient.call(prompt);
        
        // 保存对话记录
        redisTemplate.opsForList().rightPushAll(key, 
            "User: " + message, 
            "AI: " + response
        );
        
        // 限制历史长度
        redisTemplate.opsForList().trim(key, -MAX_HISTORY * 2, -1);
        
        return response;
    }
    
    private Prompt buildPromptWithHistory(List<String> history, String message) {
        StringBuilder context = new StringBuilder();
        if (history != null) {
            history.forEach(h -> context.append(h).append("\n"));
        }
        context.append("User: ").append(message).append("\nAI:");
        
        return new Prompt(new UserMessage(context.toString()));
    }
}
```

**Spring AI Advisor方式：**
```java
@Bean
public ChatClient chatClient(ChatClient.Builder builder) {
    return builder
        .defaultAdvisors(
            new MessageChatMemoryAdvisor(new InMemoryChatMemory()),
            new SimpleLoggerAdvisor()
        )
        .build();
}
```

---

### 4. 如何处理AI生成内容的安全性问题？

**答案：**

```java
@Component
public class ContentSafetyFilter {
    
    private final List<String> sensitiveWords = Arrays.asList(
        "敏感词1", "敏感词2"
    );
    
    // 输入过滤
    public boolean validateInput(String input) {
        if (input == null || input.length() > 4000) {
            return false;
        }
        return !containsSensitiveWords(input);
    }
    
    // 输出过滤
    public String filterOutput(String output) {
        // 脱敏处理
        String filtered = maskSensitiveInfo(output);
        // 添加免责声明
        return addDisclaimer(filtered);
    }
    
    private boolean containsSensitiveWords(String text) {
        return sensitiveWords.stream().anyMatch(text::contains);
    }
    
    private String maskSensitiveInfo(String text) {
        // 身份证号脱敏
        return text.replaceAll("\\d{17}[\\dXx]", "***身份证已脱敏***")
                   .replaceAll("1[3-9]\\d{9}", "***手机号已脱敏***");
    }
    
    private String addDisclaimer(String text) {
        return text + "\n\n【免责声明：以上内容由AI生成，仅供参考】";
    }
}
```

**多层次安全策略：**
1. **输入层**：长度限制、敏感词过滤、SQL注入防护
2. **模型层**：使用安全微调过的模型、设置安全提示词
3. **输出层**：内容审核、敏感信息脱敏、人工复核

---

## 二、进阶架构篇

### 5. 什么是Function Calling（函数调用）？如何在Java中实现？

**答案：**

Function Calling允许大模型根据对话上下文，决定调用外部函数来获取数据或执行操作。

```java
@Configuration
public class FunctionConfiguration {
    
    @Bean
    @Description("获取指定城市的天气信息")
    public Function<WeatherRequest, WeatherResponse> weatherFunction() {
        return request -> {
            // 调用天气API
            return weatherService.getWeather(request.city(), request.date());
        };
    }
    
    @Bean
    @Description("查询订单信息")
    public Function<OrderQueryRequest, OrderResponse> orderQueryFunction() {
        return request -> orderService.queryOrder(request.orderId());
    }
}

// 请求/响应定义
public record WeatherRequest(String city, String date) {}
public record WeatherResponse(String city, String temperature, String condition) {}
```

**使用：**
```java
@Service
public class AssistantService {
    
    @Autowired
    private ChatClient chatClient;
    
    public String chat(String message) {
        return chatClient.prompt()
                .user(message)
                .functions("weatherFunction", "orderQueryFunction")
                .call()
                .content();
    }
}
```

---

### 6. 什么是Agent（智能体）？与传统AI应用有什么区别？

**答案：**

Agent是能够自主感知环境、做出决策并执行动作的智能系统。

**对比：**

| 特性 | 传统AI应用 | Agent |
|------|------------|-------|
| 交互方式 | 单次请求-响应 | 多轮自主执行 |
| 决策能力 | 被动响应 | 主动规划 |
| 工具使用 | 预定义 | 动态选择和调用 |
| 记忆能力 | 短期 | 长期 + 反思 |

**Java中实现简单Agent：**
```java
@Component
public class SimpleAgent {
    
    @Autowired
    private ChatClient chatClient;
    
    private final List<Tool> tools = new ArrayList<>();
    
    public String execute(String goal) {
        StringBuilder memory = new StringBuilder();
        int maxSteps = 5;
        
        for (int i = 0; i < maxSteps; i++) {
            // 规划下一步
            String plan = chatClient.call(buildPlanningPrompt(goal, memory.toString()));
            
            // 解析计划，决定是调用工具还是返回结果
            Action action = parseAction(plan);
            
            if (action.isComplete()) {
                return action.getResult();
            }
            
            // 执行工具调用
            ToolResult result = executeTool(action.getToolName(), action.getParams());
            memory.append("Step ").append(i + 1)
                  .append(": ").append(result).append("\n");
        }
        
        return "达到最大执行步数，任务未完成";
    }
}
```

---

### 7. 如何设计一个支持多模态（文本+图片）的AI应用？

**答案：**

```java
@Service
public class MultimodalService {
    
    @Autowired
    private OpenAiChatClient chatClient;
    
    public String analyzeImage(String imageUrl, String question) {
        // 构建多模态消息
        UserMessage message = new UserMessage(
            question,
            List.of(new Media(MimeTypeUtils.IMAGE_PNG, imageUrl))
        );
        
        return chatClient.call(new Prompt(message));
    }
    
    public String analyzeLocalImage(MultipartFile file, String question) {
        try {
            // 将图片转为Base64
            String base64Image = Base64.getEncoder().encodeToString(file.getBytes());
            String dataUri = "data:image/png;base64," + base64Image;
            
            return analyzeImage(dataUri, question);
        } catch (IOException e) {
            throw new RuntimeException("图片处理失败", e);
        }
    }
}
```

**应用场景：**
- 文档OCR与理解
- 商品图片识别
- 医学影像分析
- 工业质检

---

### 8. 未来Java AI开发的趋势是什么？

**答案：**

**技术趋势：**

1. **模型小型化与本地化**
   - 端侧模型（MobileLLM、Phi系列）
   - Java与ONNX Runtime、TensorFlow Lite深度集成

2. **多Agent协作架构**
   - 复杂任务分解为多个专业Agent
   - 通过消息总线协调执行

3. **RAG向Agentic RAG演进**
   - 检索不再是单次操作
   - Agent可自主决定何时检索、检索什么

4. **AI原生应用架构**
   - 从AI增强现有应用 → AI优先设计应用
   - 向量数据库成为核心基础设施

5. **Java生态完善**
   - Spring AI持续迭代，支持更多模型提供商
   - LangChain4j等Java原生框架成熟
   - GraalVM原生镜像支持AI应用启动优化

**学习建议：**
- 深入理解Transformer架构和注意力机制
- 掌握向量数据库原理和调优
- 关注MCP（Model Context Protocol）等新兴标准
- 实践RAG、Agent等完整项目

---

## 总结

本文整理了工程实践与进阶架构相关的8道面试题，涵盖：

- **工程实践**：限流熔断、多租户、对话记忆、安全过滤
- **进阶架构**：Function Calling、Agent、多模态
- **未来趋势**：模型小型化、多Agent协作、AI原生架构

掌握这些知识点，将帮助你在Java AI开发中构建生产级应用和先进架构。

---

> **系列文章：**
> - [Java AI面试题：基础概念与大模型原理篇](../Java-AI面试题-基础概念与大模型原理篇/)
> - [Java AI面试题：Spring AI框架实战篇](../Java-AI面试题-Spring-AI框架实战篇/)
> - [Java AI面试题：模型集成与量化部署篇](../Java-AI面试题-模型集成与量化部署篇/)
> - [Java AI面试题：模型微调与API集成篇](../Java-AI面试题-模型微调与API集成篇/)
