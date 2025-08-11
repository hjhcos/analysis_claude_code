# MCP智能搜索规则设计指南

## 概述

本指南将帮你设计一个MCP规则系统，能够根据项目内容的存在情况智能决定是否启用网络搜索功能。当用户查询的内容在项目中没有相关上下文时，自动调用Playwright进行网络搜索。

## 1. 需求分析

### 1.1 核心需求
- **智能判断**: 自动检测项目内容是否包含用户查询的相关信息
- **上下文感知**: 分析用户查询的语义，理解其技术领域和关键词
- **降级策略**: 项目内无相关内容时，自动启用网络搜索
- **结果融合**: 将项目内容和网络搜索结果进行智能整合

### 1.2 技术场景示例
- 用户询问："如何实现数据库热部署"
- 系统首先搜索项目中是否有数据库、热部署相关代码/文档
- 如果没有找到相关内容，则通过Playwright搜索网络资源
- 将搜索结果与项目上下文结合，提供针对性建议

## 2. MCP智能搜索规则架构

### 2.1 整体架构设计

```
┌─────────────────────────────────────────────────────────────┐
│                    MCP Smart Search Rule                   │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │   Query     │  │  Context    │  │   Decision Engine   │  │
│  │  Analyzer   │  │  Detector   │  │                     │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  Project    │  │  Web Search │  │   Result Merger     │  │
│  │  Searcher   │  │  Engine     │  │                     │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│                     MCP Transport                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ Playwright  │  │  Codebase   │  │    Knowledge Base   │  │
│  │   Client    │  │   Search    │  │      Manager        │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 核心组件说明

#### 2.2.1 Query Analyzer (查询分析器)
- **功能**: 分析用户查询的语义和技术领域
- **技术**: NLP关键词提取、技术栈识别
- **输出**: 结构化查询对象

#### 2.2.2 Context Detector (上下文检测器)
- **功能**: 检测项目中是否存在相关上下文
- **技术**: 语义搜索、代码分析、文档匹配
- **输出**: 相关性评分和匹配结果

#### 2.2.3 Decision Engine (决策引擎)
- **功能**: 根据上下文检测结果决定搜索策略
- **策略**: 阈值判断、规则匹配、智能路由
- **输出**: 搜索策略指令

## 3. 实现方案

### 3.1 MCP服务器配置

```json
{
  "mcpServers": {
    "smart_search": {
      "type": "stdio",
      "command": "node",
      "args": ["./smart-search-mcp-server.js"],
      "env": {
        "NODE_ENV": "production",
        "PLAYWRIGHT_BROWSERS_PATH": "./browsers"
      }
    }
  }
}
```

### 3.2 智能搜索MCP服务器实现

```javascript
// smart-search-mcp-server.js
import { createMCPServer } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import { chromium } from 'playwright';
import fs from 'fs';
import path from 'path';

class SmartSearchMCPServer {
  constructor() {
    this.browser = null;
    this.projectPath = process.cwd();
    this.contextThreshold = 0.3; // 相关性阈值
  }

  async initialize() {
    // 初始化Playwright浏览器
    this.browser = await chromium.launch({ headless: true });
    
    // 注册MCP工具
    this.server.addTool({
      name: "smart_search",
      description: "智能搜索工具：先检测项目上下文，必要时进行网络搜索",
      parameters: {
        type: "object",
        properties: {
          query: {
            type: "string",
            description: "用户查询内容"
          },
          domain: {
            type: "string",
            description: "技术领域（可选）",
            enum: ["database", "frontend", "backend", "devops", "ai", "general"]
          },
          forceWebSearch: {
            type: "boolean",
            description: "是否强制进行网络搜索",
            default: false
          }
        },
        required: ["query"]
      }
    });
  }

