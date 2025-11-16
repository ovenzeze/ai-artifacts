# AI 实时语音聊天助手 - 设计方案

## 项目概述

创建一个具备工具调用能力的AI实时语音聊天助手，能够通过语音与用户交互，执行本地文件系统操作、运行脚本命令、编写代码等任务。

## 技术选型建议

### 方案对比

| 方案 | 优势 | 劣势 | 推荐度 |
|------|------|------|--------|
| **OpenAI Realtime API** | • 延迟极低 (<500ms)<br>• 原生支持工具调用<br>• 文档完善，示例丰富<br>• 单模型处理音频 | • 需要 OpenAI API key<br>• 商业服务有成本<br>• 依赖 OpenAI 服务 | ⭐⭐⭐⭐⭐ |
| **LiveKit Agents** | • 完全开源<br>• 支持多种 STT/LLM/TTS 提供商<br>• 可自部署<br>• 灵活的工具系统 | • 需要配置多个组件<br>• 初始设置较复杂<br>• 需要 LiveKit 服务器 | ⭐⭐⭐⭐ |
| **Vapi AI** | • 易于集成<br>• 管理的服务<br>• 支持大规模并发 | • 商业服务<br>• 定制化受限<br>• 依赖第三方 | ⭐⭐⭐ |

### 推荐方案：OpenAI Realtime API + 自定义工具

**理由：**
1. 最低延迟，最佳用户体验
2. 原生支持函数调用，易于集成文件系统操作
3. 与现有项目已使用的 OpenAI SDK 兼容
4. 丰富的社区示例和文档

## 系统架构设计

```
┌─────────────────────────────────────────────────────────────┐
│                        前端 (Next.js)                        │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Voice UI Component                                    │  │
│  │  • 麦克风权限管理                                       │  │
│  │  • 音频可视化                                          │  │
│  │  • 录音/播放控制                                       │  │
│  │  • WebSocket 连接管理                                  │  │
│  └───────────────────────────────────────────────────────┘  │
│                            ↕                                 │
│                  WebSocket (实时双向通信)                    │
└─────────────────────────────────────────────────────────────┘
                              ↕
┌─────────────────────────────────────────────────────────────┐
│                    后端 API (Next.js API Routes)             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  WebSocket Handler (/api/voice)                       │  │
│  │  • 管理客户端连接                                      │  │
│  │  • 转发音频数据                                        │  │
│  │  • 处理工具调用                                        │  │
│  └───────────────────────────────────────────────────────┘  │
│                            ↕                                 │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  OpenAI Realtime API Client                           │  │
│  │  • 建立 WebSocket 连接到 OpenAI                        │  │
│  │  • 发送/接收音频流                                     │  │
│  │  • 处理事件（转录、响应、工具调用）                    │  │
│  └───────────────────────────────────────────────────────┘  │
│                            ↕                                 │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Tool Executors (工具执行器)                          │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │ 文件系统工具                                     │  │  │
│  │  │ • readFile()   - 读取文件内容                   │  │  │
│  │  │ • writeFile()  - 写入文件                       │  │  │
│  │  │ • listDir()    - 列出目录                       │  │  │
│  │  │ • deleteFile() - 删除文件                       │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │ 脚本执行工具                                     │  │  │
│  │  │ • executeCommand() - 执行终端命令               │  │  │
│  │  │ • runScript()      - 运行脚本文件               │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │ 代码编写工具                                     │  │  │
│  │  │ • createCode()     - 创建新代码文件             │  │  │
│  │  │ • modifyCode()     - 修改现有代码               │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              ↕
┌─────────────────────────────────────────────────────────────┐
│                    OpenAI Realtime API                       │
│  • gpt-4o-realtime-preview                                   │
│  • 语音输入 → 文本转录 → LLM 推理 → 语音输出                │
│  • 原生工具调用支持                                          │
└─────────────────────────────────────────────────────────────┘
```

## 核心组件设计

### 1. 前端语音界面组件

