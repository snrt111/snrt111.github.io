---
title: Java AI开发高频面试题30道（含详细答案解析）
date: 2026-05-18 17:00:00
tags: [Java, AI, 面试题, 大模型, Spring AI, 机器学习, 深度学习]
categories:
  - 面试题
---

# Java AI开发高频面试题30道（含详细答案解析）

随着AI技术的快速发展，Java开发者也需要掌握AI相关的技术栈。本文整理了Java AI开发领域最高频的30道面试题，涵盖大模型集成、向量数据库、RAG架构、模型部署等核心知识点，帮助你在面试中脱颖而出。

---

## 一、基础概念篇

### 1. 什么是RAG（检索增强生成）？它的核心优势是什么？

**答案：**

RAG（Retrieval-Augmented Generation，检索增强生成）是一种将信息检索技术与文本生成技术相结合的AI架构。

**核心流程：**
1. **检索阶段**：将用户查询向量化，从知识库中检索相关文档
2. **增强阶段**：将检索到的上下文与用户问题拼接
3. **生成阶段**：将增强后的提示词输入大模型，生成回答

**核心优势：**
- **解决知识截止问题**：大模型可以访问最新、私域知识
- **减少幻觉**：基于检索到的真实文档生成回答，降低编造概率
- **可溯源**：回答可追溯到具体文档来源
- **成本优化**：无需微调模型即可更新知识

---

### 2. 向量数据库与传统数据库有什么区别？

**答案：**

| 特性 | 传统数据库 | 向量数据库 |
|------|------------|------------|
| 存储内容 | 结构化数据 | 高维向量（Embedding） |
| 查询方式 | 精确匹配（=、>、<） | 相似度搜索（余弦相似度、欧氏距离） |
| 索引结构 | B+树、哈希表 | HNSW、IVF、PQ等ANN索引 |
| 典型应用 | 事务处理、CRUD | 语义搜索、推荐系统、RAG |

**常见向量数据库：** Milvus、Pinecone、Weaviate、Chroma、Redis Vector、PostgreSQL with pgvector

---

### 3. 什么是Embedding（嵌入）？它在AI中的作用是什么？

**答案：**

Embedding是将高维离散数据（如文本、图像）映射到低维连续向量空间的技术。

**核心作用：**
- **语义表示**：将文本转换为固定维度的数值向量
- **语义相似度计算**：通过向量距离衡量语义相似性
- **降维**：将复杂数据转为机器学习可处理的格式

**Java中常用Embedding模型：**
- OpenAI text-embedding-ada-002
- BGE（BAAI General Embedding）
- M3E（Moka Massive Mixed Embedding）
- 通义千问Embedding

---

### 4. 什么是Prompt Engineering（提示工程）？有哪些常用技巧？

**答案：**

Prompt Engineering是通过设计和优化输入提示词，引导大模型生成更准确、有用输出的技术。

**常用技巧：**

1. **角色设定**：明确指定AI角色
   ```
   你是一位资深的Java架构师，请帮我 review 以下代码...
   ```

2. **Few-shot示例**：提供示例让模型学习模式
   ```
   输入：你好 -> 输出：Hello
   输入：世界 -> 输出：World
   输入：苹果 -> 输出：Apple
   ```

3. **Chain-of-Thought（思维链）**：引导模型分步思考
   ```
   请逐步分析这个问题，展示你的思考过程...
   ```

4. **结构化输出**：指定输出格式（JSON、XML等）
   ```
   请以JSON格式返回，包含以下字段：code、message、data
   ```

---

## 二、Spring AI框架篇

### 5. Spring AI是什么？它解决了什么问题？

**答案：**

Spring AI是Spring官方推出的AI应用开发框架，旨在简化Java开发者集成AI能力的复杂度。

**核心解决的问题：**
- **统一抽象**：屏蔽不同AI提供商（OpenAI、Azure、Ollama等）的API差异
- **简化集成**：通过Spring Boot自动配置快速接入AI能力
- **标准化接口**：提供统一的ChatClient、EmbeddingClient接口
- **生态整合**：与Spring生态（Data、Cloud、Security等）无缝集成