  // 查询分析器
  analyzeQuery(query, domain = null) {
    const techKeywords = {
      database: ['数据库', '热部署', 'mysql', 'postgresql', 'mongodb', 'redis', 'sql'],
      frontend: ['react', 'vue', 'angular', '前端', '组件', 'ui', 'css', 'javascript'],
      backend: ['api', '后端', 'server', 'microservice', 'spring', 'express', 'django'],
      devops: ['部署', 'docker', 'kubernetes', 'ci/cd', 'jenkins', 'gitlab'],
      ai: ['机器学习', 'ai', '人工智能', 'tensorflow', 'pytorch', 'llm']
    };

    // 提取关键词
    const keywords = [];
    const queryLower = query.toLowerCase();
    
    // 自动检测技术领域
    if (!domain) {
      for (const [tech, terms] of Object.entries(techKeywords)) {
        if (terms.some(term => queryLower.includes(term.toLowerCase()))) {
          domain = tech;
          break;
        }
      }
    }

    // 提取相关关键词
    if (domain && techKeywords[domain]) {
      techKeywords[domain].forEach(keyword => {
        if (queryLower.includes(keyword.toLowerCase())) {
          keywords.push(keyword);
        }
      });
    }

    return {
      originalQuery: query,
      domain: domain || 'general',
      keywords,
      searchTerms: this.generateSearchTerms(query, keywords)
    };
  }

  // 生成搜索词
  generateSearchTerms(query, keywords) {
    const terms = [query];
    
    // 添加关键词组合
    keywords.forEach(keyword => {
      terms.push(keyword);
      terms.push(`${keyword} ${query}`);
    });

    return [...new Set(terms)]; // 去重
  }

  // 项目上下文检测
  async detectProjectContext(analyzedQuery) {
    const { keywords, searchTerms, domain } = analyzedQuery;
    const contextResults = [];

    try {
      // 1. 搜索代码文件
      const codeResults = await this.searchCodeFiles(searchTerms);
      contextResults.push(...codeResults);

      // 2. 搜索文档文件
      const docResults = await this.searchDocuments(searchTerms);
      contextResults.push(...docResults);

      // 3. 搜索配置文件
      const configResults = await this.searchConfigs(searchTerms, domain);
      contextResults.push(...configResults);

      // 4. 计算相关性评分
      const relevanceScore = this.calculateRelevanceScore(contextResults, keywords);

      return {
        hasContext: relevanceScore > this.contextThreshold,
        relevanceScore,
        results: contextResults,
        summary: this.generateContextSummary(contextResults)
      };
    } catch (error) {
      console.error('项目上下文检测错误:', error);
      return {
        hasContext: false,
        relevanceScore: 0,
        results: [],
        summary: '上下文检测失败'
      };
    }
  }

  // 搜索代码文件
  async searchCodeFiles(searchTerms) {
    const results = [];
    const codeExtensions = ['.js', '.ts', '.py', '.java', '.go', '.rs', '.cpp', '.c'];
    
    for (const term of searchTerms) {
      try {
        // 使用grep或ripgrep搜索代码
        const grepResults = await this.grepSearch(term, codeExtensions);
        results.push(...grepResults);
      } catch (error) {
        console.error(`代码搜索错误 (${term}):`, error);
      }
    }

    return results;
  }

  // 搜索文档文件
  async searchDocuments(searchTerms) {
    const results = [];
    const docExtensions = ['.md', '.txt', '.rst', '.adoc'];
    
    for (const term of searchTerms) {
      try {
        const docResults = await this.grepSearch(term, docExtensions);
        results.push(...docResults);
      } catch (error) {
        console.error(`文档搜索错误 (${term}):`, error);
      }
    }

    return results;
  }

  // 搜索配置文件
  async searchConfigs(searchTerms, domain) {
    const results = [];
    const configFiles = {
      database: ['package.json', 'requirements.txt', 'pom.xml', 'docker-compose.yml'],
      frontend: ['package.json', 'webpack.config.js', 'vite.config.js'],
      backend: ['package.json', 'requirements.txt', 'go.mod', 'Cargo.toml'],
      devops: ['Dockerfile', 'docker-compose.yml', '.gitlab-ci.yml', 'Jenkinsfile'],
      general: ['package.json', 'requirements.txt', 'README.md']
    };

    const targetFiles = configFiles[domain] || configFiles.general;
    
    for (const file of targetFiles) {
      const filePath = path.join(this.projectPath, file);
      if (fs.existsSync(filePath)) {
        try {
          const content = fs.readFileSync(filePath, 'utf-8');
          for (const term of searchTerms) {
            if (content.toLowerCase().includes(term.toLowerCase())) {
              results.push({
                file: filePath,
                type: 'config',
                matches: [term],
                snippet: this.extractSnippet(content, term)
              });
            }
          }
        } catch (error) {
          console.error(`配置文件搜索错误 (${file}):`, error);
        }
      }
    }

    return results;
  }