**文件：** `components/VoiceAssistant.tsx`

**功能：**
- 麦克风权限请求和管理
- 实时音频录制（使用 MediaRecorder API）
- 音频流可视化（音频波形）
- WebSocket 连接到后端
- 播放 AI 响应音频
- 显示转录文本和工具调用状态

**关键技术：**
- Web Audio API
- MediaRecorder API
- WebSocket
- AudioContext

### 2. 后端 WebSocket 处理器

**文件：** `app/api/voice/route.ts`

**功能：**
- 接受客户端 WebSocket 连接
- 建立到 OpenAI Realtime API 的连接
- 双向转发音频数据
- 拦截和处理工具调用事件
- 执行工具并返回结果

**事件流：**
```javascript
客户端音频 → WebSocket → 后端 → OpenAI Realtime API
OpenAI 响应 ← WebSocket ← 后端 ← OpenAI Realtime API
工具调用 → 执行 → 结果返回 → OpenAI
```

### 3. 工具定义和执行器

**文件：** `lib/voice-tools/`

**工具结构：**
```typescript
interface Tool {
  name: string;
  description: string;
  parameters: {
    type: "object";
    properties: Record<string, any>;
    required: string[];
  };
  execute: (params: any) => Promise<any>;
}
```

**安全考虑：**
- 文件路径验证（防止路径遍历攻击）
- 命令白名单（限制可执行的命令）
- 权限检查（检查文件访问权限）
- 输出限制（限制返回数据大小）

### 4. 系统提示词设计

**目标：** 让 AI 理解其角色和能力

```
你是一个强大的 AI 语音助手，具备操作本地文件系统的能力。

你可以：
1. 读取、写入、删除文件
2. 列出目录内容
3. 执行终端命令
4. 编写和修改代码

在执行任何操作前：
- 向用户确认操作意图
- 对于删除或修改操作，需要明确确认
- 解释你将要执行的操作

安全原则：
- 不执行可能损害系统的命令
- 不访问敏感文件（如 .env、私钥等）
- 在操作前进行路径验证
```

## 实现步骤

### 第一阶段：基础语音对话

1. ✅ 安装必要的依赖包
2. ✅ 创建基础的语音 UI 组件
3. ✅ 实现 WebSocket API 路由
4. ✅ 集成 OpenAI Realtime API
5. ✅ 实现音频录制和播放

### 第二阶段：工具调用能力

1. ⬜ 定义文件系统工具
2. ⬜ 实现工具执行器
3. ⬜ 集成工具到 Realtime API
4. ⬜ 测试工具调用流程

### 第三阶段：增强和优化

1. ⬜ 添加音频可视化
2. ⬜ 实现对话历史记录
3. ⬜ 添加错误处理和重试机制
4. ⬜ 优化延迟和性能
5. ⬜ 添加安全检查和权限管理

## 可借鉴的开源项目

### 1. OpenAI Realtime Console
- **仓库：** https://github.com/openai/openai-realtime-console
- **借鉴：** WebSocket 连接管理、事件处理模式

### 2. Twilio + OpenAI Realtime API (Node.js)
- **仓库：** https://github.com/twilio-samples/speech-assistant-openai-realtime-api-node
- **借鉴：** 音频流处理、WebSocket 中继

### 3. LiveKit Agents Starter (Node.js)
- **仓库：** https://github.com/livekit-examples/agent-starter-node
- **借鉴：** 工具定义模式（Zod schemas）、结构化工具执行

### 4. OpenAI Realtime Voice Assistant V2
- **仓库：** https://github.com/Barty-Bart/openai-realtime-api-voice-assistant-V2
- **借鉴：** 函数调用集成、RAG 模式

## 安全和隐私考虑

### 1. 文件系统访问控制
- 实现沙箱机制，限制可访问的目录
- 路径白名单/黑名单
- 文件类型限制

### 2. 命令执行安全
- 命令白名单
- 参数验证和清理
- 超时机制
- 资源使用限制