**核心组件：**
- `ChatClient`：对话模型交互
- `EmbeddingClient`：文本向量化
- `VectorStore`：向量数据存储
- `Prompt`：提示词管理

---

### 6. Spring AI中ChatClient的核心用法是什么？

**答案：**

```java
@Service
public class ChatService {
    
    private final ChatClient chatClient;
    
    public ChatService(ChatClient.Builder chatClientBuilder) {
        this.chatClient = chatClientBuilder.build();
    }
    
    // 基础对话
    public String chat(String message) {
        return chatClient.prompt()
                .user(message)
                .call()
                .content();
    }
    
    // 带系统提示的对话
    public String chatWithSystem(String message) {
        return chatClient.prompt()
                .system("你是一位Java专家")
                .user(message)
                .call()
                .content();
    }
    
    // 流式响应
    public Flux<String> streamChat(String message) {
        return chatClient.prompt()
                .user(message)
                .stream()
                .content();
    }
    
    // 结构化输出
    public MovieRecommendation getRecommendation(String genre) {
        return chatClient.prompt()
                .user("推荐一部" + genre + "电影")
                .call()
                .entity(MovieRecommendation.class);
    }
}
```

---

### 7. 如何在Spring AI中实现RAG（检索增强生成）？

**答案：**

```java
@Service
public class RAGService {
    
    private final ChatClient chatClient;
    private final VectorStore vectorStore;
        
    简洁版 RAG 实现
    public String ragQuery(String query) {
        // QuestionAnswerAdvisor 会自动完成：
        // 1. 使用 vectorStore 进行相似性检索
        // 2. 将检索到的文档内容注入到系统提示词中
        // 3. 调用 LLM 生成回答
        return chatClient.prompt()
                .user(query)
                .advisors(new QuestionAnswerAdvisor(vectorStore))
                .call()
                .content();
    }

    自定义参数
    public String ragQuery(String query) {
        // 自定义检索参数：TopK=5，相似度阈值=0.7
        return chatClient.prompt()
                .user(query)
                .advisors(new QuestionAnswerAdvisor(
                    vectorStore, 
                    SearchRequest.query(query).withTopK(5).withSimilarityThreshold(0.7)
                ))
                .call()
                .content();
    }

    自定义提示词模板
    public String ragQuery(String query) {
        // 自定义系统提示词模板
        String systemPromptTemplate = """
            你是一个专业的AI助手，请严格基于以下上下文信息回答用户的问题。
            如果上下文中没有相关信息，请直接说"抱歉，我无法从提供的资料中找到答案"。
            
            【上下文信息】
            {question_answer_context}
            
            【回答要求】
            1. 回答要准确、简洁
            2. 不要编造上下文之外的信息
            """;
        
        return chatClient.prompt()
                .user(query)
                .advisors(new QuestionAnswerAdvisor(vectorStore, systemPromptTemplate))
                .call()
                .content();
    }
}
```

**配置向量存储：**
```yaml
spring:
  ai:
    vectorstore:
      pgvector:
        index-type: hnsw
        distance-type: cosine_distance
        dimensions: 1536
```

---

### 8. Spring AI中的Advisor是什么？有哪些常用Advisor？

**答案：**

Advisor是Spring AI提供的AOP风格的拦截器机制，用于在请求处理前后添加额外逻辑。

**常用Advisor：**

| Advisor | 功能 |
|---------|------|
| `QuestionAnswerAdvisor` | 自动实现RAG，注入检索上下文 |
| `MessageChatMemoryAdvisor` | 维护对话历史记忆 |
| `SafeGuardAdvisor` | 内容安全过滤 |
| `RetrievalAugmentationAdvisor` | 检索增强（可自定义检索逻辑） |
| `LoggingAdvisor` | 请求/响应日志记录 |

