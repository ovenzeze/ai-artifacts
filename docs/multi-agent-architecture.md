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

## 协议定义（任务协作模式）

### 核心理念

**不要把执行 Agent 当作 API，而是当作智能同事**

- ❌ 不要：严格定义每个函数调用和参数
- ✅ 应该：描述任务目标，提供背景信息，让执行 Agent 自主决定如何完成

### 任务协议 (Task Protocol)

```typescript
interface Task {
  task_id: string;              // 任务唯一标识
  description: string;          // 任务描述（自然语言）
  context?: {                   // 背景信息
    [key: string]: any;
  };
  constraints?: {               // 约束条件
    max_duration?: number;      // 最大执行时间（秒）
    require_confirmation?: boolean; // 是否需要确认
    parallel_subtasks?: boolean;    // 是否可以并行执行子任务
  };
  priority?: 'low' | 'normal' | 'high'; // 优先级
}
```

**示例：**

```json
{
  "task_id": "task_001",
  "description": "查看今天的日程，找出所有会议并整理成列表。如果有和 John 的会议，需要特别标注。",
  "context": {
    "today": "2025-11-16",
    "user_timezone": "Asia/Shanghai",
    "user_preferences": {
      "calendar_app": "Google Calendar",
      "highlight_contacts": ["John", "Mary"]
    }
  },
  "constraints": {
    "max_duration": 30,
    "require_confirmation": false,
    "parallel_subtasks": true
  }
}
```

### 响应协议 (Response Protocol)

```typescript
interface TaskResponse {
  task_id: string;              // 对应的任务 ID
  status: 'success' | 'partial' | 'failed' | 'pending';
  summary: string;              // 执行摘要（自然语言）
  result?: any;                 // 结构化结果（可选）
  steps_taken?: string[];       // 执行步骤记录（用于调试）
  error?: {
    message: string;
    recoverable: boolean;       // 是否可以恢复
    suggestion?: string;        // 建议的下一步操作
  };
  execution_time?: number;      // 执行时间（秒）
}
```

**示例：**

```json
{
  "task_id": "task_001",
  "status": "success",
  "summary": "找到今天3个会议：10:00 团队站会、14:00 与 John 的 1on1（已标注）、16:00 项目评审",
  "result": {
    "meetings": [
      {
        "time": "10:00",
        "title": "团队站会",
        "duration": "30min",
        "highlighted": false
      },
      {
        "time": "14:00",
        "title": "与 John 的 1on1",
        "duration": "60min",
        "highlighted": true
      },
      {
        "time": "16:00",
        "title": "项目评审",
        "duration": "90min",
        "highlighted": false
      }
    ]
  },
  "steps_taken": [
    "连接 Google Calendar API",
    "查询今天的所有事件",
    "过滤会议类型",
    "检查参与人中是否包含 John",
    "格式化输出"
  ],
  "execution_time": 2.3
}
```

### 并行任务协议

对于可以并行的任务，对话 Agent 可以发送任务组：

```typescript
interface TaskBatch {
  batch_id: string;
  tasks: Task[];                // 多个独立任务
  execution_mode: 'parallel' | 'sequential';
  dependencies?: {              // 任务依赖关系（可选）
    [task_id: string]: string[]; // 依赖的其他任务 ID
  };
}
```

**示例：并行执行**

```json
{
  "batch_id": "batch_001",
  "execution_mode": "parallel",
  "tasks": [
    {
      "task_id": "task_001",
      "description": "查看今天的 Gmail 重要邮件"
    },
    {
      "task_id": "task_002",
      "description": "查看 Slack 的未读消息数量"
    },
    {
      "task_id": "task_003",
      "description": "检查项目代码是否有编译错误"
    }
  ]
}
```

**示例：带依赖关系**