  // Grep搜索实现
  async grepSearch(term, extensions) {
    const { exec } = await import('child_process');
    const { promisify } = await import('util');
    const execAsync = promisify(exec);

    try {
      const extPattern = extensions.map(ext => `*${ext}`).join(' -o -name ');
      const command = `find ${this.projectPath} \\( -name ${extPattern} \\) -exec grep -l "${term}" {} \\;`;
      
      const { stdout } = await execAsync(command);
      const files = stdout.trim().split('\n').filter(f => f);
      
      const results = [];
      for (const file of files) {
        try {
          const content = fs.readFileSync(file, 'utf-8');
          results.push({
            file,
            type: 'code',
            matches: [term],
            snippet: this.extractSnippet(content, term)
          });
        } catch (error) {
          console.error(`文件读取错误 (${file}):`, error);
        }
      }
      
      return results;
    } catch (error) {
      console.error('Grep搜索错误:', error);
      return [];
    }
  }

  // 提取代码片段
  extractSnippet(content, term, contextLines = 3) {
    const lines = content.split('\n');
    const matchingLines = [];
    
    lines.forEach((line, index) => {
      if (line.toLowerCase().includes(term.toLowerCase())) {
        const start = Math.max(0, index - contextLines);
        const end = Math.min(lines.length, index + contextLines + 1);
        matchingLines.push({
          lineNumber: index + 1,
          context: lines.slice(start, end).join('\n')
        });
      }
    });

    return matchingLines.slice(0, 3); // 最多返回3个匹配片段
  }

  // 计算相关性评分
  calculateRelevanceScore(results, keywords) {
    if (results.length === 0) return 0;

    let score = 0;
    const maxScore = keywords.length * 2; // 每个关键词最多2分

    keywords.forEach(keyword => {
      const keywordMatches = results.filter(result => 
        result.matches.some(match => 
          match.toLowerCase().includes(keyword.toLowerCase())
        )
      );
      
      if (keywordMatches.length > 0) {
        score += Math.min(2, keywordMatches.length * 0.5);
      }
    });

    return Math.min(1, score / maxScore); // 归一化到0-1之间
  }

  // 生成上下文摘要
  generateContextSummary(results) {
    if (results.length === 0) {
      return '项目中未找到相关内容';
    }

    const fileTypes = {};
    results.forEach(result => {
      const type = result.type;
      fileTypes[type] = (fileTypes[type] || 0) + 1;
    });

    const summary = Object.entries(fileTypes)
      .map(([type, count]) => `${type}: ${count}个文件`)
      .join(', ');

    return `项目中找到相关内容: ${summary}`;
  }

  // 网络搜索实现
  async performWebSearch(query, domain) {
    if (!this.browser) {
      throw new Error('浏览器未初始化');
    }

    const page = await this.browser.newPage();
    const searchResults = [];

    try {
      // 构建搜索URL
      const searchUrls = this.buildSearchUrls(query, domain);
      
      for (const url of searchUrls.slice(0, 3)) { // 最多搜索3个来源
        try {
          await page.goto(url, { waitUntil: 'networkidle' });
          
          // 提取搜索结果
          const results = await this.extractSearchResults(page, domain);
          searchResults.push(...results);
          
          await page.waitForTimeout(1000); // 避免请求过快
        } catch (error) {
          console.error(`网络搜索错误 (${url}):`, error);
        }
      }

      return {
        query,
        domain,
        results: searchResults.slice(0, 10), // 最多返回10个结果
        timestamp: new Date().toISOString()
      };
    } finally {
      await page.close();
    }
  }

  // 构建搜索URL
  buildSearchUrls(query, domain) {
    const encodedQuery = encodeURIComponent(query);
    const domainSpecificTerms = {
      database: '数据库',
      frontend: '前端开发',
      backend: '后端开发',
      devops: 'DevOps',
      ai: '人工智能'
    };

    const domainTerm = domainSpecificTerms[domain] || '';
    const enhancedQuery = domainTerm ? `${query} ${domainTerm}` : query;
    const encodedEnhancedQuery = encodeURIComponent(enhancedQuery);

    return [
      `https://www.google.com/search?q=${encodedEnhancedQuery}`,
      `https://stackoverflow.com/search?q=${encodedQuery}`,
      `https://github.com/search?q=${encodedQuery}&type=repositories`
    ];
  }

