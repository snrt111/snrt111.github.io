# Hexo 博客项目

## 项目简介
这是一个基于 Hexo 框架搭建的个人博客项目，使用 [主题名称] 主题，支持 Markdown 写作，具有响应式设计。

## 技术栈
- Hexo 6.x
- Node.js 16.x+
- Git
- [主题名称] 主题
- [其他插件]

## 项目结构
blog/
├── source/ # 资源文件夹
│ ├── _posts/ # 文章目录
│ ├── _drafts/ # 草稿目录
│ └── images/ # 图片资源
├── themes/ # 主题文件夹
├── public/ # 生成的静态文件
├── _config.yml # 网站配置
└── package.json # 依赖配置

## 快速开始

### 环境要求
- Node.js 16.x 或更高版本
- Git

### 安装步骤

1. 克隆项目
```bash
git clone [您的仓库地址]
cd blog
```

2. 安装依赖
```bash
npm install
```

3.安装主题
```bash
git clone [主题仓库地址] themes/[主题名称]
```

### 本地开发
1. 启动本地服务器
```bash
hexo server
# 或使用缩写
hexo s
```

2. 打开浏览器访问 [http://localhost:4000](http://localhost:4000)

### 写作指南
1. 创建新文章
```bash
hexo new "文章标题"
```

2. 创建新草稿
```bash
hexo new draft "草稿标题"
```

3. 发布草稿
```bash
hexo publish "草稿标题"
```

4. 文章格式示例
```markdown
---
title: 文章标题
date: 2023-01-01 00:00:00
tags:
  - 标签1
  - 标签2
categories:
  - 分类1
  - 分类2
---

文章内容
```

### 部署
1. 生成静态文件
```bash
hexo generate
# 或使用缩写
hexo g
```

2. 部署到远程服务器
```bash
hexo deploy
# 或使用缩写
hexo d
```

3. 一键生成并部署
```bash
hexo clean && hexo generate && hexo deploy
# 或使用缩写
hexo clean && hexo g && hexo d
```

### 常用命令
- `hexo new <title>`：创建新文章
- `hexo new draft <title>`：创建新草稿
- `hexo new page <title>`：创建新页面
- `hexo generate`：生成静态文件
- `hexo server`：启动本地服务器
- `hexo deploy`：部署到远程服务器
- `hexo clean`：清理缓存文件
- `hexo version`：查看版本信息

### 自定义配置
主要配置文件 _config.yml 包含以下重要配置：

- 网站基本信息（标题、副标题、描述等）
- 主题配置
- URL 配置
- 写作配置
- 分类和标签配置
- 部署配置
- 详细配置说明请参考 Hexo 官方文档。


### 主题配置
主题配置文件位于 themes/[主题名称]/_config.yml，可以根据需要进行自定义配置。

### 插件推荐
- hexo-generate-feed：生成 RSS、Atom 订阅源
- hexo-generator-sitemap：生成站点地图
- hexo-generator-archive：生成归档页面
- hexo-generator-category：生成分类页面
- hexo-generator-tag：生成标签页面
- hexo-generator-index：生成首页
- hexo-generator-search：生成搜索页面
- hexo-generator-index-pin-top：文章置顶
- hexo-wordcount：字数统计


### 常见问题
1. 如何更换主题？
   1. 在 themes 文件夹下克隆主题仓库
   2. 修改 _config.yml 中的 theme 配置项为新的主题名称
   3. 执行 `hexo clean && hexo generate && hexo deploy` 重新生成并部署
   4. 访问博客查看效果
2. 如何添加自定义页面？
   1. 在 source 文件夹下创建新文件夹，如 about
   2. 在 about 文件夹下创建 index.md 文件，并添加页面内容
   3. 在 _config.yml 中添加页面配置，如：
   ```yaml
   menu:
     About: /about/
   ```
   4. 执行 `hexo clean && hexo generate && hexo deploy` 重新生成并部署
   5. 访问博客查看效果
3. 如何添加自定义插件？
   1. 在博客根目录下执行 `npm install [插件名称] --save`
   2. 在 _config.yml 中添加插件配置，如：
   ```yaml
   plugins:
     - [插件名称]
   ```
   3. 执行 `hexo clean && hexo generate && hexo deploy` 重新生成并部署
   4. 访问博客查看效果

### 贡献指南
如果您有任何建议或改进意见，欢迎提交 Pull Request 或 Issue。

### 版本历史
-  v1.0.0：初始版本  

### 许可证
本项目遵循 MIT 许可证。

### 参考链接
- [Hexo 官方文档]：https://hexo.io/zh-cn/docs/
- [Markdown 语法]：https://www.markdownguide.org/basic-syntax/

