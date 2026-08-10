---
title: deerflow-book 学习笔记
date: 2026-08-09
tags:
  - 学习
  - Agent
  - DeerFlow
  - LangGraph
---

# deerflow-book 学习笔记

> [!note] 说明
> deerflow-book 项目学习笔记(当前为框架版:仅章节标题,具体内容待填充),入口见 [[agent目前学习沉淀]]。
> 本书为 DeerFlow 二次开发的深度技术解析,与 [[easy-langent 学习笔记]] 的 LangChain/LangGraph 基础互补。

## 项目简介

- 仓库:https://github.com/hawkli-1994/deerflow-book
- 在线阅读:https://hawkli-1994.github.io/deerflow-book/
- 定位:DeerFlow 二次开发——理论、架构与源码剖析(面向硬核开发者的技术深度解析)
- 前置要求:Python 3.12+、LangChain / LangGraph 基础、Agent/LLM 应用开发经验、Docker
- 知识主线:待补充

---

## 各种配置
[[03-architecture#3.8 配置体系]]

[[04-project-structure#4.5 配置体系]]

[[05-agent-core#5.3 运行时配置（configurable）]]

## 易混复杂重点知识
[[03-architecture#3.6 请求处理流程]]

[[04-project-structure#4.1 整体目录结构]]

[[05-agent-core]]

## 第一章 引言:什么是 DeerFlow

**一句话**:待补充

### 1.1 DeerFlow 是什么
### 1.2 核心特性一览
### 1.3 技术栈
### 1.4 为什么选择 DeerFlow 做二次开发
### 1.5 本书结构
### 1.6 环境准备
### 1.7 小结

---

## 第二章 核心概念与设计哲学

**一句话**:待补充

### 2.1 设计哲学:解耦与可扩展
### 2.2 核心概念
### 2.3 工作流模型
### 2.4 与 LangGraph 的关系
### 2.5 小结

---

## 第三章 架构总览

**一句话**:待补充

### 3.1 系统架构图
### 3.2 核心组件详解
### 3.3 Agent 架构详解
### 3.4 ThreadState 与 AgentState
### 3.5 Sandbox 系统架构
### 3.6 请求处理流程
### 3.7 中间件链详解
### 3.8 配置体系
### 3.9 渐进式 Skill 加载的架构位置
### 3.10 小结

---

## 第四章 项目结构与模块划分

**一句话**:待补充

### 4.1 整体目录结构
### 4.2 核心模块详解
### 4.3 前端结构
### 4.4 开发命令
### 4.5 配置体系
### 4.6 DeerFlow 二次开发建议
### 4.7 小结

---

## 第五章 Agent 核心:LangGraph 编排逻辑

**一句话**:待补充

### 5.1 核心入口:make_lead_agent
### 5.2 ThreadState:Agent 状态定义
### 5.3 运行时配置(configurable)
### 5.4 LangGraph 工作流结构
### 5.5 中间件链(Middleware Chain)
### 5.6 工具系统(Tools)
### 5.7 系统 Prompt 生成
### 5.8 Client SDK:DeerFlowClient
### 5.9 二次开发:自定义 Lead Agent
### 5.10 小结

---

## 第六章 Skills 与 Tools:能力扩展机制

**一句话**:待补充

### 6.1 概念区分:Skill vs Tool
### 6.2 内置 Tools
### 6.3 Skills 系统架构
### 6.4 渐进式 Skill 加载
### 6.5 Skills 安全扫描
### 6.6 Skill 历史管理
### 6.7 Skill 安装器
### 6.8 Skill 与 Tool 的绑定
### 6.9 内置 Skills
### 6.10 Skill 安装与更新
### 6.11 MCP Server 集成
### 6.12 二次开发:自定义 Skill
### 6.13 小结

---

## 第七章 Sub-Agent 子代理体系

**一句话**:待补充

### 7.1 设计理念
### 7.2 Sub-Agent 配置与类型
### 7.3 Sub-AgentExecutor 执行引擎
### 7.4 背景任务管理机制
### 7.5 Agent 调用机制
### 7.6 LangGraph 中的 Sub-Agent 实现
### 7.7 DeerFlow Agent Teams 设计
### 7.8 长期记忆集成
### 7.9 上下文管理
### 7.10 二次开发指南
### 7.11 小结

---

## 第八章 Sandbox 沙箱执行环境

**一句话**:待补充

### 8.1 设计目标
### 8.2 三层架构
### 8.3 Sandbox 接口定义
### 8.4 Sandbox Provider 模式
### 8.5 Sandbox 中间件与生命周期
### 8.6 安全检查机制
### 8.7 Local Sandbox
### 8.8 Docker Sandbox
### 8.9 Provisioner Sandbox(K8s)
### 8.10 Sandbox Tools
### 8.11 Sandbox 中间件集成
### 8.12 Sandbox 审计中间件
### 8.13 安全考虑
### 8.14 小结

---

## 第九章 Memory 记忆系统

**一句话**:待补充

### 9.1 设计目标
### 9.2 模块结构
### 9.3 JSON 存储结构
### 9.4 Storage:文件存储与隔离
### 9.5 MemoryMiddleware:何时更新
### 9.6 Queue:异步与 Debounce
### 9.7 Updater:LLM 生成 JSON 更新
### 9.8 Fact 分类
### 9.9 上传文件事件不会长期记忆
### 9.10 Prompt 注入
### 9.11 配置要点
### 9.12 API 与 SDK 操作
### 9.13 企业扩展建议
### 9.14 小结

---

## 第十章 Context Engineering 上下文工程

**一句话**:待补充

### 10.1 为什么 Context Engineering 重要
### 10.2 Context 管理策略
### 10.3 上下文压缩技术
### 10.4 Memory 注入增强
### 10.5 RAG(检索增强生成)
### 10.6 DeerFlow 中的实现
### 10.7 二次开发:企业级上下文优化
### 10.8 小结

---

## 第十一章 MCP Server 集成

**一句话**:待补充

### 11.1 什么是 MCP
### 11.2 MCP 在 DeerFlow 中的位置
### 11.3 MCP Server 配置
### 11.4 MCP 客户端实现
### 11.5 MCP 工具适配器
### 11.6 MCP Server 实现
### 11.7 OAuth 认证支持
### 11.8 MCP 工具缓存机制
### 11.9 工具名前缀处理
### 11.10 错误处理和重试
### 11.11 常用 MCP Servers
### 11.12 二次开发:企业 MCP 集成
### 11.13 小结

---

## 第十二章 自定义 Skill 开发

**一句话**:待补充

### 12.1 Skill 开发流程
### 12.2 Skill 文件结构
### 12.3 Skill YAML 格式
### 12.4 Skill 实现
### 12.5 Skill 注册与加载
### 12.6 Skill 打包
### 12.7 Skill 调试
### 12.8 企业级 Skill 示例
### 12.9 多模态输出 Skill 开发
### 12.10 小结

---

## 第十三章 Human-in-the-Loop 人工审批机制

**一句话**:待补充

### 13.1 为什么需要 Human-in-the-Loop
### 13.2 审批节点设计
### 13.3 审批中间件
### 13.4 审批流程 API
### 13.5 审批通知集成
### 13.6 审批审计日志
### 13.7 二次开发指南
### 13.8 小结

---

## 第十四章 DeerFlow 企业级应用案例

**一句话**:待补充

### 14.1 案例背景:企业级应用需求分析
### 14.2 整体架构设计
### 14.3 多租户隔离实现
### 14.4 RBAC 权限控制
### 14.5 审计日志系统
### 14.6 企业知识库集成
### 14.7 Human-in-the-Loop 审批实现
### 14.8 项目级长程任务管理
### 14.9 多模态内容生成平台
### 14.10 部署架构
### 14.11 小结

---

## 附录

### 附录 A 配置参考
### 附录 B 贡献指南
### 附录 C 多模态 Skill 完整代码示例
### 附录 D 术语表(Glossary)

---

## 整体知识体系(面试串联话术)

> 待补充