### 3. 数据隐私
- 音频数据不存储在服务器
- 使用 HTTPS/WSS 加密传输
- 用户控制录音开关

### 4. API 密钥管理
- 环境变量存储
- 不暴露给客户端
- 使用用户自己的 API 密钥（可选）

## 🎉 现成工具库推荐（无需重复造轮子！）

### 重要发现：OpenAI Realtime API 原生支持 MCP！

**好消息：** OpenAI Realtime API 在 2025 年已经原生支持 MCP（Model Context Protocol），可以直接在 session 配置中传入 MCP 服务器 URL，API 会自动处理工具调用！

### 推荐使用的 MCP 服务器

#### 1. **文件系统操作**
```bash
# 官方 MCP Filesystem Server
npm install @modelcontextprotocol/server-filesystem
# 或直接运行
npx -y @modelcontextprotocol/server-filesystem /path/to/allowed/directory
```

**功能：**
- ✅ 读/写文件
- ✅ 创建/列出/删除目录
- ✅ 移动文件/目录
- ✅ 搜索文件
- ✅ 获取文件元数据
- ✅ 安全的目录访问控制

**NPM 包：** `@modelcontextprotocol/server-filesystem`

---

#### 2. **代码执行** - E2B MCP Server（你的项目已在使用 E2B！）

```bash
# E2B MCP Server (TypeScript)
npm install @e2b/mcp-server
```

**GitHub：** https://github.com/e2b-dev/mcp-server

**功能：**
- ✅ 在隔离沙箱中执行代码
- ✅ 支持 Python、JavaScript 等多种语言
- ✅ 云端安全执行环境
- ✅ 与你现有的 E2B 基础设施完美集成

**优势：** 你的项目已经使用 E2B，可以无缝集成！

---

#### 3. **终端/Shell 命令执行**

多个优秀的开源实现可选：

**选项 A - Command-Line MCP** (推荐，安全性最好)
```bash
npm install cmd-line-mcp
```
- **仓库：** andresthor/cmd-line-mcp
- **特点：** 双重安全模型（命令权限 + 目录权限）
- **权限分类：** read/write/system 三种命令类别

**选项 B - Shell MCP** (简单易用)
```bash
npm install shell-mcp
```
- **仓库：** kevinwatt/shell-mcp
- **特点：** 白名单命令和参数
- **环境变量：** `ALLOWED_COMMANDS="cat,ls,echo"`

**选项 C - SSH MCP** (远程执行)
- **仓库：** tufantunc/ssh-mcp
- **特点：** 通过 SSH 控制远程服务器

---

#### 4. **其他有用的 MCP 服务器**

**Git 操作：**
```bash
npm install @modelcontextprotocol/server-git
```
- 读取、搜索、操作 Git 仓库

**Web 内容获取：**
```bash
npm install @modelcontextprotocol/server-fetch
```
- 获取并转换网页内容供 LLM 处理

---

## 集成架构（使用 MCP）

```
┌─────────────────────────────────────────────────────────────┐
│                    前端 (Next.js)                            │
│                  Voice UI Component                          │
│                  WebSocket Connection                        │
└─────────────────────────────────────────────────────────────┘
                              ↕
┌─────────────────────────────────────────────────────────────┐
│              后端 WebSocket Handler                          │
│           (/app/api/voice/route.ts)                         │
└─────────────────────────────────────────────────────────────┘
                              ↕
┌─────────────────────────────────────────────────────────────┐
│          OpenAI Realtime API (with MCP support)             │
│                                                              │
│  session.config({                                           │
│    mcp_servers: [                                           │
│      { url: "mcp://filesystem", ... },                      │
│      { url: "mcp://e2b-code-executor", ... },               │
│      { url: "mcp://shell-commands", ... }                   │
│    ]                                                        │
│  })                                                         │
└─────────────────────────────────────────────────────────────┘
                              ↕
┌─────────────────────────────────────────────────────────────┐
│                     MCP 服务器层                             │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │ Filesystem │  │ E2B Code   │  │ Shell/CMD  │            │
│  │   Server   │  │  Executor  │  │   Server   │            │
│  └────────────┘  └────────────┘  └────────────┘            │
└─────────────────────────────────────────────────────────────┘
                              ↕
┌─────────────────────────────────────────────────────────────┐
│                   本地文件系统 / 沙箱环境                     │
└─────────────────────────────────────────────────────────────┘
```