**自定义Advisor示例：**
```java
public class CustomAdvisor implements CallAdvisor {
    
    @Override
    public AdvisedRequest adviseCall(AdvisedRequest request) {
        // 在请求前修改提示词
        Map<String, Object> context = new HashMap<>(request.context());
        context.put("timestamp", System.currentTimeMillis());
        
        return AdvisedRequest.from(request)
                .withContext(context)
                .build();
    }
    
    @Override
    public int getOrder() {
        return 0; // 执行顺序
    }
}
```

---

## 三、大模型集成篇

### 9. 如何在Java中集成OpenAI API？

**答案：**

**Maven依赖：**
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
```

**配置：**
```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      base-url: https://api.openai.com  # 或使用代理地址
      chat:
        options:
          model: gpt-4
          temperature: 0.7
          max-tokens: 2000
```

**代码使用：**
```java
@Service
public class OpenAIService {
    
    @Autowired
    private OpenAiChatClient chatClient;
    
    public String generateCode(String requirement) {
        return chatClient.call("生成Java代码: " + requirement);
    }
}
```

---

### 10. 什么是Ollama？如何在本地部署大模型？

**答案：**

Ollama是一个开源的本地大模型运行框架，支持在本地机器上轻松运行Llama、Mistral、Qwen等开源模型。

**安装与使用：**

```bash
# 安装Ollama（macOS/Linux）
curl -fsSL https://ollama.com/install.sh | sh

# 拉取模型
ollama pull llama3
ollama pull qwen:14b

# 运行模型
ollama run llama3
```

**Spring AI集成：**
```yaml
spring:
  ai:
    ollama:
      base-url: http://localhost:11434
      chat:
        options:
          model: llama3
          temperature: 0.8
```

**优势：**
- 数据隐私（本地运行，不上传云端）
- 零网络延迟
- 无API调用费用
- 可离线使用

---

### 11. 如何实现多模型路由（根据场景选择不同模型）？

**答案：**

```java
@Component
public class ModelRouter {
    
    private final Map<String, ChatClient> modelRegistry = new HashMap<>();
    
    public ModelRouter(
            OpenAiChatClient openAiClient,
            OllamaChatClient ollamaClient,
            ZhiPuAiChatClient zhipuClient) {
        modelRegistry.put("gpt-4", openAiClient);
        modelRegistry.put("llama3", ollamaClient);
        modelRegistry.put("glm-4", zhipuClient);
    }
    
    public String chat(String modelName, String message) {
        ChatClient client = modelRegistry.get(modelName);
        if (client == null) {
            throw new IllegalArgumentException("Unknown model: " + modelName);
        }
        return client.call(message);
    }
    
    // 智能路由：根据任务复杂度选择模型
    public String smartChat(String message) {
        // 简单任务使用轻量级模型
        if (isSimpleTask(message)) {
            return modelRegistry.get("llama3").call(message);
        }
        // 复杂任务使用大模型
        return modelRegistry.get("gpt-4").call(message);
    }
}
```
兼容OpenAi模式的多模型配置
```
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.model.ChatModel;
import org.springframework.ai.openai.OpenAiChatModel;
import org.springframework.ai.openai.OpenAiChatOptions;
import org.springframework.ai.openai.api.OpenAiApi;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MultiModelConfig {

    /**
     * 为 OpenAI 兼容的模型（如 DeepSeek）手动创建 ChatClient Bean
     *
     * @param deepSeekApiKey 从配置文件中注入的 DeepSeek API Key
     * @param deepSeekBaseUrl 从配置文件中注入的 DeepSeek Base URL
     * @return 专用于 DeepSeek 的 ChatClient 实例
     */
    @Bean
    public ChatClient deepSeekChatClient(
            @Value("${deepseek.api-key}") String deepSeekApiKey,
            @Value("${deepseek.base-url}") String deepSeekBaseUrl) {

        // 1. 构建 OpenAiApi 实例，这是与 OpenAI 兼容服务通信的基础
        //    构造参数需要：baseUrl, apiKey, restClientBuilder, webClientBuilder
        OpenAiApi openAiApi = new OpenAiApi(deepSeekBaseUrl, deepSeekApiKey, null, null);

        // 2. 配置该模型特有的选项，例如模型名称
        OpenAiChatOptions chatOptions = OpenAiChatOptions.builder()
                .model("deepseek-chat") // 指定模型名称
                .temperature(0.7)       // 可选：设置创造力
                .build();

        // 3. 手动构建 ChatModel
        ChatModel deepSeekChatModel = OpenAiChatModel.builder()
                .openAiApi(openAiApi)
                .defaultOptions(chatOptions)
                .build();

        // 4. 基于 ChatModel 创建 ChatClient
        return ChatClient.builder(deepSeekChatModel).build();
    }
}
```

配置完成后，在需要的地方，使用 @Qualifier 注解指定要注入哪个 ChatClient Bean，就可以像使用普通 ChatClient 一样调用它了。
```
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

