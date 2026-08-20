# 本地 LiteRT-LM + OpenAI API 使用流程

> 今日成功记录：2026-08-20/21
>
> 目标：在 Android + Termux 上运行 LiteRT-LM 模型，并通过一个本地 Node.js MJS 代理，让现成的 OpenAI-compatible 聊天客户端访问本地模型。

## 1. 整体结构

```text
现成聊天客户端 APK
        │
        │ OpenAI-compatible API
        ▼
127.0.0.1:5000
        │
        │ Node.js server.mjs
        │ - API Key
        │ - /v1/models
        │ - /v1/chat/completions
        │ - /no_think（可选）
        │ - SSE 流式转发
        ▼
127.0.0.1:9379
        │
        │ LiteRT-LM OpenAI server
        ▼
本地模型
  ├─ Qwen3-0.6B.litertlm
  └─ gemma-4-E2B-it-gpu.litertlm
```

**两个端口都要开：**

- `9379`：LiteRT-LM 真正负责模型推理
- `5000`：自己的 Node.js OpenAI-compatible 代理，前端连这个

前端不要直接连 `9379`，一般统一使用：

```text
http://127.0.0.1:5000/v1
```

---

## 2. LiteRT-LM 模型管理

查看已经导入的模型：

```bash
litert-lm list
```

典型结果：

```text
Qwen3-0.6B.litertlm
 gemma-4-E2B-it-gpu.litertlm
```

导入 Hugging Face 上的 `.litertlm`：

```bash
litert-lm import "/完整路径/model.litertlm"
```

也可以直接从 Hugging Face 运行/下载：

```bash
litert-lm run \
  --from-huggingface-repo=litert-community/Qwen3-0.6B \
  --prompt="你好"
```

---

## 3. 测试模型本身

### Qwen3：GPU + 10K context + 关闭 thinking

```bash
litert-lm run Qwen3-0.6B.litertlm \
  --backend gpu \
  --max-num-tokens 10240 \
  --thinking=false \
  --prompt $'/no_think\n你好，请用一句话介绍你自己。'
```

实测：

- Qwen3 CPU + thinking：约 51.7 秒
- Qwen3 GPU + `/no_think`：约 9 秒
- Qwen3 GPU + `--max-num-tokens 10240`：成功

说明：`--max-num-tokens` 可以在 `litert-lm run` 中扩大 KV cache 的 token 容量；你自己的 `serve` 默认配置曾经只有 4096。

### Gemma：GPU + 10K context

```bash
litert-lm run gemma-4-E2B-it-gpu.litertlm \
  --backend gpu \
  --max-num-tokens 10240 \
  --prompt "你好"
```

也已实测成功。

---

## 4. 给 LiteRT-LM serve 设置 10K context

LiteRT-LM 的 `serve` CLI 没有直接暴露 `--max-num-tokens`，但它会读取 CLI 配置。

配置文件：

```text
~/.litert-lm/config.json
```

推荐按模型设置，不要全局污染所有模型：

```json
{
  "default": {},
  "models": {
    "Qwen3-0.6B.litertlm": {
      "max_num_tokens": 10240
    },
    "gemma-4-E2B-it-gpu.litertlm": {
      "max_num_tokens": 10240
    }
  }
}
```

检查配置是否生效：

```bash
python - <<'PY'
from litert_lm_cli import config

for model in [
    "Qwen3-0.6B.litertlm",
    "gemma-4-E2B-it-gpu.litertlm",
]:
    c = config.get_model_config(model)
    print(model, "=>", c.max_num_tokens)
PY
```

期望：

```text
Qwen3-0.6B.litertlm => 10240
gemma-4-E2B-it-gpu.litertlm => 10240
```

### 注意

`max_num_tokens` 和“模型永久修改了上下文窗口”不是完全一回事。它控制 LiteRT-LM Engine 的 KV cache / 最大 token 容量。实际是否能容纳某个请求，还要看模型 artifact 和 engine 的限制。

今天实测 `run --max-num-tokens 10240` 对 Qwen3 和 Gemma 都成功。

---

## 5. 启动 LiteRT-LM API Server

建议单独开一个 Termux Session：

```bash
litert-lm serve \
  --host 127.0.0.1 \
  --port 9379 \
  --verbose
```

保持这个窗口一直运行。

检查 API：

```bash
curl http://127.0.0.1:9379/v1/models
```

聊天测试：

```bash
curl -N http://127.0.0.1:9379/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen3-0.6B.litertlm",
    "messages": [
      {
        "role": "user",
        "content": "/no_think\n你好"
      }
    ],
    "stream": true
  }'
```

`-N` 用于让 curl 不缓冲 SSE 输出。

---

## 6. Node.js MJS 代理

### 作用

`server.mjs` 负责：

