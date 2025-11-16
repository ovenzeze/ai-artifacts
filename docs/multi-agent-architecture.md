# 多级 Agent 架构设计 - AI 语音助手

## 核心理念

**责任分离的多级架构**：语音对话层专注于理解用户意图，执行层专注于实际操作。

```
┌─────────────────────────────────────────────────────────────┐
│                        用户                                  │
│                    (语音交互)                                │
└─────────────────────────────────────────────────────────────┘
                              ↕
┌─────────────────────────────────────────────────────────────┐
│              对话 Agent (Conversation Layer)                 │
│                                                              │
│  职责：                                                       │
│  • 接收用户语音输入                                           │
│  • 理解用户意图                                              │
│  • 收集必要的上下文信息                                       │
│  • 补充背景知识                                              │
│  • 将意图转化为结构化命令                                     │
│  • 解释执行结果                                              │
│  • 生成语音回复                                              │
│                                                              │
│  技术栈：                                                     │
│  • OpenAI Realtime API (语音 I/O)                           │
│  • GPT-4o Realtime (对话理解)                               │
│  • MCP Servers (获取上下文)                                  │
└─────────────────────────────────────────────────────────────┘
                              ↕
                    结构化命令/响应协议
                      (JSON Schema)
                              ↕
┌─────────────────────────────────────────────────────────────┐
│              执行 Agent (Execution Layer)                    │
│                                                              │
│  职责：                                                       │
│  • 接收结构化命令                                            │
│  • 执行文件系统操作                                          │
│  • 运行代码和脚本                                            │
│  • 操作系统调用                                              │
│  • 浏览器自动化                                              │
│  • 应用程序集成                                              │
│  • 返回结构化执行结果                                         │
│                                                              │
│  技术选择：                                                   │
│  • Option A: Open Interpreter                               │
│  • Option B: MCP Servers (直接调用)                         │
│  • Option C: 自定义执行引擎                                  │
└─────────────────────────────────────────────────────────────┘
                              ↕
┌─────────────────────────────────────────────────────────────┐
│                    本地系统资源                               │
│  • 文件系统                                                  │
│  • 终端/Shell                                               │
│  • 应用程序 (浏览器、邮件、日历等)                            │
│  • 系统 API                                                  │
└─────────────────────────────────────────────────────────────┘
```

## 数据流设计

### 1. 用户请求流程

```
用户语音输入: "帮我查看今天的日程，然后给 John 发邮件确认会议时间"
    ↓
[对话 Agent 处理]
  1. 语音转文本 (Realtime API)
  2. 意图理解：
     - 查看日历
     - 发送邮件
  3. 上下文补充：
     - 今天的日期
     - John 的邮箱地址
     - 用户的邮件签名
  4. 生成结构化命令：
     {
       "task_id": "task_001",
       "commands": [
         {
           "action": "get_calendar_events",
           "params": { "date": "2025-11-16" }
         },
         {
           "action": "send_email",
           "params": {
             "to": "john@example.com",
             "subject": "会议时间确认",
             "body": "待日历查询后生成"
           }
         }
       ]
     }
    ↓
[执行 Agent 处理]
  1. 解析命令
  2. 执行操作
  3. 返回结构化结果：
     {
       "task_id": "task_001",
       "results": [
         {
           "action": "get_calendar_events",
           "status": "success",
           "data": {
             "events": [
               {"time": "10:00", "title": "团队会议"},
               {"time": "14:00", "title": "与 John 的 1on1"}
             ]
           }
         },
         {
           "action": "send_email",
           "status": "success",
           "message_id": "msg_12345"
         }
       ]
     }
    ↓
[对话 Agent 解释]
  1. 解析执行结果
  2. 生成自然语言回复
  3. 文本转语音
    ↓
语音输出: "我看到你今天有两个会议，10点的团队会议和下午2点与 John 的 1on1。
          我已经给 John 发送了确认邮件。"
```

## 协议定义