```json
{
  "batch_id": "batch_002",
  "execution_mode": "parallel",
  "tasks": [
    {
      "task_id": "task_004",
      "description": "读取 package.json 文件内容"
    },
    {
      "task_id": "task_005",
      "description": "根据 package.json 中的依赖，生成依赖更新报告"
    }
  ],
  "dependencies": {
    "task_005": ["task_004"]  // task_005 依赖 task_004
  }
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
import asyncio
import time

app = FastAPI()

@app.websocket("/execute")
async def execute_tasks(websocket: WebSocket):
    await websocket.accept()

    while True:
        # 接收任务（可能是单个任务或任务批次）
        data = await websocket.receive_json()

        if 'batch_id' in data:
            # 批量任务处理
            await handle_task_batch(websocket, data)
        else:
            # 单个任务处理
            await handle_single_task(websocket, data)


async def handle_single_task(websocket: WebSocket, task: dict):
    """处理单个任务 - 用自然语言描述，让 Open Interpreter 自主执行"""
    task_id = task['task_id']
    description = task['description']
    context = task.get('context', {})
    constraints = task.get('constraints', {})

    # 构建给执行 Agent 的提示词
    # 注意：这里不是严格的命令，而是任务描述
    prompt = f"""
任务描述：{description}

背景信息：
{format_context(context)}

约束条件：
- 最大执行时间：{constraints.get('max_duration', 60)} 秒
- 需要确认：{'是' if constraints.get('require_confirmation') else '否'}

请完成这个任务，并在最后总结你的执行步骤和结果。
"""

    start_time = time.time()

    try:
        # 让 Open Interpreter 自主执行
        messages = interpreter.chat(prompt, stream=False, display=False)

        # 提取结果和步骤
        response = {
            "task_id": task_id,
            "status": "success",
            "summary": extract_summary(messages),
            "result": extract_result(messages),
            "steps_taken": extract_steps(messages),
            "execution_time": time.time() - start_time
        }
    except Exception as e:
        response = {
            "task_id": task_id,
            "status": "failed",
            "summary": f"任务执行失败：{str(e)}",
            "error": {
                "message": str(e),
                "recoverable": True,
                "suggestion": "可以尝试简化任务或提供更多上下文信息"
            },
            "execution_time": time.time() - start_time
        }

    await websocket.send_json(response)


async def handle_task_batch(websocket: WebSocket, batch: dict):
    """处理批量任务 - 支持并行执行"""
    batch_id = batch['batch_id']
    tasks = batch['tasks']
    execution_mode = batch.get('execution_mode', 'sequential')
    dependencies = batch.get('dependencies', {})

    if execution_mode == 'parallel' and not dependencies:
        # 并行执行所有任务
        results = await asyncio.gather(*[
            execute_task_async(task) for task in tasks
        ])

        # 返回所有结果
        for result in results:
            await websocket.send_json(result)

    elif dependencies:
        # 有依赖关系的并行执行
        results = await execute_with_dependencies(tasks, dependencies)
        for result in results:
            await websocket.send_json(result)
    else:
        # 顺序执行
        for task in tasks:
            await handle_single_task(websocket, task)


async def execute_task_async(task: dict):
    """异步执行单个任务"""
    # 将同步的 interpreter.chat 包装为异步
    loop = asyncio.get_event_loop()

    prompt = build_task_prompt(task)
    messages = await loop.run_in_executor(
        None,
        lambda: interpreter.chat(prompt, stream=False, display=False)
    )

    return {
        "task_id": task['task_id'],
        "status": "success",
        "summary": extract_summary(messages),
        "result": extract_result(messages)
    }


def format_context(context: dict) -> str:
    """格式化上下文信息为自然语言"""
    lines = []
    for key, value in context.items():
        lines.append(f"- {key}: {value}")
    return "\n".join(lines)


def extract_summary(messages: list) -> str:
    """从消息中提取执行摘要"""
    # Open Interpreter 的最后一条消息通常是总结
    for msg in reversed(messages):
        if msg.get('role') == 'assistant' and msg.get('type') == 'message':
            return msg.get('content', '')
    return "任务已完成"


def extract_steps(messages: list) -> list:
    """提取执行步骤"""
    steps = []
    for msg in messages:
        if msg.get('type') == 'code':
            steps.append(f"执行 {msg.get('format')} 代码")
    return steps


def extract_result(messages: list) -> dict:
    """提取结构化结果（如果有）"""
    # 可以从 console 输出或消息内容中提取
    # 这里简化处理，实际可以更复杂
    return {"raw_output": messages}


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8765)
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

  // 配置系统提示 - 强调任务协作模式
  await client.updateSession({
    instructions: `
      你是一个本地虚拟助手，可以帮助用户操作电脑。

      你的工作方式：
      1. 理解用户的意图和需求
      2. 收集相关的背景信息（时间、用户偏好等）
      3. 将任务委托给执行引擎（用自然语言描述任务即可）
      4. 解释执行结果并与用户对话

      重要原则：
      - 不要尝试严格定义每个步骤，执行引擎很智能
      - 专注于任务的"目标"而不是"步骤"
      - 识别可以并行执行的独立任务
      - 对于耗时任务，要分解成小任务避免超时
      - 提供充分的背景信息帮助执行引擎理解上下文

      可用工具：
      - delegate_task: 委托单个任务给执行引擎
      - delegate_tasks: 委托多个任务（可并行）
    `,
    tools: [
      {
        type: 'function',
        name: 'delegate_task',
        description: '将任务委托给执行引擎。用自然语言描述任务目标，不需要详细步骤。',
        parameters: {
          type: 'object',
          properties: {
            description: {
              type: 'string',
              description: '任务描述（自然语言），说明要做什么、为什么做'
            },
            context: {
              type: 'object',
              description: '背景信息（如时间、用户偏好、相关数据等）'
            },
            max_duration: {
              type: 'number',
              description: '最大执行时间（秒），避免任务过大'
            }
          },
          required: ['description']
        }
      },
      {
        type: 'function',
        name: 'delegate_tasks_parallel',
        description: '并行执行多个独立任务，提高效率',
        parameters: {
          type: 'object',
          properties: {
            tasks: {
              type: 'array',
              items: {
                type: 'object',
                properties: {
                  description: { type: 'string' },
                  context: { type: 'object' }
                }
              }
            }
          }
        }
      }
    ]
  });

  // 连接到执行引擎
  const executorWs = new WebSocket('ws://localhost:8765/execute');

  // 处理工具调用
  client.on('function_call', async (event) => {
    if (event.name === 'delegate_task') {
      // 单任务委托
      const task = {
        task_id: generateTaskId(),
        description: event.parameters.description,
        context: event.parameters.context || {},
        constraints: {
          max_duration: event.parameters.max_duration || 60
        }
      };

      executorWs.send(JSON.stringify(task));
      const result = await waitForExecutorResponse(executorWs);

      await client.sendToolResponse({
        tool_call_id: event.id,
        output: result.summary  // 返回自然语言摘要
      });
    }

    if (event.name === 'delegate_tasks_parallel') {
      // 并行任务委托
      const batch = {
        batch_id: generateBatchId(),
        execution_mode: 'parallel',
        tasks: event.parameters.tasks.map(t => ({
          task_id: generateTaskId(),
          description: t.description,
          context: t.context || {}
        }))
      };

      executorWs.send(JSON.stringify(batch));

      // 等待所有任务完成
      const results = await waitForBatchResults(executorWs, batch.tasks.length);

      // 合并结果摘要
      const combinedSummary = results.map(r => r.summary).join('\n');

      await client.sendToolResponse({
        tool_call_id: event.id,
        output: combinedSummary
      });
    }
  });

  return new Response(/* WebSocket upgrade */);
}

// 辅助函数
function generateTaskId(): string {
  return `task_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
}