  // 提取搜索结果
  async extractSearchResults(page, domain) {
    const url = page.url();
    
    if (url.includes('google.com')) {
      return await this.extractGoogleResults(page);
    } else if (url.includes('stackoverflow.com')) {
      return await this.extractStackOverflowResults(page);
    } else if (url.includes('github.com')) {
      return await this.extractGitHubResults(page);
    }
    
    return [];
  }

  // 提取Google搜索结果
  async extractGoogleResults(page) {
    try {
      const results = await page.evaluate(() => {
        const searchResults = [];
        const resultElements = document.querySelectorAll('div.g');
        
        resultElements.forEach((element, index) => {
          if (index >= 5) return; // 最多5个结果
          
          const titleElement = element.querySelector('h3');
          const linkElement = element.querySelector('a');
          const snippetElement = element.querySelector('.VwiC3b');
          
          if (titleElement && linkElement) {
            searchResults.push({
              title: titleElement.textContent,
              url: linkElement.href,
              snippet: snippetElement ? snippetElement.textContent : '',
              source: 'Google'
            });
          }
        });
        
        return searchResults;
      });
      
      return results;
    } catch (error) {
      console.error('Google结果提取错误:', error);
      return [];
    }
  }

  // 提取StackOverflow结果
  async extractStackOverflowResults(page) {
    try {
      const results = await page.evaluate(() => {
        const searchResults = [];
        const questionElements = document.querySelectorAll('.question-summary');
        
        questionElements.forEach((element, index) => {
          if (index >= 5) return;
          
          const titleElement = element.querySelector('.question-hyperlink');
          const excerptElement = element.querySelector('.excerpt');
          
          if (titleElement) {
            searchResults.push({
              title: titleElement.textContent,
              url: `https://stackoverflow.com${titleElement.getAttribute('href')}`,
              snippet: excerptElement ? excerptElement.textContent : '',
              source: 'StackOverflow'
            });
          }
        });
        
        return searchResults;
      });
      
      return results;
    } catch (error) {
      console.error('StackOverflow结果提取错误:', error);
      return [];
    }
  }

  // 提取GitHub结果
  async extractGitHubResults(page) {
    try {
      const results = await page.evaluate(() => {
        const searchResults = [];
        const repoElements = document.querySelectorAll('div[data-testid="results-list"] > div');
        
        repoElements.forEach((element, index) => {
          if (index >= 5) return;
          
          const titleElement = element.querySelector('a[data-testid="results-list-repo-path"]');
          const descElement = element.querySelector('p');
          
          if (titleElement) {
            searchResults.push({
              title: titleElement.textContent,
              url: `https://github.com${titleElement.getAttribute('href')}`,
              snippet: descElement ? descElement.textContent : '',
              source: 'GitHub'
            });
          }
        });
        
        return searchResults;
      });
      
      return results;
    } catch (error) {
      console.error('GitHub结果提取错误:', error);
      return [];
    }
  }

  // 结果融合器
  mergeResults(projectContext, webSearchResults) {
    const mergedResults = {
      hasProjectContext: projectContext.hasContext,
      projectRelevanceScore: projectContext.relevanceScore,
      projectResults: projectContext.results,
      webResults: webSearchResults.results,
      recommendations: []
    };

    // 生成建议
    if (projectContext.hasContext) {
      mergedResults.recommendations.push(
        '基于项目现有代码和文档，建议参考以下实现方式:'
      );
      
      // 添加项目相关建议
      projectContext.results.slice(0, 3).forEach(result => {
        mergedResults.recommendations.push(
          `- 查看文件: ${result.file.replace(this.projectPath, '.')}`
        );
      });
    }

    if (webSearchResults.results.length > 0) {
      const separator = projectContext.hasContext ? '\n\n另外，以下是相关的在线资源:' : '在线搜索到以下相关资源:';
      mergedResults.recommendations.push(separator);
      
      // 添加网络搜索建议
      webSearchResults.results.slice(0, 3).forEach(result => {
        mergedResults.recommendations.push(
          `- [${result.source}] ${result.title}: ${result.url}`
        );
      });
    }

    return mergedResults;
  }