- 给现成聊天客户端提供 OpenAI-compatible API
- API Key 鉴权
- `/v1/models`
- `/v1/chat/completions`
- SSE 流式透传
- Qwen 自动加入 `/no_think`
- 根据需要把模型名变成 `模型名,gpu`
- 不让前端知道后面实际上是 LiteRT-LM

### 通用模板

```js
import http from "node:http";

const HOST = "127.0.0.1";
const PORT = 5000;
const BACKEND = "http://127.0.0.1:9379";
const API_KEY = process.env.LOCAL_API_KEY || "change-me";

const server = http.createServer(async (req, res) => {
  try {
    res.setHeader("Access-Control-Allow-Origin", "*");
    res.setHeader(
      "Access-Control-Allow-Headers",
      "Content-Type, Authorization"
    );
    res.setHeader(
      "Access-Control-Allow-Methods",
      "GET, POST, OPTIONS"
    );

    if (req.method === "OPTIONS") {
      res.writeHead(204);
      res.end();
      return;
    }

    const authorization = req.headers.authorization;

    if (authorization !== `Bearer ${API_KEY}`) {
      res.writeHead(401, {
        "Content-Type": "application/json",
      });
      res.end(JSON.stringify({
        error: {
          message: "Invalid API key",
          type: "authentication_error",
        },
      }));
      return;
    }

    // -------------------------
    // GET /v1/models
    // -------------------------
    if (req.method === "GET" && req.url === "/v1/models") {
      const response = await fetch(`${BACKEND}/v1/models`);
      const body = await response.text();

      res.writeHead(response.status, {
        "Content-Type":
          response.headers.get("content-type") || "application/json",
      });
      res.end(body);
      return;
    }

    // -------------------------
    // POST /v1/chat/completions
    // -------------------------
    if (
      req.method === "POST" &&
      req.url === "/v1/chat/completions"
    ) {
      const chunks = [];

      for await (const chunk of req) {
        chunks.push(chunk);
      }

      const body = Buffer.concat(chunks).toString("utf8");

      let requestData;
      try {
        requestData = JSON.parse(body);
      } catch {
        res.writeHead(400, {
          "Content-Type": "application/json",
        });
        res.end(JSON.stringify({
          error: {
            message: "Invalid JSON",
            type: "invalid_request_error",
          },
        }));
        return;
      }

      const originalModel = requestData.model;

      // -------------------------
      // Qwen-specific settings
      // -------------------------
      if (originalModel === "Qwen3-0.6B.litertlm") {
        // LiteRT-LM serve supports backend selection in model ID.
        requestData.model = "Qwen3-0.6B.litertlm,gpu";

        // Disable Qwen3 thinking via the supported control token.
        if (Array.isArray(requestData.messages)) {
          const firstUserIndex = requestData.messages.findIndex(
            (message) => message && message.role === "user"
          );

          if (firstUserIndex !== -1) {
            const message = requestData.messages[firstUserIndex];

            if (typeof message.content === "string") {
              const content = message.content;

              if (!content.trimStart().startsWith("/no_think")) {
                requestData.messages[firstUserIndex] = {
                  ...message,
                  content: `/no_think\n${content}`,
                };
              }
            }
          }
        }
      }

      // -------------------------
      // Streaming or normal JSON
      // -------------------------
      const isStream = requestData.stream === true;

      const response = await fetch(
        `${BACKEND}/v1/chat/completions`,
        {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
          },
          body: JSON.stringify(requestData),
        }
      );

      if (!response.ok) {
        const errorBody = await response.text();
        res.writeHead(response.status, {
          "Content-Type":
            response.headers.get("content-type") || "application/json",
        });
        res.end(errorBody);
        return;
      }

      // -------------------------
      // SSE streaming
      // -------------------------
      if (isStream) {
        res.writeHead(200, {
          "Content-Type": "text/event-stream",
          "Cache-Control": "no-cache, no-transform",
          "Connection": "keep-alive",
          "X-Accel-Buffering": "no",
        });

        if (response.body) {
          for await (const chunk of response.body) {
            res.write(chunk);
          }
        }

        res.end();
        return;
      }

      // -------------------------
      // Non-streaming
      // -------------------------
      const responseBody = await response.text();

      res.writeHead(200, {
        "Content-Type":
          response.headers.get("content-type") || "application/json",
      });
      res.end(responseBody);
      return;
    }

    // -------------------------
    // 404
    // -------------------------
    res.writeHead(404, {
      "Content-Type": "application/json",
    });
    res.end(JSON.stringify({
      error: {
        message: "Not Found",
        type: "invalid_request_error",
      },
    }));
  } catch (error) {
    console.error("[Proxy Error]", error);

    if (!res.headersSent) {
      res.writeHead(502, {
        "Content-Type": "application/json",
      });
    }

    res.end(JSON.stringify({
      error: {
        message: "LiteRT-LM backend unavailable",
        type: "server_error",
        details: error.message,
      },
    }));
  }
});

server.listen(PORT, HOST, () => {
  console.log("Local OpenAI API proxy started.");
  console.log(`API:     http://${HOST}:${PORT}/v1`);
  console.log(`Backend: ${BACKEND}/v1`);
});
```

启动：

```bash
cd ~/local-api
export LOCAL_API_KEY="sk-my-custom-key-123"
node server.mjs
```

如果你不想使用环境变量，也可以把：

```js
const API_KEY = process.env.LOCAL_API_KEY || "change-me";
```

改成：

```js
const API_KEY = "sk-my-custom-key-123";
```

---

## 7. 两个 Termux Session 的启动顺序

### Session A：LiteRT-LM

```bash
litert-lm serve \
  --host 127.0.0.1 \
  --port 9379 \
  --verbose