function generateBatchId(): string {
  return `batch_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
}

async function waitForExecutorResponse(ws: WebSocket): Promise<any> {
  return new Promise((resolve) => {
    ws.once('message', (data) => {
      resolve(JSON.parse(data.toString()));
    });
  });
}

async function waitForBatchResults(ws: WebSocket, count: number): Promise<any[]> {
  return new Promise((resolve) => {
    const results: any[] = [];
    const handler = (data: any) => {
      results.push(JSON.parse(data.toString()));
      if (results.length === count) {
        ws.off('message', handler);
        resolve(results);
      }
    };
    ws.on('message', handler);
  });
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

### 完整能力示例：任务协作模式

### 示例 1：智能工作总结（并行执行）

```
用户语音：「帮我整理今天的工作」

↓ 对话 Agent 处理 ↓

1. 理解意图：用户想要今天的工作总结
2. 识别可并行的独立任务
3. 委托给执行引擎

发送的任务批次（注意：是自然语言描述，不是 API 调用！）：
{
  "batch_id": "batch_001",
  "execution_mode": "parallel",
  "tasks": [
    {
      "task_id": "task_001",
      "description": "查看今天的日历，列出所有会议的时间、标题和参与人",
      "context": {
        "date": "2025-11-16",
        "calendar_app": "Google Calendar"
      }
    },
    {
      "task_id": "task_002",
      "description": "检查 Slack 的未读消息，总结主要话题和需要回复的人",
      "context": {
        "workspace": "my-company"
      }
    },
    {
      "task_id": "task_003",
      "description": "查看 Gmail 今天收到的重要邮件（标星或来自老板的），列出发件人和主题",
      "context": {
        "email": "user@company.com",
        "filter": "is:important OR is:starred"
      }
    }
  ]
}

↓ 执行 Agent 并行处理（Open Interpreter 自主决定怎么做）↓

Task 001 执行：
- 自主选择使用 Google Calendar API 或 MCP Server
- 查询今天的事件
- 格式化输出

Task 002 执行：
- 连接 Slack
- 获取未读消息
- 分析话题

Task 003 执行：
- 连接 Gmail
- 应用过滤器
- 提取关键信息

↓ 返回结果 ↓

{
  "task_id": "task_001",
  "summary": "今天有3个会议：10:00团队站会(30min)、14:00与John的1on1(60min)、16:00项目评审(90min)"
}

{
  "task_id": "task_002",
  "summary": "Slack有12条未读消息，主要话题：新功能开发进度(5条)、Bug修复讨论(4条)、代码审查(3条)。需要回复：@Sarah关于API设计"
}

{
  "task_id": "task_003",
  "summary": "2封重要邮件：1. 老板Alice关于Q4目标讨论；2. 客户Beta Corp关于功能需求"
}

↓ 对话 Agent 整合并回复 ↓

语音输出：
「我已经帮你整理好了今天的工作：

📅 日程：你今天有3个会议，包括下午2点和John的1对1，以及4点的项目评审。

💬 Slack：有12条未读消息，主要讨论新功能开发和Bug修复，Sarah在等你回复关于API设计的问题。

📧 邮件：有2封重要邮件需要处理，一封是Alice关于Q4目标，另一封是Beta Corp的功能需求。

需要我帮你处理其中哪一项吗？」
```

### 示例 2：代码生成任务（给予自由度）

```
用户语音：「帮我写一个Python脚本，分析这个项目的代码复杂度，生成报告」

↓ 对话 Agent 处理 ↓

发送给执行 Agent（注意描述的灵活性）：
{
  "task_id": "task_004",
  "description": `
    编写一个 Python 脚本分析代码复杂度。

    需求：
    - 扫描当前项目的 Python 文件
    - 计算圈复杂度（Cyclomatic Complexity）
    - 找出最复杂的10个函数
    - 生成 Markdown 格式的报告

    可以使用任何你认为合适的工具和库。
    报告保存为 complexity_report.md
  `,
  "context": {
    "project_path": "/home/user/ai-artifacts",
    "programming_language": "Python"
  },
  "constraints": {
    "max_duration": 120
  }
}

↓ 执行 Agent 自主决定 ↓

Open Interpreter 自己决定：
1. 使用 radon 库（自动安装 pip install radon）
2. 编写脚本扫描文件
3. 解析结果
4. 生成报告

返回：
{
  "task_id": "task_004",
  "status": "success",
  "summary": "已生成代码复杂度报告，发现3个高复杂度函数需要重构",
  "result": {
    "total_files": 45,
    "total_functions": 312,
    "high_complexity_count": 3,
    "report_path": "complexity_report.md"
  },
  "steps_taken": [
    "安装 radon 库",
    "扫描项目 Python 文件",
    "计算圈复杂度",
    "排序并筛选",
    "生成 Markdown 报告"
  ]
}

↓ 语音回复 ↓

「分析完成！我扫描了45个Python文件，发现有3个函数复杂度较高需要重构。
 详细报告已保存在 complexity_report.md。
 要我帮你打开看看吗？」
```

### 示例 3：任务依赖（自动处理）

```
用户：「查看最新的销售数据Excel，生成趋势图」

↓ 对话 Agent 识别依赖关系 ↓

{
  "batch_id": "batch_003",
  "execution_mode": "parallel",
  "tasks": [
    {
      "task_id": "task_005",
      "description": "找到最新的销售数据Excel文件（应该在 ~/Documents/Sales/ 目录）",
      "context": {
        "search_path": "~/Documents/Sales",
        "file_pattern": "*.xlsx"
      }
    },
    {
      "task_id": "task_006",
      "description": "读取销售数据，生成月度销售趋势图（折线图），保存为PNG",
      "context": {
        "data_source": "待 task_005 提供"
      }
    }
  ],
  "dependencies": {
    "task_006": ["task_005"]
  }
}

↓ 执行引擎智能处理依赖 ↓

1. 先执行 task_005 → 找到 sales_2025_Q4.xlsx
2. 将结果传递给 task_006
3. task_006 使用 pandas + matplotlib 生成图表

语音输出：
「我找到了最新的销售数据文件（Q4销售数据），已生成趋势图保存为 sales_trend.png。
 从图上看，11月的销售额比10月增长了15%！」
```

### 对比：传统接口调用 vs 任务协作

**❌ 传统方式（过于严格）：**
```json
{
  "commands": [
    { "action": "list_directory", "params": { "path": "~/Documents/Sales" } },
    { "action": "filter_files", "params": { "pattern": "*.xlsx" } },
    { "action": "sort_by_date", "params": { "order": "desc" } },
    { "action": "get_first", "params": {} },
    { "action": "read_excel", "params": { "file": "..." } },
    { "action": "create_dataframe", "params": { "data": "..." } },
    { "action": "plot_line_chart", "params": { "x": "month", "y": "sales" } },
    { "action": "save_image", "params": { "filename": "trend.png" } }
  ]
}
```
问题：
- 太死板，需要预定义每一步
- 执行 Agent 没有自主权
- 如果路径不对，无法灵活调整

**✅ 任务协作方式（优雅灵活）：**
```json
{
  "description": "找到最新的销售Excel文件并生成趋势图",
  "context": {
    "likely_location": "~/Documents/Sales"
  }
}
```
优势：
- 简洁明了
- 执行 Agent 可以自主决定最佳路径
- 如果目录不对，可以搜索其他位置
- 如果没有合适的库，可以自己安装
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