  // 主要的智能搜索方法
  async handleSmartSearch(params) {
    const { query, domain, forceWebSearch = false } = params;
    
    try {
      // 1. 分析查询
      const analyzedQuery = this.analyzeQuery(query, domain);
      
      // 2. 检测项目上下文
      const projectContext = await this.detectProjectContext(analyzedQuery);
      
      // 3. 决策是否需要网络搜索
      const shouldWebSearch = forceWebSearch || !projectContext.hasContext;
      
      let webSearchResults = { results: [] };
      if (shouldWebSearch) {
        // 4. 执行网络搜索
        webSearchResults = await this.performWebSearch(query, analyzedQuery.domain);
      }
      
      // 5. 融合结果
      const mergedResults = this.mergeResults(projectContext, webSearchResults);
      
      return {
        success: true,
        query: analyzedQuery,
        results: mergedResults,
        strategy: shouldWebSearch ? 'hybrid' : 'project-only',
        timestamp: new Date().toISOString()
      };
      
    } catch (error) {
      console.error('智能搜索错误:', error);
      return {
        success: false,
        error: error.message,
        query,
        timestamp: new Date().toISOString()
      };
    }
  }

  async cleanup() {
    if (this.browser) {
      await this.browser.close();
    }
  }
}

// 启动MCP服务器
async function main() {
  const server = new SmartSearchMCPServer();
  
  try {
    await server.initialize();
    
    // 处理工具调用
    server.server.addToolHandler(async (request) => {
      if (request.method === 'tools/call' && request.params.name === 'smart_search') {
        return await server.handleSmartSearch(request.params.arguments);
      }
    });
    
    // 启动服务器
    const transport = new StdioTransport();
    await server.server.connect(transport);
    
    console.error('智能搜索MCP服务器已启动');
    
  } catch (error) {
    console.error('MCP服务器启动失败:', error);
    process.exit(1);
  }
}

// 优雅关闭处理
process.on('SIGINT', async () => {
  console.error('接收到SIGINT信号，正在关闭服务器...');
  await server.cleanup();
  process.exit(0);
});

process.on('SIGTERM', async () => {
  console.error('接收到SIGTERM信号，正在关闭服务器...');
  await server.cleanup();
  process.exit(0);
});

if (import.meta.url === `file://${process.argv[1]}`) {
  main().catch(console.error);
}
```

## 4. 使用示例

### 4.1 基本使用

```javascript
// 在Claude Code中使用智能搜索
const searchTool = await client.tools.call({
  name: "smart_search",
  arguments: {
    query: "如何实现数据库热部署",
    domain: "database"
  }
});

console.log(searchTool.results);
```

### 4.2 响应示例

```json
{
  "success": true,
  "query": {
    "originalQuery": "如何实现数据库热部署",
    "domain": "database",
    "keywords": ["数据库", "热部署"],
    "searchTerms": ["如何实现数据库热部署", "数据库", "热部署"]
  },
  "results": {
    "hasProjectContext": false,
    "projectRelevanceScore": 0.1,
    "projectResults": [],
    "webResults": [
      {
        "title": "数据库热部署最佳实践",
        "url": "https://example.com/db-hotdeploy",
        "snippet": "介绍如何在生产环境中实现数据库的热部署...",
        "source": "Google"
      }
    ],
    "recommendations": [
      "在线搜索到以下相关资源:",
      "- [Google] 数据库热部署最佳实践: https://example.com/db-hotdeploy",
      "- [StackOverflow] Database Hot Deployment Strategies: https://stackoverflow.com/q/12345",
      "- [GitHub] hot-deploy-db: https://github.com/user/hot-deploy-db"
    ]
  },
  "strategy": "hybrid",
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

## 5. 高级配置

### 5.1 相关性阈值调整

```javascript
// 在服务器初始化时调整阈值
this.contextThreshold = 0.3; // 降低阈值，更容易触发网络搜索
```

### 5.2 自定义搜索源

```javascript
// 添加自定义搜索源
buildSearchUrls(query, domain) {
  const encodedQuery = encodeURIComponent(query);
  
  return [
    `https://www.google.com/search?q=${encodedQuery}`,
    `https://stackoverflow.com/search?q=${encodedQuery}`,
    `https://github.com/search?q=${encodedQuery}`,
    `https://dev.to/search?q=${encodedQuery}`, // 添加Dev.to
    `https://medium.com/search?q=${encodedQuery}` // 添加Medium
  ];
}
```

### 5.3 缓存机制

```javascript
class SmartSearchMCPServer {
  constructor() {
    this.searchCache = new Map();
    this.cacheExpiry = 1000 * 60 * 60; // 1小时缓存
  }

