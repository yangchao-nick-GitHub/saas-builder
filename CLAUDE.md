# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

**SaaS-Builder** 是一个 AI 原生的 Python CLI 工具,用于快速生成全栈 SaaS 应用程序。用户只需提供项目描述,AI 就会自动生成包含 React 前端、Flask 后端、数据库、认证系统的完整应用。

### 核心工作流程

```
用户输入 → BuildAgent → Tools执行 → LLM生成代码 → 文件写入
```

1. **CLI 入口**: `gocodeo_cli/cli.py` - 定义 `saas-builder init` 命令
2. **命令层**: `gocodeo_cli/commands/build.py:init()` - 收集用户输入并启动构建流程
3. **Agent 层**: `gocodeo_cli/agents/build_agent.py:run_build_flow()` - 编排三个阶段的执行
4. **工具层**: `gocodeo_cli/tools/build_tools.py` - 每个工具对应一个构建阶段
5. **服务层**: `gocodeo_cli/services/llm_service.py` - 统一的 LLM 接口(支持 Anthropic/OpenAI/Gemini)

## 重要模块说明

### Agent 系统 (`gocodeo_cli/agents/`)

- **base.py**: 定义 `BaseAgent` 基类和 `AgentMemory`
  - `AgentMemory`: 存储消息、上下文和已生成的文件
  - `BaseAgent.process_response()`: 解析 LLM 返回的 JSON 并写入文件
  - `BaseAgent._load_reference_zip()`: 从 zip 文件加载参考代码

- **build_agent.py**: `BuildAgent` 实现完整的构建流程
  - Stage 1: 初始化项目脚手架 (所有模式)
  - Stage 2: 添加认证系统 (仅 React+Flask+SQLite)
  - Stage 3: 添加数据持久化 (仅 React+Flask+SQLite)

### Tools 系统 (`gocodeo_cli/tools/`)

每个 Tool 都是一个异步执行器,负责调用 LLM 生成特定功能的代码:

- **InitializeTool**: 生成项目基础结构和 UI
- **AddAuthTool**: 添加 Flask 认证系统
- **AddDataTool**: 添加数据库模型和迁移
- **NpmInstallTool**: 安装前端依赖
- **NpmDevServerTool**: 启动开发服务器
- **SqlMigrationTool**: 执行 SQL 迁移

### 模板系统 (`gocodeo_cli/templates/`)

- **stacks/**: 参考代码 ZIP 包(被 LLM 用作 few-shot 示例)
  - `e-commerce_template.zip`: 电商模板
  - `marketing_template.zip`: SaaS 营销模板
  - `crm.zip`: CRM 模板
  - `sample_ui_e-commerce.zip`: UI-only 电商模板

- **prompts/**: AI 提示词模板
  - `init.txt`: 项目初始化提示词
  - `auth.txt`: 认证实现提示词
  - `data.txt`: 数据持久化提示词
  - `system.txt`: 系统/格式提示词(JSON 格式要求)
  - `init_ui.txt`: UI-only 模式初始化提示词

### 技术栈选项

- **1**: React.js (UI Only) - 仅前端,使用 `init_ui.txt` 提示词
- **2**: React + Flask + SQLite - 全栈,使用 `init.txt` 提示词,执行认证和数据阶段

## 常用命令

### 开发安装
```bash
pip install -e .
```

### 运行构建
```bash
saas-builder init
```

### 测试
```bash
# 目前项目没有配置测试命令
# 运行: saas-builder init 并手动验证生成的项目
```

### Lint
```bash
# 项目使用 Ruff (配置在 pyproject.toml)
ruff check gocodeo_cli/
```

## 关键设计决策

### 1. Agent-Tool 架构
- Agent 负责编排和状态管理
- Tool 负责具体的代码生成任务
- 每个 Tool 独立执行,返回成功/失败状态

### 2. 参考代码注入
- 参考代码从 ZIP 文件加载,而非硬编码
- 使用 `importlib.resources` 读取打包后的模板文件
- 参考代码被注入到 LLM 提示词中作为 few-shot 示例

### 3. LLM 响应处理
- LLM 返回 JSON 格式的文件结构
- `process_response()` 支持多种 JSON 格式:
  - `{"file_path": "content"}`
  - `{"files": {...}, "migrations": {...}}`
  - `{"frontend": {...}, "backend": {...}}`
- 使用 `json-repair` 库修复格式错误的 JSON

### 4. 进程管理
- `NpmDevServerTool` 在后台启动开发服务器
- BuildAgent 注册信号处理器(SIGINT/SIGTERM)确保清理
- `atexit` 处理器确保终端关闭时清理子进程

## AI 模型支持

项目支持三种 LLM 提供商,通过 `services/llm_service.py` 统一接口:

- **Claude 3.7 Sonnet** (`claude-3-7-sonnet-20250219`): 默认,最高质量
- **GPT-4.1** (`gpt-4.1`): 快速可靠
- **Gemini 2.5 Pro** (`gemini-2.5-pro-preview-05-06`): 高级推理

API 密钥通过环境变量或交互式提示输入:
- `ANTHROPIC_API_KEY`
- `OPENAI_API_KEY`
- `GOOGLE_API_KEY`

## 添加新模板

1. 在 `gocodeo_cli/templates/stacks/` 添加新的 ZIP 文件
2. 在 `commands/build.py:TEMPLATE_STACKS` 中添加选项
3. 在 `agents/base.py:_load_reference_project()` 中添加映射

## 调试技巧

- LLM 调用失败时,检查 `services/llm_service.py` 的异常处理
- JSON 解析失败时,`agents/base.py:process_response()` 会尝试修复
- 模板加载失败时,检查 ZIP 文件路径和 `importlib.resources` 使用
- 启用日志: 查看 `agents/base.py` 中的 logger 输出