@Service
public class MultiModelService {

    private final ChatClient openAiChatClient;
    private final ChatClient deepSeekChatClient;

    // 使用 @Qualifier 明确区分并注入不同的 ChatClient
    public MultiModelService(
            @Qualifier("deepSeekChatClient") ChatClient deepSeekChatClient,
            @Qualifier("openAiChatClient") ChatClient openAiChatClient) {
        this.deepSeekChatClient = deepSeekChatClient;
        this.openAiChatClient = openAiChatClient;
    }

    public String chatWithDeepSeek(String userMessage) {
        // 直接调用 DeepSeek 的 ChatClient
        return deepSeekChatClient.prompt()
                .user(userMessage)
                .call()
                .content();
    }

    public String chatWithOpenAi(String userMessage) {
        // 直接调用 OpenAI 的 ChatClient
        return openAiChatClient.prompt()
                .user(userMessage)
                .call()
                .content();
    }
}
```

---

### 12. 如何处理大模型的Token限制问题？

**答案：**

**常见策略：**

1. **文本截断**
```java
public String truncateText(String text, int maxTokens) {
    // 近似估算：1 token ≈ 0.75 个英文单词
    int maxChars = maxTokens * 4;
    if (text.length() <= maxChars) {
        return text;
    }
    return text.substring(0, maxChars) + "...";
}
```

2. **分块处理（Chunking）**
```java
public List<String> splitIntoChunks(String text, int chunkSize) {
    List<String> chunks = new ArrayList<>();
    for (int i = 0; i < text.length(); i += chunkSize) {
        chunks.add(text.substring(i, Math.min(i + chunkSize, text.length())));
    }
    return chunks;
}
```

3. **Map-Reduce模式**
```java
public String mapReduceSummarize(String longText) {
    // Map阶段：分块摘要
    List<String> chunks = splitIntoChunks(longText, 3000);
    List<String> summaries = chunks.stream()
        .map(chunk -> chatClient.call("请摘要以下内容：" + chunk))
        .collect(Collectors.toList());
    
    // Reduce阶段：合并摘要
    String combined = String.join("\n", summaries);
    return chatClient.call("请综合以下摘要：" + combined);
}
```

---

## 四、向量数据库与检索篇

### 13. 如何在Spring AI中使用Redis作为向量数据库？

**答案：**

**依赖：**
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-redis-store-spring-boot-starter</artifactId>
</dependency>
```

**配置：**
```yaml
spring:
  ai:
    vectorstore:
      redis:
        index: document-index
        prefix: doc:
        uri: redis://localhost:6379
  data:
    redis:
      host: localhost
      port: 6379
```

**使用：**
```java
@Service
public class DocumentService {
    
    @Autowired
    private VectorStore vectorStore;
    
    @Autowired
    private EmbeddingClient embeddingClient;
    
    // 添加文档
    public void addDocument(String content) {
        Document doc = new Document(content);
        vectorStore.add(List.of(doc));
    }
    
    // 相似度搜索
    public List<Document> search(String query, int topK) {
        return vectorStore.similaritySearch(
            SearchRequest.query(query).withTopK(topK)
        );
    }
}
```