### 命令协议 (Command Protocol)

```typescript
interface Command {
  task_id: string;              // 任务唯一标识
  commands: Action[];           // 要执行的操作列表
  context?: Record<string, any>; // 可选的上下文信息
}

interface Action {
  action: string;               // 操作类型
  params: Record<string, any>;  // 操作参数
  requires_confirmation?: boolean; // 是否需要用户确认
}
```

### 响应协议 (Response Protocol)

```typescript
interface Response {
  task_id: string;              // 对应的任务 ID
  results: ActionResult[];      // 执行结果列表
  overall_status: 'success' | 'partial' | 'failed';
}

interface ActionResult {
  action: string;               // 对应的操作类型
  status: 'success' | 'failed' | 'pending';
  data?: any;                   // 成功时的返回数据
  error?: {                     // 失败时的错误信息
    code: string;
    message: string;
  };
}
```

## 执行层选择

### Option A: Open Interpreter

**优势：**
- ✅ 成熟的代码执行引擎
- ✅ 支持多种编程语言
- ✅ 已有安全沙箱机制
- ✅ Python API 易于集成

**集成方式：**

```python
# 后端服务 (FastAPI)
from interpreter import interpreter
from fastapi import FastAPI, WebSocket

app = FastAPI()

@app.websocket("/execute")
async def execute_commands(websocket: WebSocket):
    await websocket.accept()

    while True:
        # 接收对话 Agent 的命令
        command = await websocket.receive_json()

        # 使用 Open Interpreter 执行
        result = interpreter.chat(command['params']['prompt'],
                                  stream=False,
                                  display=False)

        # 返回结构化结果
        response = {
            "task_id": command['task_id'],
            "status": "success" if result else "failed",
            "data": result
        }
        await websocket.send_json(response)
```

**劣势：**
- ⚠️ 可能过于重量级（包含 LLM）
- ⚠️ 需要额外的进程管理

---

### Option B: MCP Servers (直接调用)

**优势：**
- ✅ 轻量级，专注单一功能
- ✅ 标准化协议
- ✅ 社区维护
- ✅ 与 OpenAI Realtime API 无缝集成

**集成方式：**

对话 Agent 直接调用 MCP 工具，无需额外的执行层。

```typescript
// OpenAI Realtime API 配置
await client.updateSession({
  tools: [
    {
      type: "function",
      name: "read_file",
      description: "读取文件内容",
      parameters: { /* ... */ }
    },
    {
      type: "function",
      name: "execute_command",
      description: "执行终端命令",
      parameters: { /* ... */ }
    }
  ]
});

// Realtime API 会自动调用工具并获取结果
```

**劣势：**
- ⚠️ 需要为每个功能配置 MCP 服务器
- ⚠️ 复杂操作需要组合多个工具

---

### Option C: 自定义执行引擎

**优势：**
- ✅ 完全控制
- ✅ 针对性优化
- ✅ 安全策略可定制

**劣势：**
- ❌ 开发工作量大
- ❌ 需要自己维护
- ❌ 不推荐（违背不重复造轮子原则）

---

## 推荐方案

### 混合架构：Realtime API + MCP Servers + Open Interpreter

```
对话 Agent (OpenAI Realtime API)
    ↓
决策层：根据任务类型选择执行方式
    ↓
┌─────────────────┬─────────────────┬──────────────────┐
│  简单操作        │  复杂任务        │  代码执行         │
│  MCP Servers    │  Open Interpreter│  Open Interpreter│
└─────────────────┴─────────────────┴──────────────────┘
    ↓                   ↓                   ↓
  本地文件系统        系统操作           Python/JS 代码
```

**决策规则：**
- **简单文件操作** → 直接用 MCP Filesystem Server
- **Shell 命令** → 用 MCP Shell Server
- **需要推理的复杂任务** → 委托给 Open Interpreter
- **代码生成和执行** → 委托给 Open Interpreter

## 实现示例