## 依赖包清单（更新版）

```json
{
  "dependencies": {
    "@openai/realtime-api-beta": "^0.4.0",
    "ws": "^8.18.0",
    "zod": "^3.23.8",

    // MCP 核心
    "@modelcontextprotocol/sdk": "^1.0.0",

    // MCP 服务器（按需选择）
    "@modelcontextprotocol/server-filesystem": "^1.0.0",
    "@e2b/mcp-server": "latest",
    "cmd-line-mcp": "latest",  // 或 "shell-mcp"

    // 可选：其他工具
    "@modelcontextprotocol/server-git": "^1.0.0",
    "@modelcontextprotocol/server-fetch": "^1.0.0"
  },
  "devDependencies": {
    "@types/ws": "^8.5.12",
    "@modelcontextprotocol/inspector": "latest"  // MCP 调试工具
  }
}
```

## 预期功能演示

### 用户场景示例

**场景 1：查看文件**
```
用户：请帮我看一下 package.json 文件的内容
助手：好的，我来读取 package.json 文件
      [调用 readFile 工具]
      这个项目是一个 Next.js 应用，使用了...
```

**场景 2：创建代码**
```
用户：帮我创建一个新的 React 组件 Button.tsx
助手：好的，我将创建一个 Button 组件
      [调用 createCode 工具]
      我已经创建了 Button.tsx，包含了基础的按钮组件...
```

**场景 3：执行命令**
```
用户：运行 npm test 看看测试结果
助手：我将为你运行测试
      [调用 executeCommand 工具]
      测试已完成，所有 12 个测试都通过了...
```

## 🚀 快速开始指南（使用现成工具）

### 第一步：安装 MCP 服务器

```bash
# 安装核心依赖
npm install @openai/realtime-api-beta ws
npm install @modelcontextprotocol/sdk

# 安装 MCP 服务器（根据需求选择）
npm install @modelcontextprotocol/server-filesystem
npm install @e2b/mcp-server
npm install cmd-line-mcp
```

### 第二步：配置 MCP 服务器

创建 `mcp-config.json`：
```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/home/user/ai-artifacts"
      ]
    },
    "e2b": {
      "command": "npx",
      "args": ["-y", "@e2b/mcp-server"],
      "env": {
        "E2B_API_KEY": "your-e2b-api-key"
      }
    },
    "shell": {
      "command": "npx",
      "args": ["-y", "cmd-line-mcp"],
      "env": {
        "ALLOWED_COMMANDS": "ls,cat,echo,npm,git"
      }
    }
  }
}
```

### 第三步：集成到 OpenAI Realtime API

```typescript
// app/api/voice/route.ts
import { RealtimeClient } from '@openai/realtime-api-beta';
import { MCPClient } from '@modelcontextprotocol/sdk/client/index.js';

const client = new RealtimeClient({
  apiKey: process.env.OPENAI_API_KEY,
  model: 'gpt-4o-realtime-preview',
});

// 配置 MCP 服务器
await client.updateSession({
  mcp_servers: [
    {
      name: 'filesystem',
      url: 'stdio://filesystem',
      config: { /* ... */ }
    },
    {
      name: 'e2b',
      url: 'stdio://e2b',
      config: { /* ... */ }
    }
  ]
});
```

### 第四步：前端语音 UI

