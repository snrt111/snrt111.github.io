---
title: Spring AI Alibaba搭建知识库详细教程
date: 2026-03-24 14:00:00
tags:
---

## 前言

Spring AI Alibaba是阿里云推出的面向Java开发者的AI应用开发框架，它基于Spring AI构建，提供了与阿里云大模型（通义千问）的无缝集成。通过Spring AI Alibaba，开发者可以快速构建RAG（检索增强生成）知识库系统，实现智能问答、文档处理等功能。

本教程将手把手教您从零开始搭建一个完整的Spring AI Alibaba知识库系统，包括环境准备、项目搭建、向量数据库配置、文档处理、检索问答等全流程。

## 目录

<!-- toc -->

- [前言](#前言)
- [什么是RAG](#什么是rag)
- [技术栈介绍](#技术栈介绍)
- [环境准备](#环境准备)
- [项目搭建](#项目搭建)
- [向量数据库配置](#向量数据库配置)
- [文档处理与存储](#文档处理与存储)
- [检索与问答](#检索与问答)
- [高级功能](#高级功能)
- [完整示例代码](#完整示例代码)
- [常见问题](#常见问题)
- [结语](#结语)

<!-- more -->

## 什么是RAG

### 1. RAG概念

RAG（Retrieval-Augmented Generation，检索增强生成）是一种结合了信息检索和大语言模型生成的技术架构。

**简单理解**：让大模型"开卷考试"

- **传统大模型**：闭卷答题，只能用训练时学到的知识
- **RAG模式**：先从知识库中检索相关内容，再把内容"喂"给大模型，让它基于真实资料回答

### 2. RAG工作流程

```
用户提问 → 向量化查询 → 向量检索 → 找到相关文档 → 注入Prompt → 大模型生成答案
```

### 3. RAG的优势

1. **知识实时更新**：无需重新训练模型，只需更新知识库
2. **减少幻觉**：基于真实文档回答，降低胡说八道的概率
3. **可追溯性**：可以知道答案来源于哪些文档
4. **成本更低**：比微调大模型成本低得多

## 技术栈介绍

### 1. Spring AI Alibaba

Spring AI Alibaba是阿里云基于Spring AI开发的框架，主要特性：

- 支持阿里云通义千问大模型
- 提供统一的AI模型调用接口
- 支持Prompt模板和变量替换
- 内置RAG支持
- 支持Function Calling

### 2. 向量数据库

本教程使用**PostgreSQL + pgvector**作为向量数据库：

- **PostgreSQL**：关系型数据库，稳定可靠
- **pgvector**：PostgreSQL的向量扩展插件
- 支持向量相似度搜索
- 免费开源

### 3. Embedding模型

使用阿里云的Embedding服务：

- 将文本转换为向量（768维或1536维）
- 支持中文语义理解
- 与通义千问模型协同工作

## 环境准备

### 1. 开发环境要求

- **JDK**：17或更高版本
- **Maven**：3.8+ 或 **Gradle**：7.5+
- **Spring Boot**：3.2+
- **PostgreSQL**：14+（带pgvector扩展）

### 2. 阿里云账号准备

#### 2.1 开通百炼大模型服务

1. 访问阿里云百炼控制台：https://bailian.console.aliyun.com/
2. 开通服务（有免费额度）
3. 创建API Key

#### 2.2 获取API Key

```
登录阿里云 → 百炼控制台 → API Key管理 → 创建API Key
```

记录下API Key，后续配置需要用到。

### 3. 安装PostgreSQL和pgvector

#### 3.1 Docker方式安装（推荐）

```bash
# 拉取带pgvector的PostgreSQL镜像
docker pull ankane/pgvector:latest

# 运行容器
docker run -d \
  --name pgvector \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=123456 \
  -e POSTGRES_DB=ragdb \
  -p 5432:5432 \
  -v pgvector_data:/var/lib/postgresql/data \
  ankane/pgvector:latest
```

#### 3.2 验证安装

```bash
# 进入容器
docker exec -it pgvector psql -U postgres -d ragdb

# 在psql中执行
CREATE EXTENSION IF NOT EXISTS vector;

# 查看版本
SELECT * FROM pg_extension WHERE extname = 'vector';
```

看到vector扩展信息说明安装成功。

## 项目搭建

### 1. 创建Spring Boot项目

使用Spring Initializr创建项目：https://start.spring.io/

**配置选项**：
- **Project**：Maven
- **Language**：Java
- **Spring Boot**：3.2.0
- **Group**：com.example
- **Artifact**：rag-demo
- **Name**：rag-demo
- **Package name**：com.example.ragdemo
- **Java**：17

**依赖选择**：
- Spring Web
- Spring Data JPA
- PostgreSQL Driver
- Lombok（可选）

### 2. 添加Spring AI Alibaba依赖

在`pom.xml`中添加依赖：

```xml
<properties>
    <java.version>17</java.version>
    <spring-ai-alibaba.version>1.0.0-M3</spring-ai-alibaba.version>
</properties>

<dependencies>
    <!-- Spring Boot Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    
    <!-- PostgreSQL -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
    
    <!-- Spring AI Alibaba -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-ai-alibaba-starter</artifactId>
        <version>${spring-ai-alibaba.version}</version>
    </dependency>
    
    <!-- 向量数据库支持 -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-pgvector-store</artifactId>
        <version>1.0.0-M3</version>
    </dependency>
    
    <!-- 文档解析 -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-tika-document-reader</artifactId>
        <version>1.0.0-M3</version>
    </dependency>
    
    <!-- Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    
    <!-- 测试 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<repositories>
    <repository>
        <id>spring-milestones</id>
        <name>Spring Milestones</name>
        <url>https://repo.spring.io/milestone</url>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
    </repository>
</repositories>
```

### 3. 配置application.yml

```yaml
server:
  port: 8080

spring:
  application:
    name: rag-demo
  
  # 数据库配置
  datasource:
    url: jdbc:postgresql://localhost:5432/ragdb
    username: postgres
    password: 123456
    driver-class-name: org.postgresql.Driver
  
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
  
  # Spring AI Alibaba配置
  ai:
    dashscope:
      api-key: ${DASHSCOPE_API_KEY:your-api-key-here}
      chat:
        options:
          model: qwen-turbo  # 通义千问模型
      embedding:
        options:
          model: text-embedding-v2  # Embedding模型

# 日志配置
logging:
  level:
    com.alibaba.cloud.ai: DEBUG
    org.springframework.ai: DEBUG
```

**注意**：将`${DASHSCOPE_API_KEY}`替换为您的实际API Key，或者设置环境变量。

## 向量数据库配置

### 1. 创建向量存储配置类

```java
package com.example.ragdemo.config;

import org.springframework.ai.embedding.EmbeddingModel;
import org.springframework.ai.vectorstore.PgVectorStore;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.jdbc.core.JdbcTemplate;

@Configuration
public class VectorStoreConfig {

    @Bean
    public VectorStore vectorStore(JdbcTemplate jdbcTemplate, EmbeddingModel embeddingModel) {
        return PgVectorStore.builder(jdbcTemplate, embeddingModel)
                .dimensions(1536)  // 向量维度
                .distanceType(PgVectorStore.PgDistanceType.COSINE_DISTANCE)  // 余弦相似度
                .initializeSchema(true)  // 自动初始化表结构
                .build();
    }
}
```

### 2. 创建文档实体类

```java
package com.example.ragdemo.entity;

import jakarta.persistence.*;
import lombok.Data;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

import java.time.LocalDateTime;

@Entity
@Table(name = "documents")
@Data
public class Document {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "doc_id", nullable = false, unique = true)
    private String docId;
    
    @Column(name = "title", nullable = false)
    private String title;
    
    @Column(name = "content", columnDefinition = "TEXT")
    private String content;
    
    @Column(name = "source")
    private String source;
    
    @Column(name = "doc_type")
    private String docType;
    
    @CreationTimestamp
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    @UpdateTimestamp
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
}
```

### 3. 创建Repository接口

```java
package com.example.ragdemo.repository;

import com.example.ragdemo.entity.Document;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public interface DocumentRepository extends JpaRepository<Document, Long> {
    
    Optional<Document> findByDocId(String docId);
    
    void deleteByDocId(String docId);
}
```

## 文档处理与存储

### 1. 创建文档服务类

```java
package com.example.ragdemo.service;

import com.example.ragdemo.entity.Document;
import com.example.ragdemo.repository.DocumentRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.ai.document.DocumentReader;
import org.springframework.ai.reader.tika.TikaDocumentReader;
import org.springframework.ai.transformer.splitter.TokenTextSplitter;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;
import java.util.UUID;

@Service
@RequiredArgsConstructor
@Slf4j
public class DocumentService {

    private final DocumentRepository documentRepository;
    private final VectorStore vectorStore;
    
    /**
     * 上传并处理文档
     */
    @Transactional
    public Document uploadDocument(MultipartFile file, String title) throws IOException {
        // 生成文档ID
        String docId = UUID.randomUUID().toString();
        
        // 保存文件到临时目录
        Path tempFile = Files.createTempFile("doc_", "_" + file.getOriginalFilename());
        file.transferTo(tempFile.toFile());
        
        try {
            // 读取文档内容
            String content = extractContent(tempFile);
            
            // 保存文档信息到数据库
            Document document = new Document();
            document.setDocId(docId);
            document.setTitle(title);
            document.setContent(content);
            document.setSource(file.getOriginalFilename());
            document.setDocType(getFileExtension(file.getOriginalFilename()));
            
            documentRepository.save(document);
            
            // 向量化并存储到向量数据库
            vectorizeAndStore(docId, title, content);
            
            log.info("文档上传成功: {}", title);
            return document;
            
        } finally {
            // 清理临时文件
            Files.deleteIfExists(tempFile);
        }
    }
    
    /**
     * 从文本内容创建文档
     */
    @Transactional
    public Document createDocumentFromText(String title, String content, String source) {
        String docId = UUID.randomUUID().toString();
        
        Document document = new Document();
        document.setDocId(docId);
        document.setTitle(title);
        document.setContent(content);
        document.setSource(source);
        document.setDocType("TEXT");
        
        documentRepository.save(document);
        
        // 向量化存储
        vectorizeAndStore(docId, title, content);
        
        log.info("文本文档创建成功: {}", title);
        return document;
    }
    
    /**
     * 提取文档内容
     */
    private String extractContent(Path filePath) {
        try {
            DocumentReader reader = new TikaDocumentReader(filePath.toUri().toString());
            List<org.springframework.ai.document.Document> documents = reader.get();
            
            StringBuilder content = new StringBuilder();
            for (org.springframework.ai.document.Document doc : documents) {
                content.append(doc.getContent()).append("\n");
            }
            
            return content.toString();
        } catch (Exception e) {
            log.error("文档解析失败: {}", e.getMessage());
            throw new RuntimeException("文档解析失败", e);
        }
    }
    
    /**
     * 向量化并存储
     */
    private void vectorizeAndStore(String docId, String title, String content) {
        // 使用TokenTextSplitter分割长文本
        TokenTextSplitter splitter = new TokenTextSplitter(500, 100, 50, 1000, true);
        
        List<org.springframework.ai.document.Document> chunks = splitter.split(
            new org.springframework.ai.document.Document(content)
        );
        
        // 为每个分块添加元数据
        for (int i = 0; i < chunks.size(); i++) {
            org.springframework.ai.document.Document chunk = chunks.get(i);
            chunk.getMetadata().put("doc_id", docId);
            chunk.getMetadata().put("title", title);
            chunk.getMetadata().put("chunk_index", i);
            chunk.getMetadata().put("chunk_count", chunks.size());
        }
        
        // 存储到向量数据库
        vectorStore.add(chunks);
        
        log.info("文档向量化完成: {}, 分块数: {}", title, chunks.size());
    }
    
    /**
     * 删除文档
     */
    @Transactional
    public void deleteDocument(String docId) {
        documentRepository.deleteByDocId(docId);
        // 注意：向量数据库中的数据也需要清理
        log.info("文档删除成功: {}", docId);
    }
    
    /**
     * 获取所有文档
     */
    public List<Document> getAllDocuments() {
        return documentRepository.findAll();
    }
    
    /**
     * 获取文件扩展名
     */
    private String getFileExtension(String filename) {
        if (filename == null || filename.lastIndexOf(".") == -1) {
            return "UNKNOWN";
        }
        return filename.substring(filename.lastIndexOf(".") + 1).toUpperCase();
    }
}
```

### 2. 创建文档上传Controller

```java
package com.example.ragdemo.controller;

import com.example.ragdemo.entity.Document;
import com.example.ragdemo.service.DocumentService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/documents")
@RequiredArgsConstructor
@CrossOrigin(origins = "*")
public class DocumentController {

    private final DocumentService documentService;
    
    /**
     * 上传文档
     */
    @PostMapping("/upload")
    public ResponseEntity<?> uploadDocument(
            @RequestParam("file") MultipartFile file,
            @RequestParam("title") String title) {
        try {
            Document document = documentService.uploadDocument(file, title);
            return ResponseEntity.ok(document);
        } catch (IOException e) {
            return ResponseEntity.badRequest().body("文件上传失败: " + e.getMessage());
        }
    }
    
    /**
     * 从文本创建文档
     */
    @PostMapping("/text")
    public ResponseEntity<?> createFromText(
            @RequestParam("title") String title,
            @RequestParam("content") String content,
            @RequestParam(value = "source", defaultValue = "manual") String source) {
        Document document = documentService.createDocumentFromText(title, content, source);
        return ResponseEntity.ok(document);
    }
    
    /**
     * 获取所有文档
     */
    @GetMapping
    public ResponseEntity<List<Document>> getAllDocuments() {
        return ResponseEntity.ok(documentService.getAllDocuments());
    }
    
    /**
     * 删除文档
     */
    @DeleteMapping("/{docId}")
    public ResponseEntity<?> deleteDocument(@PathVariable String docId) {
        documentService.deleteDocument(docId);
        Map<String, String> response = new HashMap<>();
        response.put("message", "文档删除成功");
        response.put("docId", docId);
        return ResponseEntity.ok(response);
    }
}
```

## 检索与问答

### 1. 创建RAG服务类

```java
package com.example.ragdemo.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.ChatResponse;
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.messages.SystemMessage;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.prompt.SystemPromptTemplate;
import org.springframework.ai.document.Document;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
@Slf4j
public class RagService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;
    
    // RAG系统提示词模板
    private static final String RAG_SYSTEM_TEMPLATE = """
        你是一个专业的AI助手，基于以下检索到的信息来回答用户的问题。
        
        检索到的相关信息：
        {documents}
        
        回答要求：
        1. 基于上述检索到的信息回答问题
        2. 如果检索信息不足以回答问题，请明确说明
        3. 保持回答简洁准确
        4. 可以引用相关文档的标题作为参考
        """;
    
    /**
     * RAG问答
     */
    public String chat(String question) {
        // 1. 检索相关文档
        List<Document> relevantDocs = retrieveRelevantDocuments(question);
        
        // 2. 构建上下文
        String context = buildContext(relevantDocs);
        
        // 3. 构建Prompt
        Message systemMessage = new SystemPromptTemplate(RAG_SYSTEM_TEMPLATE)
                .createMessage(Map.of("documents", context));
        
        Message userMessage = new UserMessage(question);
        
        Prompt prompt = new Prompt(List.of(systemMessage, userMessage));
        
        // 4. 调用大模型
        ChatResponse response = chatClient.call(prompt);
        
        return response.getResult().getOutput().getContent();
    }
    
    /**
     * RAG问答（带引用）
     */
    public RagResponse chatWithReferences(String question) {
        // 1. 检索相关文档
        List<Document> relevantDocs = retrieveRelevantDocuments(question);
        
        // 2. 构建上下文
        String context = buildContext(relevantDocs);
        
        // 3. 构建Prompt
        Message systemMessage = new SystemPromptTemplate(RAG_SYSTEM_TEMPLATE)
                .createMessage(Map.of("documents", context));
        
        Message userMessage = new UserMessage(question);
        
        Prompt prompt = new Prompt(List.of(systemMessage, userMessage));
        
        // 4. 调用大模型
        ChatResponse response = chatClient.call(prompt);
        String answer = response.getResult().getOutput().getContent();
        
        // 5. 提取引用信息
        List<String> references = extractReferences(relevantDocs);
        
        return new RagResponse(answer, references);
    }
    
    /**
     * 检索相关文档
     */
    private List<Document> retrieveRelevantDocuments(String query) {
        SearchRequest searchRequest = SearchRequest.query(query)
                .withTopK(5)  // 返回最相关的5个文档
                .withSimilarityThreshold(0.7);  // 相似度阈值
        
        List<Document> documents = vectorStore.similaritySearch(searchRequest);
        
        log.info("检索到 {} 个相关文档", documents.size());
        return documents;
    }
    
    /**
     * 构建上下文
     */
    private String buildContext(List<Document> documents) {
        if (documents.isEmpty()) {
            return "未检索到相关文档。";
        }
        
        StringBuilder context = new StringBuilder();
        for (int i = 0; i < documents.size(); i++) {
            Document doc = documents.get(i);
            context.append("\n--- 文档 ").append(i + 1).append(" ---\n");
            context.append("标题: ").append(doc.getMetadata().getOrDefault("title", "未知")).append("\n");
            context.append("内容: ").append(doc.getContent()).append("\n");
        }
        
        return context.toString();
    }
    
    /**
     * 提取引用信息
     */
    private List<String> extractReferences(List<Document> documents) {
        return documents.stream()
                .map(doc -> (String) doc.getMetadata().getOrDefault("title", "未知文档"))
                .distinct()
                .collect(Collectors.toList());
    }
    
    /**
     * RAG响应对象
     */
    public record RagResponse(String answer, List<String> references) {}
}
```

### 2. 创建问答Controller

```java
package com.example.ragdemo.controller;

import com.example.ragdemo.service.RagService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.Map;

@RestController
@RequestMapping("/api/chat")
@RequiredArgsConstructor
@CrossOrigin(origins = "*")
public class ChatController {

    private final RagService ragService;
    
    /**
     * 简单问答
     */
    @PostMapping
    public ResponseEntity<Map<String, String>> chat(@RequestBody Map<String, String> request) {
        String question = request.get("question");
        if (question == null || question.trim().isEmpty()) {
            return ResponseEntity.badRequest().body(Map.of("error", "问题不能为空"));
        }
        
        String answer = ragService.chat(question);
        
        Map<String, String> response = new HashMap<>();
        response.put("question", question);
        response.put("answer", answer);
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * 带引用的问答
     */
    @PostMapping("/with-references")
    public ResponseEntity<?> chatWithReferences(@RequestBody Map<String, String> request) {
        String question = request.get("question");
        if (question == null || question.trim().isEmpty()) {
            return ResponseEntity.badRequest().body(Map.of("error", "问题不能为空"));
        }
        
        RagService.RagResponse ragResponse = ragService.chatWithReferences(question);
        
        Map<String, Object> response = new HashMap<>();
        response.put("question", question);
        response.put("answer", ragResponse.answer());
        response.put("references", ragResponse.references());
        
        return ResponseEntity.ok(response);
    }
}
```

## 高级功能

### 1. 添加文档相似度搜索API

```java
package com.example.ragdemo.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.ai.document.Document;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/search")
@RequiredArgsConstructor
@CrossOrigin(origins = "*")
public class SearchController {

    private final VectorStore vectorStore;
    
    /**
     * 相似度搜索
     */
    @GetMapping("/similar")
    public ResponseEntity<?> searchSimilar(
            @RequestParam("query") String query,
            @RequestParam(value = "topK", defaultValue = "5") int topK,
            @RequestParam(value = "threshold", defaultValue = "0.7") double threshold) {
        
        SearchRequest searchRequest = SearchRequest.query(query)
                .withTopK(topK)
                .withSimilarityThreshold(threshold);
        
        List<Document> documents = vectorStore.similaritySearch(searchRequest);
        
        List<Map<String, Object>> results = documents.stream()
                .map(doc -> {
                    Map<String, Object> result = new HashMap<>();
                    result.put("content", doc.getContent());
                    result.put("metadata", doc.getMetadata());
                    return result;
                })
                .toList();
        
        return ResponseEntity.ok(results);
    }
}
```

### 2. 添加健康检查接口

```java
package com.example.ragdemo.controller;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.HashMap;
import java.util.Map;

@RestController
public class HealthController {

    @GetMapping("/health")
    public ResponseEntity<Map<String, Object>> health() {
        Map<String, Object> response = new HashMap<>();
        response.put("status", "UP");
        response.put("service", "RAG Demo Service");
        response.put("timestamp", System.currentTimeMillis());
        return ResponseEntity.ok(response);
    }
}
```

## 完整示例代码

### 1. 项目结构

```
rag-demo/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/
│   │   │       └── example/
│   │   │           └── ragdemo/
│   │   │               ├── RagDemoApplication.java
│   │   │               ├── config/
│   │   │               │   └── VectorStoreConfig.java
│   │   │               ├── controller/
│   │   │               │   ├── ChatController.java
│   │   │               │   ├── DocumentController.java
│   │   │               │   ├── HealthController.java
│   │   │               │   └── SearchController.java
│   │   │               ├── entity/
│   │   │               │   └── Document.java
│   │   │               ├── repository/
│   │   │               │   └── DocumentRepository.java
│   │   │               └── service/
│   │   │                   ├── DocumentService.java
│   │   │                   └── RagService.java
│   │   └── resources/
│   │       └── application.yml
│   └── test/
│       └── java/
├── pom.xml
└── README.md
```

### 2. 启动类

```java
package com.example.ragdemo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class RagDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(RagDemoApplication.class, args);
    }
}
```

### 3. README.md

```markdown
# Spring AI Alibaba RAG 知识库演示

基于Spring AI Alibaba + PostgreSQL/pgvector构建的RAG知识库系统。

## 功能特性

- 📄 文档上传与解析（支持PDF、Word、TXT等格式）
- 🔍 向量检索与相似度搜索
- 💬 智能问答（基于RAG）
- 📚 知识库管理

## 快速开始

### 1. 环境准备

- JDK 17+
- PostgreSQL 14+（带pgvector扩展）
- 阿里云百炼API Key

### 2. 配置

编辑 `application.yml`：

```yaml
spring:
  ai:
    dashscope:
      api-key: your-api-key-here
```

### 3. 运行

```bash
mvn spring-boot:run
```

### 4. 测试

访问 http://localhost:8080

## API文档

### 文档管理

- `POST /api/documents/upload` - 上传文档
- `POST /api/documents/text` - 创建文本文档
- `GET /api/documents` - 获取所有文档
- `DELETE /api/documents/{docId}` - 删除文档

### 问答

- `POST /api/chat` - 简单问答
- `POST /api/chat/with-references` - 带引用的问答

### 搜索

- `GET /api/search/similar` - 相似度搜索

## 技术栈

- Spring Boot 3.2
- Spring AI Alibaba
- PostgreSQL + pgvector
- 通义千问大模型
```

## 常见问题

### Q1: 如何获取阿里云API Key？

**解答：**
1. 访问 https://bailian.console.aliyun.com/
2. 开通百炼服务
3. 在API Key管理页面创建新的API Key
4. 复制Key并配置到application.yml

### Q2: 向量数据库连接失败怎么办？

**解答：**
1. 检查PostgreSQL是否正常运行
2. 确认pgvector扩展已安装
3. 检查数据库连接配置（URL、用户名、密码）
4. 确认防火墙允许5432端口访问

### Q3: 文档解析失败怎么办？

**解答：**
1. 确认文档格式受支持（PDF、Word、TXT等）
2. 检查文档是否损坏
3. 查看日志获取详细错误信息
4. 尝试使用其他文档测试

### Q4: 检索结果不准确怎么办？

**解答：**
1. 调整相似度阈值（默认0.7）
2. 增加返回文档数量（topK）
3. 优化文档分块策略
4. 检查Embedding模型是否正常工作

### Q5: 如何优化RAG性能？

**解答：**
1. 使用合适的文档分块大小（建议500-1000字符）
2. 添加索引优化向量检索
3. 使用缓存减少重复计算
4. 考虑使用更高效的向量数据库（如Milvus）

### Q6: 支持哪些文档格式？

**解答：**
通过Apache Tika支持多种格式：
- 文本：TXT、MD、CSV
- Office：DOC、DOCX、PPT、PPTX、XLS、XLSX
- PDF：PDF
- 网页：HTML、XML
- 代码文件：Java、Python、JavaScript等

## 结语

通过本教程，您已经学会了：

1. ✅ RAG的基本概念和工作原理
2. ✅ Spring AI Alibaba框架的使用
3. ✅ 向量数据库（PostgreSQL + pgvector）的配置
4. ✅ 文档处理和向量化的实现
5. ✅ 基于RAG的智能问答系统构建
6. ✅ 完整的项目开发和部署流程

### 下一步建议

1. **优化分块策略**：根据文档类型调整分块大小和重叠度
2. **添加重排序**：使用更精确的排序模型优化检索结果
3. **多模态支持**：扩展支持图片、音频等多模态内容
4. **对话历史**：添加多轮对话支持
5. **用户管理**：添加多租户和用户权限管理

### 参考资源

- [Spring AI Alibaba官方文档](https://github.com/alibaba/spring-ai-alibaba)
- [Spring AI官方文档](https://spring.io/projects/spring-ai)
- [pgvector文档](https://github.com/pgvector/pgvector)
- [阿里云百炼控制台](https://bailian.console.aliyun.com/)

祝您开发顺利！
