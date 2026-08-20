---
title: "Termux 本地 LiteRT-LM API 实操"
description: "用 termux 实现本地模型的 API，内网范围都可访问。"
date: "2026-08-20"
categories: AI
tags:
  - AI
---


### 1. 安装基础环境

```bash
pkg update
pkg upgrade
pkg install python nodejs
```

检查：

```
python --version
node --version
npm --version
```

输出版本号就代表安装成功了

---

### 2. 确认 LiteRT-LM

安装 LiteRT-LM

```
pip install -U litert-lm
```

查看版本：
```
litert-lm --version
```

---

### 3. 创建 MJS 项目目录

```
mkdir -p ~/local-api
cd ~/local-api
```

创建 MJS：
```
nano server.mjs
```
粘贴这个：

```
import http from "node:http";

const HOST = "127.0.0.1";
const PORT = 5000;

const BACKEND = "http://127.0.0.1:9379";

const API_KEY = "sk-my-custom-key-123";

const QWEN_MODEL = "Qwen3-0.6B.litertlm";
const GEMMA_MODEL = "gemma-4-E2B-it-gpu.litertlm";

const server = http.createServer(async (req, res) => {
  try {
    // =========================
    // CORS
    // =========================

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

    // =========================
    // API Key
    // =========================

    const authorization = req.headers.authorization;

    if (authorization !== `Bearer ${API_KEY}`) {
      res.writeHead(401, {
        "Content-Type": "application/json",
      });

      res.end(
        JSON.stringify({
          error: {
            message: "Invalid API key",
            type: "authentication_error",
          },
        })
      );

      return;
    }

    // =========================
    // GET /v1/models
    // =========================

    if (
      req.method === "GET" &&
      req.url === "/v1/models"
    ) {
      const response = await fetch(
        `${BACKEND}/v1/models`
      );

      const body = await response.text();

      res.writeHead(response.status, {
        "Content-Type":
          response.headers.get("content-type") ||
          "application/json",
      });

      res.end(body);
      return;
    }

    // =========================
    // POST /v1/chat/completions
    // =========================

    if (
      req.method === "POST" &&
      req.url === "/v1/chat/completions"
    ) {
      // =========================
      // 读取请求 Body
      // =========================

      const chunks = [];

      for await (const chunk of req) {
        chunks.push(chunk);
      }

      const body = Buffer
        .concat(chunks)
        .toString("utf8");

      // =========================
      // 解析 JSON
      // =========================

      let requestData;

      try {
        requestData = JSON.parse(body);
      } catch {
        res.writeHead(400, {
          "Content-Type": "application/json",
        });

        res.end(
          JSON.stringify({
            error: {
              message: "Invalid JSON",
              type: "invalid_request_error",
            },
          })
        );

        return;
      }

      // =========================
      // 原始模型名
      // =========================

      const originalModel = requestData.model;

      // =========================
      // Qwen3
      // =========================
      //
      // 1. 强制 GPU
      // 2. 自动 /no_think
      //
      // 前端仍然只需要填写：
      //
      // Qwen3-0.6B.litertlm
      //
      // Node.js 内部转换成：
      //
      // Qwen3-0.6B.litertlm,gpu
      //
      // =========================

      if (originalModel === QWEN_MODEL) {
        requestData.model = `${QWEN_MODEL},gpu`;

        if (Array.isArray(requestData.messages)) {
          const firstUserIndex =
            requestData.messages.findIndex(
              (message) =>
                message &&
                message.role === "user"
            );

          if (firstUserIndex !== -1) {
            const message =
              requestData.messages[firstUserIndex];

            if (
              typeof message.content === "string"
            ) {
              const content = message.content;

              if (
                !content
                  .trimStart()
                  .startsWith("/no_think")
              ) {
                requestData.messages[firstUserIndex] = {
                  ...message,
                  content: `/no_think\n${content}`,
                };
              }
            }
          }
        }
      }

      // =========================
      // Gemma 4 E2B
      // =========================
      //
      // 尝试把 KV cache / 最大 token 数
      // 提高到 10240。
      //
      // 注意：
      // 这取决于 LiteRT-LM 和模型本身是否允许。
      // 如果模型仍然报告 4096 限制，
      // 就说明 artifact 本身仍有限制。
      //
      // =========================

      if (originalModel === GEMMA_MODEL) {
        requestData.max_num_tokens = 10240;

        // Gemma 的模型已经是 GPU 专用 artifact，
        // 这里不修改 model 名称。
      }

      // =========================
      // Streaming
      // =========================

      const isStream =
        requestData.stream === true;

      // =========================
      // 调试日志
      // =========================
      //
      // 只打印必要信息，避免 Termux 刷屏。
      //
      console.log(
        `[Proxy] ${originalModel} -> ${requestData.model}` +
        (requestData.max_num_tokens
          ? ` | max_tokens=${requestData.max_num_tokens}`
          : "") +
        (isStream
          ? " | stream=true"
          : " | stream=false")
      );

      // =========================
      // 请求 LiteRT-LM
      // =========================

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

      // =========================
      // 后端错误
      // =========================

      if (!response.ok) {
        const errorBody =
          await response.text();

        res.writeHead(response.status, {
          "Content-Type":
            response.headers.get("content-type") ||
            "application/json",
        });

        res.end(errorBody);
        return;
      }

      // =========================
      // Streaming
      // =========================

      if (isStream) {
        res.writeHead(200, {
          "Content-Type":
            "text/event-stream",

          "Cache-Control":
            "no-cache, no-transform",

          "Connection":
            "keep-alive",

          "X-Accel-Buffering":
            "no",
        });

        if (response.body) {
          for await (const chunk of response.body) {
            res.write(chunk);
          }
        }

        res.end();
        return;
      }

      // =========================
      // 非 Streaming
      // =========================

      const responseBody =
        await response.text();

      res.writeHead(200, {
        "Content-Type":
          response.headers.get("content-type") ||
          "application/json",
      });

      res.end(responseBody);
      return;
    }

    // =========================
    // 404
    // =========================

    res.writeHead(404, {
      "Content-Type": "application/json",
    });

    res.end(
      JSON.stringify({
        error: {
          message: "Not Found",
          type: "invalid_request_error",
        },
      })
    );

  } catch (error) {
    console.error(
      "[Proxy Error]",
      error
    );

    if (!res.headersSent) {
      res.writeHead(502, {
        "Content-Type":
          "application/json",
      });
    }

    res.end(
      JSON.stringify({
        error: {
          message:
            "LiteRT-LM backend unavailable",

          type: "server_error",

          details:
            error.message,
        },
      })
    );
  }
});

// =========================
// 启动
// =========================

server.listen(
  PORT,
  HOST,
  () => {
    console.log("");
    console.log(
      "Local OpenAI API proxy started."
    );

    console.log(
      `API:     http://${HOST}:${PORT}/v1`
    );

    console.log(
      `Backend: ${BACKEND}/v1`
    );

    console.log("");
  }
);
```

Ctrl + O

回车

Ctrl + X

就保存了

---


### 4. 确认 LiteRT-LM 模型

先导入你要的模型 这里以 gemma 4 E2B 为例

```
litert-lm import ~/.litert-lm/cache/huggingface/litert-community/gemma-4-E2B-it-litert-lm/gemma-4-E2B-it-gpu.litertlm
```

查看模型：

```
litert-lm list
```

输出
> Listing models in: /data/data/com.termux/files/home/.litert-lm/models
ID                            SIZE            MODIFIED
gemma-4-E2B-it-gpu.litertlm   1.9 GB          2026-08-20 21:08:05


---

### 5. 启动 LiteRT-LM API

打开一个 Termux Session：

```
litert-lm serve \
  --host 127.0.0.1 \
  --port 9379 \
  --verbose