  async handleSmartSearch(params) {
    const cacheKey = JSON.stringify(params);
    const cached = this.searchCache.get(cacheKey);
    
    if (cached && Date.now() - cached.timestamp < this.cacheExpiry) {
      return cached.result;
    }
    
    const result = await this.performSearch(params);
    this.searchCache.set(cacheKey, {
      result,
      timestamp: Date.now()
    });
    
    return result;
  }
}
```

## 6. 测试和调试

### 6.1 单元测试

```javascript
// test/smart-search.test.js
import { test, expect } from '@jest/globals';
import { SmartSearchMCPServer } from '../smart-search-mcp-server.js';

test('查询分析功能', () => {
  const server = new SmartSearchMCPServer();
  const result = server.analyzeQuery('如何实现数据库热部署', 'database');
  
  expect(result.domain).toBe('database');
  expect(result.keywords).toContain('数据库');
  expect(result.keywords).toContain('热部署');
});

test('项目上下文检测', async () => {
  const server = new SmartSearchMCPServer();
  const analyzedQuery = server.analyzeQuery('React组件开发');
  const context = await server.detectProjectContext(analyzedQuery);
  
  expect(context).toHaveProperty('hasContext');
  expect(context).toHaveProperty('relevanceScore');
});
```

### 6.2 调试模式

```javascript
// 启用详细日志
const server = new SmartSearchMCPServer();
server.debugMode = process.env.NODE_ENV === 'development';

if (server.debugMode) {
  console.log('查询分析结果:', analyzedQuery);
  console.log('项目上下文:', projectContext);
  console.log('搜索策略:', shouldWebSearch ? 'hybrid' : 'project-only');
}
```

## 7. 部署和维护

### 7.1 依赖安装

```bash
# 安装必要依赖
npm install @modelcontextprotocol/sdk playwright

# 安装浏览器
npx playwright install chromium
```

### 7.2 系统要求

- Node.js >= 16.0.0
- 足够的磁盘空间存储Playwright浏览器
- 网络连接用于网络搜索
- 读取项目文件的权限

### 7.3 性能优化

- 使用缓存减少重复搜索
- 限制并发搜索请求数量
- 定期清理浏览器缓存
- 优化正则表达式和文件搜索

## 8. 扩展功能

### 8.1 多语言支持

```javascript
const languageDetector = {
  detect(query) {
    // 检测查询语言
    if (/[\u4e00-\u9fff]/.test(query)) return 'zh';
    return 'en';
  },
  
  translate(query, from, to) {
    // 实现翻译逻辑
    return translatedQuery;
  }
};
```

### 8.2 学习机制

```javascript
class LearningSystem {
  constructor() {
    this.userFeedback = new Map();
    this.queryPatterns = new Map();
  }
  
  recordFeedback(query, results, helpful) {
    // 记录用户反馈，优化搜索策略
  }
  
  analyzePatterns() {
    // 分析查询模式，改进相关性算法
  }
}
```

### 8.3 自定义规则

```javascript
// rules/custom-rules.js
export const customRules = {
  // 特定项目的自定义规则
  database: {
    threshold: 0.2,
    preferredSources: ['dba.stackexchange.com'],
    excludeTerms: ['deprecated', 'legacy']
  },
  
  frontend: {
    threshold: 0.4,
    preferredSources: ['reactjs.org', 'vuejs.org'],
    includeGithubTrending: true
  }
};
```

## 总结

这个MCP智能搜索规则系统提供了：

1. **智能判断**: 自动分析项目内容相关性
2. **多源搜索**: 整合项目内容和网络资源
3. **可配置性**: 支持自定义阈值和搜索源
4. **扩展性**: 易于添加新的搜索引擎和功能
5. **性能优化**: 缓存和并发控制

通过这个系统，你可以实现真正智能的搜索体验，既能充分利用项目现有资源，又能在必要时获取最新的网络信息。