---

### 14. 什么是HNSW索引？它的优缺点是什么？

**答案：**

HNSW（Hierarchical Navigable Small World，分层可导航小世界）是一种基于图的近似最近邻（ANN）搜索算法。

**原理：**
- 构建多层图结构，每层是上一层的子集
- 搜索时从顶层开始，逐层向下精确定位
- 通过贪心算法在图中导航找到最近邻

**优点：**
- 查询速度快（O(log n)）
- 召回率高（通常>95%）
- 支持增量添加向量

**缺点：**
- 内存占用较高
- 构建索引较慢
- 不适合频繁删除的场景

**Java配置示例：**
```java
@Bean
public VectorStore vectorStore(EmbeddingClient embeddingClient) {
    return PgVectorStore.builder(embeddingClient)
            .indexType(PgVectorStore.PgIndexType.HNSW)
            .distanceType(PgVectorStore.PgDistanceType.COSINE_DISTANCE)
            .dimensions(1536)
            .build();
}
```

---

### 15. 文档分块（Chunking）有哪些策略？如何选择？

**答案：**

| 策略 | 适用场景 | 优缺点 |
|------|----------|--------|
| **固定长度** | 通用场景 | 简单，可能切断语义 |
| **按段落** | 文章、报告 | 保持段落完整性 |
| **按句子** | 短文本 | 粒度细，上下文可能丢失 |
| **重叠分块** | 需要上下文的场景 | 保留边界信息，有冗余 |
| **语义分块** | 长文档 | 按主题分割，复杂度高 |

**重叠分块实现：**
```java
public List<String> overlappingChunks(String text, int chunkSize, int overlap) {
    List<String> chunks = new ArrayList<>();
    int step = chunkSize - overlap;
    
    for (int i = 0; i < text.length(); i += step) {
        int end = Math.min(i + chunkSize, text.length());
        chunks.add(text.substring(i, end));
        if (end == text.length()) break;
    }
    return chunks;
}
```

---

## 五、模型部署与优化篇

### 16. 什么是模型量化（Quantization）？有什么作用？

**答案：**

模型量化是将模型权重从高精度（如FP32）转换为低精度（如INT8、INT4）表示的技术。

**常见量化类型：**

| 类型 | 精度 | 模型大小 | 精度损失 |
|------|------|----------|----------|
| FP32 | 32位浮点 | 100% | 基准 |
| FP16 | 16位浮点 | 50% | 极小 |
| INT8 | 8位整数 | 25% | 较小 |
| INT4 | 4位整数 | 12.5% | 可接受 |

**作用：**
- **减少内存占用**：模型体积缩小2-8倍
- **加速推理**：低精度运算更快
- **降低功耗**：适合边缘设备部署

**Java集成量化模型（Ollama）：**
```bash
# 运行量化版本的Llama3
ollama run llama3:8b-instruct-q4_0
```

---

### 17. 如何在Java中实现流式（Streaming）响应？

**答案：**

```java
@RestController
public class ChatController {
    
    @Autowired
    private ChatClient chatClient;
    
    @GetMapping(value = "/chat/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<String>> streamChat(@RequestParam String message) {
        return chatClient.prompt()
                .user(message)
                .stream()
                .content()
                .map(content -> ServerSentEvent.builder(content).build());
    }
}
```

**前端接收（JavaScript）：**
```javascript
const eventSource = new EventSource('/chat/stream?message=你好');

eventSource.onmessage = (event) => {
    console.log('收到:', event.data);
    appendToChat(event.data);
};

eventSource.onerror = () => {
    eventSource.close();
};
```

**优势：**
- 提升用户体验（逐字显示）
- 减少首字节等待时间
- 适合长文本生成场景

---

### 18. 如何评估RAG系统的检索质量？

**答案：**

