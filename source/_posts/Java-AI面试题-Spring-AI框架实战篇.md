---
title: Java AI面试题：Spring AI框架实战篇（4道）
date: 2026-05-18 17:05:00
tags: [Java, AI, 面试题, Spring AI, Spring Boot, RAG, ChatClient]
categories:
  - 面试题
---

# Java AI面试题：Spring AI框架实战篇（4道）

本文整理了Spring AI框架相关的4道面试题，涵盖Spring AI核心概念、ChatClient用法、RAG实现、Advisor机制等实战内容，帮助你掌握Java AI应用开发的核心技能。

---

## Spring AI框架篇

### 1. Spring AI是什么？它解决了什么问题？

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

### 2. Spring AI中ChatClient的核心用法是什么？

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

### 3. 如何在Spring AI中实现RAG（检索增强生成）？

**答案：**

```java
@Service
public class RAGService {
    
    private final ChatClient chatClient;
    private final VectorStore vectorStore;
        
    // 简洁版 RAG 实现
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

    // 自定义参数
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

    // 自定义提示词模板
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

### 4. Spring AI中的Advisor是什么？有哪些常用Advisor？

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

## 总结

本文整理了Spring AI框架相关的4道面试题，涵盖：

- **Spring AI核心概念**：框架定位、解决的问题
- **ChatClient用法**：基础对话、系统提示、流式响应、结构化输出
- **RAG实现**：使用QuestionAnswerAdvisor快速构建RAG应用
- **Advisor机制**：常用Advisor介绍及自定义Advisor开发

掌握Spring AI框架，可以让你在Java生态中快速构建AI应用。

---

> **系列文章：**
> - [Java AI面试题：基础概念与大模型原理篇](../Java-AI面试题-基础概念与大模型原理篇/)
> - [Java AI面试题：模型集成与量化部署篇](../Java-AI面试题-模型集成与量化部署篇/)
> - [Java AI面试题：模型微调与API集成篇](../Java-AI面试题-模型微调与API集成篇/)
> - [Java AI面试题：工程实践与进阶架构篇](../Java-AI面试题-工程实践与进阶架构篇/)