```
看到：

> Starting OpenAI-compatible API server on 127.0.0.1:9379...

就说明 API 服务已经启动。

不要关闭这个 Session。


---

### 6. 测试 LiteRT-LM API

重新打开一个 Termux Session。

测试：

curl http://127.0.0.1:9379/v1/models

如果能返回模型信息，说明 LiteRT-LM API 正常。


---

### 7. 启动自己的 MJS API

进入项目：
```
cd ~/local-api
```
启动：
```
node server.mjs
```
如果你的 MJS 使用 5000 端口，应该看到类似：

> Server listening on http://127.0.0.1:5000

现在保持两个 Session 都运行：

Session 1
```
litert-lm serve --host 127.0.0.1 --port 9379 --verbose
```
Session 2
```
cd ~/local-api
node server.mjs
```

---

### 8. 测试 MJS API

再开一个 Session：
```
curl http://127.0.0.1:5000/v1/models
```
如果能正常返回，就说明：

MJS API
↓
127.0.0.1:5000
↓
LiteRT-LM
↓
127.0.0.1:9379
↓
本地模型


---

### 9. 聊天客户端配置

OpenAI API Base URL：

http://127.0.0.1:5000/v1

API Key：

使用 MJS 要求的 Key

模型名称：

gemma-4-E2B-it-gpu.litertlm


---

### 10. 以后重新启动

以后不用重新配置。

直接：

Session 1
```
litert-lm serve --host 127.0.0.1 --port 9379 --verbose
```

Session 2

```
cd ~/local-api
node server.mjs
```
然后客户端连接：

照着这个输入就行
API Key：sk-my-custom-key-123
API Base URL：http://127.0.0.1:5000/v1
API Path：/chat/completions

再添加模型 输入你的模型 ID
例：
```
gemma-4-E2B-it-gpu.litertlm
```

就此就能愉快地在你的聊天客户端聊天了。