```tsx
// components/VoiceAssistant.tsx
import { useState, useRef } from 'react';

export function VoiceAssistant() {
  const [isRecording, setIsRecording] = useState(false);
  const wsRef = useRef<WebSocket | null>(null);

  const startRecording = async () => {
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
    const ws = new WebSocket('ws://localhost:3000/api/voice');

    // 处理音频流...
    setIsRecording(true);
  };

  return (
    <button onClick={startRecording}>
      {isRecording ? '🎤 Recording...' : '🎙️ Start Voice Chat'}
    </button>
  );
}
```

---

## MCP vs 自定义工具对比

| 特性 | MCP 服务器（推荐） | 自定义工具实现 |
|------|------------------|--------------|
| 开发时间 | ⚡ 分钟级（即装即用） | 🐢 小时到天级 |
| 维护成本 | ✅ 社区维护 | ❌ 自己维护 |
| 安全性 | ✅ 经过社区审查 | ⚠️ 需要自己保证 |
| 功能完整性 | ✅ 功能丰富 | ⚠️ 需要逐步完善 |
| 文档支持 | ✅ 完善的文档 | ❌ 需要自己编写 |
| 标准化 | ✅ 遵循 MCP 协议 | ❌ 可能不兼容 |
| 更新频率 | ✅ 持续更新 | ⚠️ 依赖自己 |

**结论：** 使用 MCP 服务器可以节省 80% 以上的开发时间！

---

## 优势总结

### 1. **E2B MCP Server - 完美匹配！**
你的项目已经在使用 E2B 执行代码，现在可以直接使用 E2B 的 MCP 服务器：
- ✅ 无缝集成现有基础设施
- ✅ 相同的安全沙箱环境
- ✅ 不需要额外配置

### 2. **Filesystem Server - 官方支持**
- ✅ Anthropic/OpenAI 官方维护
- ✅ 经过严格的安全审计
- ✅ 支持细粒度权限控制

### 3. **Shell MCP - 多种选择**
- ✅ 社区有多个成熟实现
- ✅ 不同的安全策略可选
- ✅ 灵活的白名单配置

---

## 备选方案

如果 OpenAI Realtime API 不可用或不适合，可以考虑：

### 方案 B：LiveKit Agents + MCP
- 更灵活的提供商选择
- 可自部署
- 开源免费
- 同样支持 MCP 工具

### 方案 C：传统管道（STT + LLM + TTS）
- 使用 Whisper (STT) + Claude/GPT (LLM) + ElevenLabs (TTS)
- 更高的延迟，但更灵活
- 可以混合使用不同提供商
- 仍可使用 MCP 工具

---

## 推荐的实现路线图

### 阶段 1：最小可行产品（1-2 天）
- [ ] 安装 OpenAI Realtime API 依赖
- [ ] 安装 3 个核心 MCP 服务器
- [ ] 创建基础语音 UI 组件
- [ ] 实现 WebSocket 连接
- [ ] 测试基本的语音对话

### 阶段 2：工具集成（1 天）
- [ ] 配置 Filesystem MCP 服务器
- [ ] 配置 E2B MCP 服务器
- [ ] 配置 Shell MCP 服务器
- [ ] 测试文件操作
- [ ] 测试代码执行
- [ ] 测试命令执行

### 阶段 3：增强和优化（2-3 天）
- [ ] 添加音频可视化
- [ ] 实现对话历史
- [ ] 添加错误处理
- [ ] 优化延迟
- [ ] 安全测试
- [ ] 用户界面优化

**总计：4-6 天完成全功能语音助手！**

（相比从零实现需要 2-3 周，节省 70% 时间）

---

## 下一步行动

### 选择 A：立即开始实现 ✨（推荐）
我可以帮你：
1. 安装所有必要的 MCP 服务器
2. 创建语音 UI 组件
3. 实现 WebSocket API 路由
4. 配置 OpenAI Realtime API + MCP

### 选择 B：深入了解某个工具
- E2B MCP Server 的详细配置
- 各种 Shell MCP 的安全对比
- Filesystem Server 的权限设置

### 选择 C：调整方案
- 使用 LiveKit 代替 OpenAI
- 添加其他 MCP 服务器（Git、Fetch 等）
- 自定义安全策略