```

### Session B：Node.js

```bash
cd ~/local-api
export LOCAL_API_KEY="sk-my-custom-key-123"
node server.mjs
```

### 聊天客户端

```text
API 类型：OpenAI Compatible

Base URL:
http://127.0.0.1:5000/v1

API Key:
sk-my-custom-key-123

Model:
Qwen3-0.6B.litertlm
或
gemma-4-E2B-it-gpu.litertlm
```

不要把 `/v1/chat/completions` 写进 Base URL，客户端会自己拼。

---

## 8. 今天遇到过的坑

### 端口 9379 被占用

错误：

```text
OSError: [Errno 98] Address already in use
```

说明已经有一个 `litert-lm serve` 占用了 9379。

查看：

```bash
ss -ltnp | grep ':9379'
```

必要时：

```bash
pkill -f 'litert-lm serve'
```

---

### `/v1/models` 404

这是 Node.js 代理缺少该路由导致的。

现在模板已经包含：

```text
GET /v1/models
POST /v1/chat/completions
```

---

### Responses API 404

当前这套 LiteRT-LM OpenAI server 主要支持：

```text
/v1/models
/v1/chat/completions
```

所以现成聊天客户端应该使用 **Chat Completions**，不要打开 Responses API。

---

### Qwen3 thinking 太慢

Qwen3 默认会思考。单纯在 MJS 里过滤 `<think>` 没有意义，因为模型已经花时间生成了 thinking。

目前有效方法是给用户消息加入：

```text
/no_think
```

实测 Qwen3 从约 51 秒级下降到约 9 秒级（具体速度取决于上下文、GPU 和请求）。

---

### Qwen3 / Gemma CPU 与 GPU

强制 GPU 的 CLI：

```bash
litert-lm run MODEL.litertlm --backend gpu
```

Qwen3 GPU 日志中可看到：

```text
Loaded OpenCL library with dlopen.
Created OpenCL device...
Created LiteRT GpuEnvironment.
Replacing ... node(s) with delegate (LITERT_CL)
```

CPU 则常见：

```text
Created TensorFlow Lite XNNPACK delegate for CPU.
```

---

### 模型切换会重新初始化 Engine

LiteRT-LM 当前 server 生命周期是单一持久 Engine。

从一个模型切到另一个模型时可能出现：

```text
Re-initializing engine
```

因此不要为了测试 Qwen / Gemma 在一个会话里来回乱切；切模型可能带来明显初始化延迟。

---

### MCP 一开就爆 context

以前：

```text
5732 >= 4096
```

以及：

```text
5978 >= 4096
```

根本原因不是聊天记录太多，而是 MCP / system / user profile / tool schema / 搜索结果一起塞进上下文。

后来通过 LiteRT-LM CLI 配置：

```json
{
  "default": {},
  "models": {
    "Qwen3-0.6B.litertlm": {
      "max_num_tokens": 10240
    },
    "gemma-4-E2B-it-gpu.litertlm": {
      "max_num_tokens": 10240
    }
  }
}
```

解决了默认 4096 的限制。

---

### 搜索工具返回太多内容

10K context 不代表搜索结果可以无限塞。

如果一次搜索结果接近 9000 tokens，模型仍然很容易被上下文撑爆。

更合理的方法：

```text
搜索
 ↓
去重 / 截断 / 摘要
 ↓
只保留最相关的内容
 ↓
模型
```

特别是小模型，工具结果越短越容易处理。

---

## 9. 最终记忆版

忘了所有细节时，只记住这几行：

```bash
# 终端 1：模型服务
litert-lm serve --host 127.0.0.1 --port 9379 --verbose

# 终端 2：OpenAI 代理
cd ~/local-api
export LOCAL_API_KEY="sk-my-custom-key-123"
node server.mjs

# 前端
Base URL = http://127.0.0.1:5000/v1
API Key  = sk-my-custom-key-123
```

然后：

```text
5000 = 我的 Node.js API 代理
9379 = LiteRT-LM 模型服务

Qwen3 = 推荐加 /no_think + GPU
MCP   = 注意上下文长度
```

这套结构的核心思想就是：**前端永远只认识 5000，5000 再去找 9379。**