**常用评估指标：**

| 指标 | 说明 | 计算方式 |
|------|------|----------|
| **Recall@K** | 前K个结果中相关文档比例 | 相关文档数 / 总相关文档数 |
| **Precision@K** | 前K个结果的准确率 | 相关文档数 / K |
| **MRR** | 平均倒数排名 | 1 / 首个相关文档排名 |
| **NDCG** | 归一化折损累积增益 | 考虑文档相关度和位置 |

**Java评估实现：**
```java
@Component
public class RAGEvaluator {
    
    public double calculateRecallAtK(List<Document> retrieved, 
                                      Set<String> relevantDocs, int k) {
        long relevantRetrieved = retrieved.stream()
            .limit(k)
            .filter(doc -> relevantDocs.contains(doc.getId()))
            .count();
        return (double) relevantRetrieved / relevantDocs.size();
    }
    
    public double calculateMRR(List<List<Document>> queriesResults,
                                List<Set<String>> relevantDocsList) {
        double sumRR = 0;
        for (int i = 0; i < queriesResults.size(); i++) {
            List<Document> results = queriesResults.get(i);
            Set<String> relevant = relevantDocsList.get(i);
            
            for (int rank = 0; rank < results.size(); rank++) {
                if (relevant.contains(results.get(rank).getId())) {
                    sumRR += 1.0 / (rank + 1);
                    break;
                }
            }
        }
        return sumRR / queriesResults.size();
    }
}
```

---

## 六、工程实践篇

### 19. 如何实现AI应用的限流和熔断？

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

### 20. 如何设计一个支持多租户的AI知识库系统？

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

### 21. 如何实现AI对话的历史记忆功能？

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

### 22. 如何处理AI生成内容的安全性问题？

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

## 七、进阶架构篇

### 23. 什么是Function Calling（函数调用）？如何在Java中实现？

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

### 24. 什么是Agent（智能体）？与传统AI应用有什么区别？

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

### 25. 如何设计一个支持多模态（文本+图片）的AI应用？

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

### 26. 什么是Embedding模型的微调（Fine-tuning）？何时需要？

**答案：**

Embedding微调是在特定领域数据上继续训练预训练Embedding模型，使其更好地理解领域语义。

**何时需要微调：**
- 领域术语与通用语义差异大（医疗、法律、金融）
- 检索效果不佳，相似度计算不准确
- 有充足的领域标注数据

**微调流程：**
```python
# 使用sentence-transformers微调（示意）
from sentence_transformers import SentenceTransformer, InputExample, losses

model = SentenceTransformer('BAAI/bge-large-zh')

train_examples = [
    InputExample(texts=["查询1", "相关文档1"], label=1.0),
    InputExample(texts=["查询1", "不相关文档"], label=0.0),
]

train_loss = losses.CosineSimilarityLoss(model)
model.fit(train_objectives=[(train_examples, train_loss)], epochs=3)
```

**Java中使用微调后的模型：**
```java
// 通过Ollama加载本地微调模型
ollama run my-custom-embedding:latest
```

---

### 27. 如何优化RAG系统的检索效果？

**答案：**

**优化策略清单：**

1. **文档预处理优化**
   - 清洗HTML标签、特殊字符
   - 提取结构化信息（标题、表格、列表）
   - 添加文档元数据（来源、时间、作者）

2. **分块策略优化**
   ```java
   // 按语义段落分块
   public List<String> semanticChunk(String text) {
       // 使用标题、段落分隔符分割
       return Arrays.asList(text.split("(?=\\n#{1,3} )"));
   }
   ```

3. **混合检索**
   ```java
   public List<Document> hybridSearch(String query) {
       // 向量检索
       List<Document> vectorResults = vectorStore.similaritySearch(query);
       // 关键词检索
       List<Document> keywordResults = keywordSearch(query);
       // 融合排序（RRF算法）
       return reciprocalRankFusion(vectorResults, keywordResults);
   }
   ```