### 对话 Agent (TypeScript/Next.js)

```typescript
// app/api/voice/route.ts
import { RealtimeClient } from '@openai/realtime-api-beta';
import { WebSocket } from 'ws';

export async function GET(req: Request) {
  // 创建 Realtime 客户端
  const client = new RealtimeClient({
    apiKey: process.env.OPENAI_API_KEY,
    model: 'gpt-4o-realtime-preview',
  });

  // 配置系统提示
  await client.updateSession({
    instructions: `
      你是一个本地虚拟助手，可以帮助用户操作电脑。

      你的能力包括：
      - 文件系统操作（读写文件、目录管理）
      - 执行终端命令
      - 浏览器自动化
      - 邮件和日历管理

      对于简单操作，直接使用可用的工具。
      对于复杂任务，将任务发送给执行引擎。
    `,
    tools: [
      // MCP 工具定义
      {
        type: 'function',
        name: 'delegate_to_executor',
        description: '将复杂任务委托给执行引擎',
        parameters: {
          type: 'object',
          properties: {
            task_description: { type: 'string' },
            requires_code: { type: 'boolean' }
          }
        }
      }
    ]
  });

  // 连接到执行引擎
  const executorWs = new WebSocket('ws://localhost:8765/execute');

  // 处理工具调用
  client.on('function_call', async (event) => {
    if (event.name === 'delegate_to_executor') {
      // 发送到执行引擎
      executorWs.send(JSON.stringify({
        task_id: generateTaskId(),
        prompt: event.parameters.task_description
      }));

      // 等待结果
      const result = await waitForExecutorResponse(executorWs);

      // 返回给 Realtime API
      await client.sendToolResponse({
        tool_call_id: event.id,
        output: JSON.stringify(result)
      });
    }
  });

  return new Response(/* WebSocket upgrade */);
}
```

### 执行引擎 (Python/FastAPI)

```python
# executor_service.py
from fastapi import FastAPI, WebSocket
from interpreter import interpreter
import asyncio

app = FastAPI()

# 配置 Open Interpreter
interpreter.auto_run = False  # 需要确认
interpreter.llm.model = "gpt-4"

@app.websocket("/execute")
async def execute_endpoint(websocket: WebSocket):
    await websocket.accept()

    try:
        while True:
            # 接收命令
            data = await websocket.receive_json()
            task_id = data['task_id']
            prompt = data['prompt']

            # 使用 Open Interpreter 执行
            messages = interpreter.chat(prompt, stream=False, display=False)

            # 提取结果
            result = {
                "task_id": task_id,
                "status": "success",
                "output": extract_output(messages),
                "code_executed": extract_code(messages)
            }

            # 返回结果
            await websocket.send_json(result)

    except Exception as e:
        await websocket.send_json({
            "status": "error",
            "error": str(e)
        })

def extract_output(messages):
    """从 interpreter 消息中提取输出"""
    outputs = []
    for msg in messages:
        if msg.get('type') == 'code':
            outputs.append({
                'language': msg.get('format'),
                'code': msg.get('content')
            })
        elif msg.get('type') == 'console':
            outputs.append({
                'type': 'output',
                'content': msg.get('content')
            })
    return outputs

def extract_code(messages):
    """提取执行的代码"""
    code_blocks = []
    for msg in messages:
        if msg.get('type') == 'code':
            code_blocks.append(msg.get('content'))
    return code_blocks

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8765)
```

## 本地系统集成能力

### 可用的 MCP Servers

| 功能 | MCP Server | 能力 |
|------|------------|------|
| 📁 文件系统 | `@modelcontextprotocol/server-filesystem` | 读写文件、目录操作 |
| 💻 Shell 命令 | `cmd-line-mcp` 或 `shell-mcp` | 执行终端命令 |
| 🌐 浏览器 | `browsermcp/mcp` | 自动化浏览器操作 |
| 📧 Gmail | `gmail-mcp-server` | 发送/读取邮件 |
| 📅 Google Calendar | `google-calendar-mcp` | 管理日程 |
| 💬 Slack | `slack-mcp-server` | 发送消息、管理频道 |
| 📱 WhatsApp | `wweb-mcp` | WhatsApp 自动化 |
| 🔧 Git | `@modelcontextprotocol/server-git` | Git 操作 |

### 完整能力示例

```typescript
// 用户说："帮我整理今天的工作"

// 对话 Agent 理解为：
{
  "intent": "daily_work_summary",
  "sub_tasks": [
    "查看今天的日历事件",
    "检查 Slack 未读消息",
    "查看 Gmail 重要邮件",
    "生成工作总结",
    "创建明日待办清单"
  ]
}

// 执行流程：
1. 调用 Google Calendar MCP → 获取今日事件
2. 调用 Slack MCP → 获取未读消息
3. 调用 Gmail MCP → 获取重要邮件
4. 委托给 Open Interpreter → 生成总结报告
5. 调用 Filesystem MCP → 保存到文件

// 语音回复用户：
"我已经帮你整理好了今天的工作。你今天有3个会议，
 Slack 有12条未读消息，主要关于项目进展，
 Gmail 有2封重要邮件需要回复。
 我已经生成了工作总结并保存在 daily_summary.md 文件中。"
```

## 安全和权限管理

### 1. 分级授权

```typescript
interface PermissionLevel {
  // Level 1: 无需确认 (只读操作)
  safe_read: string[];  // ['read_file', 'list_directory', 'get_calendar']

  // Level 2: 需要确认 (写操作)
  confirm_write: string[];  // ['write_file', 'send_email', 'execute_command']

  // Level 3: 危险操作 (需要明确授权)
  dangerous: string[];  // ['delete_file', 'system_command', 'install_software']
}
```

### 2. 确认流程

```
用户: "删除所有临时文件"
    ↓
对话 Agent: 识别为危险操作
    ↓
语音确认: "我找到了25个临时文件，总共150MB，确认要全部删除吗？"
    ↓
用户: "确认"
    ↓
执行删除操作
```

## 优势总结

### 与单层架构对比

| 特性 | 多级架构 | 单层架构 |
|------|---------|---------|
| **责任分离** | ✅ 清晰 | ❌ 混杂 |
| **可维护性** | ✅ 高 | ⚠️ 中等 |
| **可扩展性** | ✅ 容易添加新能力 | ⚠️ 修改困难 |
| **安全性** | ✅ 分层控制 | ⚠️ 难以管理 |
| **测试** | ✅ 独立测试各层 | ❌ 集成测试复杂 |
| **性能** | ⚠️ 多一层通信 | ✅ 直接执行 |

### 关键优势

1. **对话 Agent 专注于理解**：不需要知道如何执行，只需要理解用户意图
2. **执行层可替换**：可以切换不同的执行引擎而不影响对话层
3. **更好的错误处理**：执行失败时，对话 Agent 可以用自然语言解释并提供建议
4. **上下文管理**：对话层维护用户上下文，执行层无状态

## 下一步

### 阶段 1：搭建基础架构（2-3 天）
- [ ] 设置 OpenAI Realtime API
- [ ] 创建语音 UI 组件
- [ ] 搭建执行引擎服务 (FastAPI + Open Interpreter)
- [ ] 定义命令/响应协议
- [ ] 实现 WebSocket 通信

### 阶段 2：集成 MCP Servers（1-2 天）
- [ ] 集成 Filesystem MCP
- [ ] 集成 Shell MCP
- [ ] 集成 Browser MCP
- [ ] 测试基本操作

### 阶段 3：高级功能（2-3 天）
- [ ] 集成 Gmail/Calendar MCP
- [ ] 实现权限管理
- [ ] 添加确认机制
- [ ] 优化错误处理
- [ ] 添加对话历史

**总计：5-8 天完成全功能本地虚拟助手**