4. **查询重写**
   ```java
   public String rewriteQuery(String originalQuery) {
       String prompt = "将以下查询扩展为3个相关查询，用于文档检索：" + originalQuery;
       return chatClient.call(prompt);
   }
   ```

5. **重排序（Rerank）**
   ```java
   // 使用Cross-Encoder模型对初步检索结果重排序
   public List<Document> rerank(String query, List<Document> candidates) {
       // 调用重排序模型
       return reranker.rerank(query, candidates);
   }
   ```

---

### 28. 如何实现AI应用的A/B测试？

**答案：**

```java
@Service
public class AITestService {
    
    private final ChatClient modelA;  // GPT-4
    private final ChatClient modelB;  // Claude
    
    @Autowired
    private ExperimentTracker tracker;
    
    public String chat(String userId, String message) {
        // 根据用户ID分流
        String variant = assignVariant(userId);
        
        long startTime = System.currentTimeMillis();
        String response;
        boolean success = true;
        
        try {
            if ("A".equals(variant)) {
                response = modelA.call(message);
            } else {
                response = modelB.call(message);
            }
        } catch (Exception e) {
            response = "服务异常";
            success = false;
        }
        
        // 记录实验数据
        long latency = System.currentTimeMillis() - startTime;
        tracker.record(userId, variant, latency, success, response.length());
        
        return response;
    }
    
    private String assignVariant(String userId) {
        // 一致性哈希，确保同一用户始终分配到同一组
        int hash = userId.hashCode();
        return hash % 2 == 0 ? "A" : "B";
    }
}
```

**评估指标：**
- 响应延迟（Latency）
- 输出长度
- 用户满意度（人工标注）
- 业务转化率

---

### 29. 如何监控AI应用的性能和成本？

**答案：**

```java
@Component
public class AIMetricsCollector {
    
    private final MeterRegistry meterRegistry;
    
    public void recordMetrics(String model, long latency, int inputTokens, 
                              int outputTokens, boolean success) {
        // 延迟指标
        meterRegistry.timer("ai.request.latency", "model", model)
                     .record(latency, TimeUnit.MILLISECONDS);
        
        // Token使用量
        meterRegistry.counter("ai.tokens.input", "model", model)
                     .increment(inputTokens);
        meterRegistry.counter("ai.tokens.output", "model", model)
                     .increment(outputTokens);
        
        // 成功率
        meterRegistry.counter("ai.request.total", "model", model).increment();
        if (success) {
            meterRegistry.counter("ai.request.success", "model", model).increment();
        }
        
        // 估算成本（以OpenAI GPT-4为例）
        double cost = (inputTokens * 0.03 + outputTokens * 0.06) / 1000;
        meterRegistry.counter("ai.cost", "model", model).increment(cost);
    }
}
```

**关键监控指标：**

| 指标类型 | 具体指标 | 告警阈值建议 |
|----------|----------|--------------|
| 性能 | P99延迟 | > 5s |
| 性能 | 错误率 | > 1% |
| 成本 | 单次请求成本 | 持续增长 |
| 质量 | 用户反馈差评率 | > 5% |
| 资源 | Token消耗量/日 | 超预算 |

---

### 30. 未来Java AI开发的趋势是什么？

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

本文整理了Java AI开发领域30道高频面试题，涵盖：

- **基础概念**：RAG、Embedding、Prompt Engineering
- **Spring AI**：ChatClient、Advisor、向量存储
- **模型集成**：OpenAI、Ollama、多模型路由
- **工程实践**：限流熔断、多租户、安全过滤
- **进阶架构**：Function Calling、Agent、多模态

掌握这些知识点，将帮助你在Java AI开发领域建立扎实的技术基础，应对各类面试挑战。

---

> **推荐阅读：**
> - [Spring AI官方文档](https://docs.spring.io/spring-ai/reference/)
> - [LangChain4j GitHub](https://github.com/langchain4j/langchain4j)
> - [向量数据库选型指南](https://milvus.io/docs)
