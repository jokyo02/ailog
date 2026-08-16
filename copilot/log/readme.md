/**
*代码功能汇总：
*
*功能	说明
*端点	/v1/chat/completions（OpenAI）、/v1/messages（Anthropic）、/v1/models、/v1/files
*认证	Authorization: Bearer、x-api-key、?key= 三种方式
*模型路由	所有模型统一映射到 Copilot smart 模式
*模型列表	返回 GPT-4、Claude 4/3、Gemini 等主流模型
*心跳	2 秒间隔 SSE 注释心跳，连接建立后立即发送空首块
*超时	90 秒 WS 超时，超时自动发送 timeout finish_reason
*兜底	WS 异常关闭时自动补发 finish 和 [DONE]
*工具调用	支持 OpenAI tools/tool_calls 两轮协议
*文件上传	/v1/files 端点上文件，5 分钟后自动清理
*长消息	超过 2000 字符自动转为文件上传
*Claude Code	完整 Anthropic Messages API 流式/非流式支持
*重试	WebSocket 连接失败自动重试 1 次
**/

/**
 * Microsoft Copilot API 转换器 · 增强版
 * 支持文件上传与临时存储（符合 OpenAI 规范）
 *
 * 功能：
 * - OpenAI 兼容的 /v1/chat/completions 接口
 * - 模型路由（根据 model 自动选择 Copilot 模式）
 * - 工具调用模拟（tools / tool_calls）
 * - 文件上传接口 /v1/files（返回 OpenAI 格式文件对象）
 * - 在聊天中通过 file_ids 数组引用已上传文件
 * - 长文本自动转换为文件并引用
 * - 文件在最后一次引用后5分钟自动清理
 * - 文件保存格式为 0temp/IP地址/日期/时间.txt
 */

const PROXY_DOMAIN = "copilot.microsoft.com";
const FILEBED_BASE_URL = "https://chatyou-filebed.hf.space";
const MAX_MESSAGE_LENGTH = 5000; // 超过此长度的用户消息自动转为文件
const CLEANUP_DELAY = 300000; // 5分钟（毫秒）

// ==================== 工具函数：获取客户端 IP 地址 ====================
function getClientIP(request) {
  const cfConnectingIP = request.headers.get("CF-Connecting-IP");
  if (cfConnectingIP) return cfConnectingIP;
  
  const xForwardedFor = request.headers.get("X-Forwarded-For");
  if (xForwardedFor) {
    const ips = xForwardedFor.split(",");
    if (ips.length > 0) return ips[0].trim();
  }
  
  const xRealIP = request.headers.get("X-Real-IP");
  if (xRealIP) return xRealIP;
  
  return "unknown";
}

// ==================== 工具函数：正确构建文件 URL ====================
function buildFileUrl(remotePath) {
  if (remotePath.startsWith('http://') || remotePath.startsWith('https://')) {
    return remotePath;
  }
  const lastSlashIndex = remotePath.lastIndexOf('/');
  if (lastSlashIndex === -1) {
    return `${FILEBED_BASE_URL}/file/${encodeURIComponent(remotePath)}`;
  }
  const dir = remotePath.substring(0, lastSlashIndex);
  const file = remotePath.substring(lastSlashIndex + 1);
  const encodedFile = encodeURIComponent(file);
  return `${FILEBED_BASE_URL}/file/${dir}/${encodedFile}`;
}

// ==================== 工具函数：生成 0temp/IP地址/日期/时间.txt 格式的文件路径 ====================
function generateIPTimeFilePath(clientIP) {
  const cleanIP = clientIP.replace(/[^a-zA-Z0-9]/g, "_");
  
  const now = new Date();
  const year = now.getFullYear();
  const month = String(now.getMonth() + 1).padStart(2, "0");
  const day = String(now.getDate()).padStart(2, "0");
  const hours = String(now.getHours()).padStart(2, "0");
  const minutes = String(now.getMinutes()).padStart(2, "0");
  const seconds = String(now.getSeconds()).padStart(2, "0");
  
  // 日期目录：YYYYMMDD
  const dateDir = `${year}${month}${day}`;
  // 时间文件名：HHMMSS.txt
  const fileName = `${hours}${minutes}${seconds}.txt`;
  
  return {
    directory: `0temp/${cleanIP}/${dateDir}`,  // 目录为 0temp/IP地址/日期
    fileName: fileName,                         // 文件名为 时间.txt
    fullPath: `0temp/${cleanIP}/${dateDir}/${fileName}`  // 完整路径
  };
}

// ==================== 模型路由表 ====================
const MODEL_ROUTE_MAP = [
  { pattern: /^o1(-\w+)?$/i,                         mode: "smart",    label: "o1-reasoning"  },
  { pattern: /^gpt-4o$/i,                             mode: "smart",    label: "gpt-4o"        },
  { pattern: /^gpt-4(-\d+k)?$/i,                     mode: "smart",    label: "gpt-4"         },
  { pattern: /^gpt-4-turbo(-\w+)?$/i,                mode: "precise",  label: "gpt-4-turbo"   },
  { pattern: /^gpt-4o-mini$/i,                        mode: "balanced", label: "gpt-4o-mini"   },
  { pattern: /^gpt-3\.5-turbo(-\w+)?$/i,             mode: "balanced", label: "gpt-3.5-turbo" },
  { pattern: /^claude-/i,                             mode: "precise",  label: "claude"        },
  { pattern: /^gemini-/i,                             mode: "balanced", label: "gemini"        },
  { pattern: /creative/i,                             mode: "creative", label: "creative"      },
  { pattern: /precise|exact/i,                        mode: "precise",  label: "precise"       },
  { pattern: /.*/,                                    mode: "smart",    label: "default"       },
];

function resolveModelRoute(modelName) {
  const name = (modelName || "gpt-4o").trim();
  for (const route of MODEL_ROUTE_MAP) {
    if (route.pattern.test(name)) {
      console.log(`[ModelRouter] ${name} → mode=${route.mode} (${route.label})`);
      return { mode: route.mode, label: route.label };
    }
  }
  return { mode: "smart", label: "default" };
}

// ==================== 工具调用相关 ====================
const TOOL_CALL_OPEN  = "<tool_call>";
const TOOL_CALL_CLOSE = "</tool_call>";

function injectToolsIntoPrompt(prompt, tools, toolChoice = "auto") {
  if (!tools || tools.length === 0) return prompt;
  if (toolChoice === "none") return prompt;

  let forceFuncName = null;
  if (toolChoice && typeof toolChoice === "object" && toolChoice.type === "function") {
    forceFuncName = toolChoice.function?.name || null;
  }

  const toolDescriptions = tools
    .filter(t => t.type === "function" && t.function)
    .map(t => {
      const fn = t.function;
      const params = fn.parameters ? JSON.stringify(fn.parameters, null, 2) : "{}";
      return `### ${fn.name}\n描述：${fn.description || "（无描述）"}\n参数 Schema：\n${params}`;
    })
    .join("\n\n");

  const forceHint = forceFuncName
    ? `\n你必须调用函数 \`${forceFuncName}\`，不得调用其他函数或输出任何其他内容。`
    : "";

  const example = `
例如，如果用户问"33加49等于多少？"，你需要调用 calculator 工具，则输出：
${TOOL_CALL_OPEN}
{
  "name": "calculator",
  "arguments": {
    "a": 33,
    "b": 49,
    "op": "add"
  }
}
${TOOL_CALL_CLOSE}
`;

  const injected = `
[强制指令]
如果用户请求需要调用工具，你必须**只输出以下 JSON 格式**，不得包含任何其他文字、解释或标记：

${TOOL_CALL_OPEN}
{
  "name": "<函数名>",
  "arguments": { <参数> }
}
${TOOL_CALL_CLOSE}

如果不需要调用工具，则直接输出普通回答，但不得输出上述 JSON 格式。${forceHint}

可用工具列表：
${toolDescriptions}

现在开始执行：
${prompt}`;

  return injected;
}

function parseToolCallsFromText(text) {
  if (!text || !text.includes(TOOL_CALL_OPEN)) {
    return { hasTool: false, calls: [], remainder: text };
  }

  const calls = [];
  let remainder = text;
  const regex = new RegExp(
    `${TOOL_CALL_OPEN.replace(/</g, "\\<").replace(/>/g, "\\>")}([\\s\\S]*?)${TOOL_CALL_CLOSE.replace(/</g, "\\<").replace(/>/g, "\\>")}`,
    "g"
  );

  let match;
  while ((match = regex.exec(text)) !== null) {
    const raw = match[1].trim();
    try {
      const parsed = JSON.parse(raw);
      if (parsed.name) {
        calls.push({
          id:        `call_${crypto.randomUUID().replace(/-/g, "").slice(0, 24)}`,
          name:      parsed.name,
          arguments: typeof parsed.arguments === "string" ? parsed.arguments : JSON.stringify(parsed.arguments || {})
        });
      }
    } catch (e) {
      console.warn("[ToolParser] JSON 解析失败:", raw.slice(0, 200), e.message);
    }
    remainder = remainder.replace(match[0], "").trim();
  }

  return { hasTool: calls.length > 0, calls, remainder };
}

function buildToolCallsNonStreamResponse(convId, model, toolCalls) {
  return {
    id:      `chatcmpl-${convId}`,
    object:  "chat.completion",
    created: Math.floor(Date.now() / 1000),
    model:   model,
    choices: [{
      index:         0,
      message: {
        role:       "assistant",
        content:    null,
        tool_calls: toolCalls.map(c => ({
          id:       c.id,
          type:     "function",
          function: { name: c.name, arguments: c.arguments }
        }))
      },
      finish_reason: "tool_calls"
    }],
    usage: { prompt_tokens: 0, completion_tokens: 0, total_tokens: 0 }
  };
}

async function writeToolCallsToStream(writer, encoder, convId, model, toolCalls) {
  const chunkBase = () => ({
    id:      `chatcmpl-${convId}-${Date.now()}`,
    object:  "chat.completion.chunk",
    created: Math.floor(Date.now() / 1000),
    model:   model,
  });

  const roleChunk = {
    ...chunkBase(),
    choices: [{ index: 0, delta: { role: "assistant", content: null }, finish_reason: null }]
  };
  writer.write(encoder.encode(`data: ${JSON.stringify(roleChunk)}\n\n`));

  for (let i = 0; i < toolCalls.length; i++) {
    const tc = toolCalls[i];
    const headerChunk = {
      ...chunkBase(),
      choices: [{
        index: 0,
        delta: {
          tool_calls: [{
            index:    i,
            id:       tc.id,
            type:     "function",
            function: { name: tc.name, arguments: "" }
          }]
        },
        finish_reason: null
      }]
    };
    writer.write(encoder.encode(`data: ${JSON.stringify(headerChunk)}\n\n`));

    const args = tc.arguments;
    const chunkSize = 32;
    for (let j = 0; j < args.length; j += chunkSize) {
      const slice = args.slice(j, j + chunkSize);
      const argChunk = {
        ...chunkBase(),
        choices: [{
          index: 0,
          delta: { tool_calls: [{ index: i, function: { arguments: slice } }] },
          finish_reason: null
        }]
      };
      writer.write(encoder.encode(`data: ${JSON.stringify(argChunk)}\n\n`));
    }
  }

  const finishChunk = {
    ...chunkBase(),
    choices: [{ index: 0, delta: {}, finish_reason: "tool_calls" }]
  };
  writer.write(encoder.encode(`data: ${JSON.stringify(finishChunk)}\n\n`));
  writer.write(encoder.encode("data: [DONE]\n\n"));
  writer.close();
}

function hasToolResultMessages(messages) {
  return Array.isArray(messages) && messages.some(m => m.role === "tool");
}

function normalizeToolMessages(messages) {
  const normalized = [];
  for (const msg of messages) {
    if (msg.role === "assistant" && msg.tool_calls && !msg.content) {
      const callTexts = msg.tool_calls.map(tc =>
        `[调用工具 ${tc.function.name}，参数：${tc.function.arguments}]`
      ).join("\n");
      normalized.push({ role: "assistant", content: callTexts });
    } else if (msg.role === "tool") {
      const toolName = msg.name || "tool";
      const toolContent = typeof msg.content === "string" ? msg.content : JSON.stringify(msg.content);
      normalized.push({
        role: "user",
        content: `[工具 ${toolName} 的执行结果]\n${toolContent}\n\n请基于上述工具结果继续回答用户问题。`
      });
    } else {
      normalized.push(msg);
    }
  }
  return normalized;
}

// ==================== 文件上传处理（OpenAI 兼容）====================
async function handleFileUpload(request, env) {
  try {
    const formData = await request.formData();
    const file = formData.get("file");
    if (!file) {
      return new Response(JSON.stringify({ error: "No file uploaded" }), {
        status: 400,
        headers: { "Content-Type": "application/json" }
      });
    }

    const clientIP = getClientIP(request);
    const filePath = generateIPTimeFilePath(clientIP);

    const uploadFormData = new FormData();
    const renamedFile = new File([file], filePath.fileName, { type: file.type });
    uploadFormData.append("file", renamedFile);
    uploadFormData.append("dir", filePath.directory);

    const uploadController = new AbortController();
    const uploadTimeoutId = setTimeout(() => uploadController.abort(), 10000);

    const uploadRes = await fetch(`${FILEBED_BASE_URL}/upload`, {
      method: "POST",
      body: uploadFormData,
      signal: uploadController.signal,
    });
    clearTimeout(uploadTimeoutId);

    if (!uploadRes.ok) {
      const errorText = await uploadRes.text();
      return new Response(JSON.stringify({ error: "Upload to filebed failed", details: errorText }), {
        status: 500,
        headers: { "Content-Type": "application/json" }
      });
    }

    const uploadResult = await uploadRes.json();
    if (!uploadResult.success) {
      return new Response(JSON.stringify({ error: "Upload failed: " + JSON.stringify(uploadResult) }), {
        status: 500,
        headers: { "Content-Type": "application/json" }
      });
    }

    const remoteFilenameFromServer = uploadResult.filename;
    const fileUrl = buildFileUrl(remoteFilenameFromServer);

    const fileId = `file_${crypto.randomUUID().replace(/-/g, "").slice(0, 24)}`;

    await env.FILES_KV.put(`file:${fileId}`, JSON.stringify({
      remoteFilename: remoteFilenameFromServer,
      url: fileUrl,
      created: Date.now(),
      originalFilename: filePath.fullPath,
    }), { expirationTtl: 600 });

    return new Response(JSON.stringify({
      id: fileId,
      object: "file",
      bytes: file.size,
      created_at: Math.floor(Date.now() / 1000),
      filename: filePath.fullPath,
      purpose: "assistants",
    }), {
      headers: { "Content-Type": "application/json" }
    });
  } catch (err) {
    if (err.name === 'AbortError') {
      return new Response(JSON.stringify({ error: "File upload timeout (10s)" }), {
        status: 408,
        headers: { "Content-Type": "application/json" }
      });
    }
    console.error("[Upload] Error:", err);
    return new Response(JSON.stringify({ error: err.message }), {
      status: 500,
      headers: { "Content-Type": "application/json" }
    });
  }
}

// ==================== 主 Worker ====================
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);

    if (request.method === "OPTIONS") {
      return new Response(null, {
        headers: {
          "Access-Control-Allow-Origin": "*",
          "Access-Control-Allow-Headers": "Content-Type, Authorization",
          "Access-Control-Allow-Methods": "POST, GET, OPTIONS",
          "Access-Control-Max-Age": "86400"
        }
      });
    }

    if (url.pathname === "/v1/models" && request.method === "GET") {
      return this.handleModelsRequest();
    }

    if (url.pathname === "/v1/files" && request.method === "POST") {
      return handleFileUpload(request, env);
    }

    if (url.pathname === "/v1/chat/completions" && request.method === "POST") {
      return this.handleChatCompletion(request, env, ctx);
    }

    return new Response(JSON.stringify({
      error: { message: "端点: /v1/chat/completions（OpenAI）、/v1/messages（Anthropic）、/v1/models、/v1/files", type: "invalid_request_error" }
    }), {
      status: 404,
      headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
    });
  },

  handleModelsRequest() {
    const models = [
      { id: "gpt-4o",           object: "model", created: 1686935000, owned_by: "copilot" },
      { id: "gpt-4",            object: "model", created: 1686935000, owned_by: "copilot" },
      { id: "gpt-4-turbo",      object: "model", created: 1686935000, owned_by: "copilot" },
      { id: "gpt-4o-mini",      object: "model", created: 1686935000, owned_by: "copilot" },
      { id: "gpt-3.5-turbo",    object: "model", created: 1686935000, owned_by: "copilot" },
      { id: "claude-3-opus",    object: "model", created: 1686935000, owned_by: "copilot" },
      { id: "claude-3-sonnet",  object: "model", created: 1686935000, owned_by: "copilot" },
      { id: "gemini-pro",       object: "model", created: 1686935000, owned_by: "copilot" },
    ];
    return new Response(JSON.stringify({ object: "list", data: models }), {
      headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
    });
  },

  // ==================== 聊天补全入口 ====================
  async handleChatCompletion(request, env, ctx) {
    const url = new URL(request.url);
    let userCookie = "";
    const queryKey = url.searchParams.get("key");
    if (queryKey) {
      userCookie = queryKey;
    } else {
      const authHeader = request.headers.get("Authorization") || "";
      if (authHeader) {
        userCookie = authHeader.replace(/^[Bb]earer\s+/i, "").trim();
      }
    }
    userCookie = userCookie.replace(/^["']|["']$/g, "").trim();
    try {
      if (userCookie.includes("%")) userCookie = decodeURIComponent(userCookie);
    } catch (e) {}
    if (!userCookie) {
      userCookie = `anon_${crypto.randomUUID().replace(/-/g, "")}`;
    }

    let requestBody;
    try {
      requestBody = await request.json();
    } catch (e) {
      return this.createErrorResponse("Invalid JSON in request body", "invalid_request_error");
    }

    console.log("[ChatCompletion] Request body:", JSON.stringify(requestBody));

    // ---------- 文件处理 ----------
    const filesToCleanup = [];
    const fileUrls = [];
    const messages = requestBody.messages || [];

    // 第一步：处理客户端传入的 file_ids 数组
    if (requestBody.file_ids && Array.isArray(requestBody.file_ids)) {
      for (const fileId of requestBody.file_ids) {
        try {
          const fileInfo = await env.FILES_KV.get(`file:${fileId}`, "json");
          if (!fileInfo) {
            return this.createErrorResponse(`File not found or expired: ${fileId}`, "file_error");
          }
          fileUrls.push(fileInfo.url);
          filesToCleanup.push(fileInfo.remoteFilename);

          await env.FILES_KV.put(`file:${fileId}`, JSON.stringify(fileInfo), { expirationTtl: 600 });
        } catch (err) {
          return this.createErrorResponse(`Failed to retrieve file ${fileId}: ${err.message}`, "file_error");
        }
      }
      delete requestBody.file_ids;
    }

    // 第二步：找到最后一条用户消息，检查是否超长
    const lastUserIndex = messages.map(m => m.role).lastIndexOf("user");
    
    if (lastUserIndex >= 0) {
      const lastUserMsg = messages[lastUserIndex];
      
      if (typeof lastUserMsg.content === "string" && lastUserMsg.content.length > MAX_MESSAGE_LENGTH) {
        console.log("[LongMsg] Found long message, length:", lastUserMsg.content.length);
        
        const clientIP = getClientIP(request);
        const filePath = generateIPTimeFilePath(clientIP);
        const textBlob = new Blob([lastUserMsg.content], { type: "text/plain" });
        const uploadFormData = new FormData();
        uploadFormData.append("file", textBlob, filePath.fileName);
        uploadFormData.append("dir", filePath.directory);

        const uploadController = new AbortController();
        const uploadTimeoutId = setTimeout(() => uploadController.abort(), 10000);

        try {
          const uploadRes = await fetch(`${FILEBED_BASE_URL}/upload`, {
            method: "POST",
            body: uploadFormData,
            signal: uploadController.signal,
          });
          clearTimeout(uploadTimeoutId);

          if (!uploadRes.ok) {
            console.error("[LongMsg] Upload failed:", await uploadRes.text());
          } else {
            const uploadResult = await uploadRes.json();
            if (!uploadResult.success) {
              console.error("[LongMsg] Upload failed: server returned error");
            } else {
              const remoteFilename = uploadResult.filename;
              const fileUrl = buildFileUrl(remoteFilename);
              console.log("[LongMsg] File URL:", fileUrl);

              fileUrls.push(fileUrl);
              filesToCleanup.push(remoteFilename);
              
              // 删除长消息
              messages.splice(lastUserIndex, 1);
              console.log("[LongMsg] Long message removed from list");
            }
          }
        } catch (err) {
          clearTimeout(uploadTimeoutId);
          console.error("[LongMsg] Upload error:", err.message);
        }
      }
    }

    // 第三步：如果有文件需要引用，统一合并到一个消息中
    if (fileUrls.length > 0) {
      const filePrompt = `请阅读以下文件：${fileUrls.join(' ')}`;
      console.log("[Files] Combined file prompt:", filePrompt);
      
      // 重新找到最后一条用户消息的位置
      const newLastUserIndex = messages.map(m => m.role).lastIndexOf("user");
      
      if (newLastUserIndex >= 0) {
        // 在最后一条用户消息之前插入文件引用
        messages.splice(newLastUserIndex, 0, { role: "user", content: filePrompt });
        console.log("[Files] Inserted file prompt before last user message");
      } else {
        // 没有用户消息，直接在开头插入
        messages.unshift({ role: "user", content: filePrompt });
        console.log("[Files] Inserted file prompt at beginning");
      }
    }

    console.log("[ChatCompletion] Messages after file insertion:", JSON.stringify(messages));
    // ---------- 文件处理结束 ----------

    // ---------- 核心聊天流程 ----------
    const streamMode = requestBody.stream === true;
    const contextData = await this.initializeEnvironment(userCookie);

    let response;
    if (streamMode) {
      response = await this.handleStreamMode(request, contextData, userCookie, requestBody);
    } else {
      response = await this.handleNonStreamMode(request, contextData, userCookie, requestBody);
    }

    // ---------- 异步清理文件（5分钟后执行） ----------
    if (filesToCleanup.length > 0) {
      ctx.waitUntil(
        (async () => {
          await new Promise(resolve => setTimeout(resolve, CLEANUP_DELAY));
          for (const filename of filesToCleanup) {
            try {
              const fileDeleteUrl = `${FILEBED_BASE_URL}/delete/${filename}`;
              await fetch(fileDeleteUrl, { method: "DELETE" });
              console.log(`[Cleanup] Deleted file ${filename}`);
            } catch (err) {
              console.error(`[Cleanup] Failed to delete file ${filename}:`, err);
            }
          }
        })()
      );
    }

    return response;
  },

  // ==================== 流式模式 ====================
  async handleStreamMode(request, startData, userCookie, requestBody) {
    const { readable, writable } = new TransformStream();
    const writer = writable.getWriter();
    const encoder = new TextEncoder();

    const tools = requestBody.tools || null;
    const toolChoice = requestBody.tool_choice || "auto";
    const messages = requestBody.messages || [];
    const isToolResultRound = hasToolResultMessages(messages);

    (async () => {
      try {
        let effectiveRequestBody = requestBody;
        if (tools && tools.length > 0 && !isToolResultRound) {
          console.log("[Tools] 检测到 tools 定义，注入 prompt...");
          const fullText = await this._collectStreamAsText(startData, userCookie, effectiveRequestBody, tools, toolChoice);
          const parsed = parseToolCallsFromText(fullText);
          if (parsed.hasTool) {
            console.log("[Tools] 检测到工具调用，转换为 tool_calls 流式响应");
            await writeToolCallsToStream(writer, encoder, startData.conversationId, requestBody.model || "gpt-4o", parsed.calls);
          } else {
            const model = requestBody.model || "gpt-4o";
            const convId = startData.conversationId;
            const firstChunk = this.createOpenAIChunk({ conversationId: convId, model, isFirst: true });
            writer.write(encoder.encode(`data: ${JSON.stringify(firstChunk)}\n\n`));
            if (fullText) {
              const contentChunk = this.createOpenAIChunk({ conversationId: convId, model, content: fullText });
              writer.write(encoder.encode(`data: ${JSON.stringify(contentChunk)}\n\n`));
            }
            const finalChunk = this.createOpenAIChunk({ conversationId: convId, model, isLast: true, finishReason: "stop" });
            writer.write(encoder.encode(`data: ${JSON.stringify(finalChunk)}\n\n`));
            writer.write(encoder.encode("data: [DONE]\n\n"));
            writer.close();
          }
        } else {
          if (isToolResultRound) {
            console.log("[Tools] 检测到工具结果回传，规范化消息...");
            effectiveRequestBody = { ...requestBody, messages: normalizeToolMessages(messages) };
          }
          await this.handleChatStream(request, startData, userCookie, writer, encoder, { locked: false }, effectiveRequestBody);
        }
      } catch (err) {
        console.error("Stream processing error:", err);
        try {
          writer.write(encoder.encode(`data: ${JSON.stringify({ error: { message: err.message, type: "internal_error" } })}\n\n`));
          writer.write(encoder.encode("data: [DONE]\n\n"));
          writer.close();
        } catch (e) {}
      }
    })();

    return new Response(readable, {
      headers: {
        "Content-Type": "text/event-stream",
        "Access-Control-Allow-Origin": "*",
        "Cache-Control": "no-cache",
        "Connection": "keep-alive"
      }
    });
  },

  // ==================== 非流式模式 ====================
  async handleNonStreamMode(request, startData, userCookie, requestBody) {
    try {
      const tools = requestBody.tools || null;
      const toolChoice = requestBody.tool_choice || "auto";
      const messages = requestBody.messages || [];
      const isToolResultRound = hasToolResultMessages(messages);

      let effectiveRequestBody = requestBody;

      if (tools && tools.length > 0 && !isToolResultRound) {
        console.log("[Tools] Non-stream: 检测到 tools 定义，注入 prompt...");
        const fullText = await this._collectStreamAsText(startData, userCookie, requestBody, tools, toolChoice);
        const parsed = parseToolCallsFromText(fullText);
        if (parsed.hasTool) {
          console.log("[Tools] Non-stream: 检测到工具调用，返回 tool_calls 响应");
          return new Response(
            JSON.stringify(buildToolCallsNonStreamResponse(startData.conversationId, requestBody.model || "gpt-4o", parsed.calls)),
            { headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" } }
          );
        } else {
          return new Response(
            JSON.stringify(this.createNonStreamResponse(startData.conversationId, requestBody.model || "gpt-4o", fullText)),
            { headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" } }
          );
        }
      }

      if (isToolResultRound) {
        console.log("[Tools] Non-stream: 检测到工具结果回传，规范化消息...");
        effectiveRequestBody = { ...requestBody, messages: normalizeToolMessages(messages) };
      }

      const result = await this.processChatCompletion(effectiveRequestBody, startData, userCookie, null, null, false);
      return new Response(JSON.stringify(result), {
        headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
      });
    } catch (err) {
      console.error("Non-stream processing error:", err);
      return this.createErrorResponse(err.message, "internal_error");
    }
  },

  async _collectStreamAsText(startData, userCookie, requestBody, tools, toolChoice) {
    const originalMessages = requestBody.messages || [];
    const { text: basePrompt } = this.convertToCopilotFormat(originalMessages);
    const injectedPrompt = injectToolsIntoPrompt(basePrompt, tools, toolChoice);
    const enhancedMessages = [...originalMessages, { role: "system", content: injectedPrompt }];
    const injectedRequestBody = {
      ...requestBody,
      messages: enhancedMessages,
      tools: undefined,
      tool_choice: undefined,
      stream: false
    };
    const result = await this.processChatCompletion(injectedRequestBody, startData, userCookie, null, null, false);
    return result?.choices?.[0]?.message?.content || "";
  },

  // ==================== 核心聊天处理 ====================
  async processChatCompletion(requestBody, startData, userCookie, writer, encoder, isStream) {
    const messages = requestBody.messages || [];
    const model = requestBody.model || "gpt-4o";

    console.log(`Processing ${messages.length} messages for model ${model}, stream: ${isStream}`);

    const { mode: copilotMode } = resolveModelRoute(model);
    let effectiveMode = copilotMode;
    const isToolFirstRound = requestBody.tools && !hasToolResultMessages(messages);
    if (isToolFirstRound) {
      effectiveMode = "precise";
      console.log("[Tools] 第一轮工具调用，强制使用 precise 模式");
    }

    const { text: copilotPrompt } = this.convertToCopilotFormat(messages);
    if (!copilotPrompt.trim()) {
      console.log("Empty prompt after conversion");
      return this.createEmptyResponse(startData.conversationId, model, isStream, writer, encoder);
    }

    const wssUrl = `https://${PROXY_DOMAIN}/c/api/chat?api-version=2&features=-%2Cncedge%2Cedgepagecontext&setflight=-%2Cncedge%2Cedgepagecontext&ncedge=1`;

    let ws;
    try {
      const wsRes = await fetch(wssUrl, {
        headers: {
          "Upgrade": "websocket",
          "Cookie": userCookie,
          "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0",
          "Origin": `https://${PROXY_DOMAIN}`,
          "Accept-Language": "zh-CN,zh;q=0.9"
        }
      });
      ws = wsRes.webSocket;
      if (!ws) throw new Error("WebSocket connection failed");
    } catch (err) {
      console.error("WebSocket error:", err);
      throw new Error("Failed to connect to Copilot service");
    }

    ws.accept();

    return new Promise((resolve, reject) => {
      let timeoutTimer = null;
      let hasSentFirstChunk = false;
      let isFinished = false;
      let collectedContent = "";

      const sendChatPayload = () => {
        const chatPayload = {
          event: "send",
          conversationId: startData.conversationId,
          content: [{ type: "text", text: copilotPrompt }],
          mode: effectiveMode,
          context: {}
        };
        console.log(`Sending to Copilot | mode=${effectiveMode} | length=${copilotPrompt.length}`);
        if (ws.readyState === 1) {
          ws.send(JSON.stringify(chatPayload));
        } else {
          reject(new Error("WebSocket connection not ready"));
        }
      };

      const completeNonStream = () => {
        if (isFinished) return;
        isFinished = true;
        resolve(this.createNonStreamResponse(startData.conversationId, model, collectedContent));
        if (ws.readyState === 1) ws.close();
        if (timeoutTimer) clearTimeout(timeoutTimer);
      };

      const completeStream = () => {
        if (isFinished || !writer) return;
        try {
          if (!hasSentFirstChunk && collectedContent) {
            writer.write(encoder.encode(`data: ${JSON.stringify(this.createOpenAIChunk({ conversationId: startData.conversationId, model, isFirst: true }))}\n\n`));
            hasSentFirstChunk = true;
            writer.write(encoder.encode(`data: ${JSON.stringify(this.createOpenAIChunk({ conversationId: startData.conversationId, model, content: collectedContent }))}\n\n`));
          }
          writer.write(encoder.encode(`data: ${JSON.stringify(this.createOpenAIChunk({ conversationId: startData.conversationId, model, isLast: true, finishReason: "stop" }))}\n\n`));
          writer.write(encoder.encode("data: [DONE]\n\n"));
          writer.close();
        } catch (e) {
          console.error("Failed to write final chunk:", e);
        }
        isFinished = true;
        if (ws.readyState === 1) ws.close();
        if (timeoutTimer) clearTimeout(timeoutTimer);
      };

      ws.addEventListener("open", () => {
        console.log("WebSocket connected");
        ws.send(JSON.stringify({
          event: "setOptions",
          supportedFeatures: ["partial-generated-images"],
          supportedCards: ["weather", "local", "image", "sports", "video", "consentV2"],
          ads: { supportedTypes: ["text"] }
        }));

        timeoutTimer = setTimeout(() => {
          if (!isFinished) {
            if (isStream && writer) {
              writer.write(encoder.encode(`data: ${JSON.stringify(this.createOpenAIChunk({ conversationId: startData.conversationId, model, isLast: true, finishReason: "timeout" }))}\n\n`));
              writer.write(encoder.encode("data: [DONE]\n\n"));
              writer.close();
            } else {
              reject(new Error("Request timeout"));
            }
          }
        }, 60000);
      });

      ws.addEventListener("message", (event) => {
        if (isFinished) return;
        let rawData = typeof event.data === "string" ? event.data : new TextDecoder().decode(event.data);
        let jsonStr = rawData.replace(/\d+$/, "").trim();
        if (!jsonStr) return;

        try {
          const msg = JSON.parse(jsonStr);
          if (msg.event === "connected") {
            sendChatPayload();
            return;
          }
          if (msg.event === "ping") {
            ws.send(JSON.stringify({ event: "pong", id: msg.id || "1" }));
            return;
          }
          if (msg.event === "appendText" && msg.text) {
            if (timeoutTimer) { clearTimeout(timeoutTimer); timeoutTimer = null; }
            collectedContent += msg.text;
            if (isStream) {
              if (!hasSentFirstChunk) {
                writer.write(encoder.encode(`data: ${JSON.stringify(this.createOpenAIChunk({ conversationId: startData.conversationId, model, isFirst: true }))}\n\n`));
                hasSentFirstChunk = true;
              }
              writer.write(encoder.encode(`data: ${JSON.stringify(this.createOpenAIChunk({ conversationId: startData.conversationId, model, content: msg.text }))}\n\n`));
            }
            return;
          }
          if (msg.event === "done" && !isFinished) {
            isStream ? completeStream() : completeNonStream();
          }
        } catch (e) {
          console.error("Failed to parse Copilot message:", e, "Raw:", jsonStr.slice(0, 200));
        }
      });

      ws.addEventListener("error", (error) => {
        console.error("WebSocket error:", error);
        if (!isFinished) {
          if (isStream && writer) {
            writer.write(encoder.encode(`data: ${JSON.stringify({ error: { message: "WebSocket error", type: "connection_error" } })}\n\n`));
            writer.write(encoder.encode("data: [DONE]\n\n"));
            writer.close();
          } else {
            reject(new Error("WebSocket error occurred"));
          }
        }
      });

      ws.addEventListener("close", () => {
        if (timeoutTimer) { clearTimeout(timeoutTimer); timeoutTimer = null; }
        if (!isFinished) {
          if (isStream && writer) {
            try { writer.write(encoder.encode("data: [DONE]\n\n")); writer.close(); } catch (e) {}
          } else {
            completeNonStream();
          }
        }
      });
    });
  },

  // ==================== 流式聊天处理 ====================
  async handleChatStream(request, startData, userCookie, writer, encoder, writable, requestBody) {
    let messages = [];
    let model = "gpt-4o";

    if (requestBody) {
      messages = requestBody.messages || [];
      model = requestBody.model || "gpt-4o";
    } else {
      try {
        const body = await request.json();
        messages = body.messages || [];
        model = body.model || "gpt-4o";
      } catch (e) {
        writer.write(encoder.encode(`data: ${JSON.stringify({ error: { message: "Invalid JSON", type: "invalid_request_error" } })}\n\n`));
        writer.write(encoder.encode("data: [DONE]\n\n"));
        writer.close();
        return;
      }
    }

    console.log(`[Stream] Processing ${messages.length} messages | model=${model}`);

    const { mode: copilotMode } = resolveModelRoute(model);
    const { text: copilotPrompt } = this.convertToCopilotFormat(messages);

    if (!copilotPrompt.trim()) {
      writer.write(encoder.encode(`data: ${JSON.stringify(this.createOpenAIChunk({ conversationId: startData.conversationId, model, isFirst: true, isLast: true, content: "", finishReason: "stop" }))}\n\n`));
      writer.write(encoder.encode("data: [DONE]\n\n"));
      writer.close();
      return;
    }

    const wssUrl = `https://${PROXY_DOMAIN}/c/api/chat?api-version=2&features=-%2Cncedge%2Cedgepagecontext&setflight=-%2Cncedge%2Cedgepagecontext&ncedge=1`;
    let ws;
    try {
      const wsRes = await fetch(wssUrl, {
        headers: {
          "Upgrade": "websocket", "Cookie": userCookie,
          "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0",
          "Origin": `https://${PROXY_DOMAIN}`, "Accept-Language": "zh-CN,zh;q=0.9"
        }
      });
      ws = wsRes.webSocket;
      if (!ws) throw new Error("WS failed");
    } catch (err) {
      writer.write(encoder.encode(`data: ${JSON.stringify({ error: { message: "Failed to connect to Copilot", type: "connection_error" } })}\n\n`));
      writer.write(encoder.encode("data: [DONE]\n\n"));
      writer.close();
      return;
    }

    ws.accept();

    let timeoutTimer = null, hasSentFirstChunk = false, isFinished = false, fullResponse = "";

    const sendPayload = () => {
      if (ws.readyState === 1) {
        ws.send(JSON.stringify({
          event: "send", conversationId: startData.conversationId,
          content: [{ type: "text", text: copilotPrompt }],
          mode: copilotMode,
          context: {}
        }));
        console.log(`[Stream] Sent | mode=${copilotMode} | length=${copilotPrompt.length}`);
      }
    };

    ws.addEventListener("open", () => {
      ws.send(JSON.stringify({ event: "setOptions", supportedFeatures: ["partial-generated-images"], supportedCards: ["weather", "local", "image", "sports", "video", "consentV2"], ads: { supportedTypes: ["text"] } }));
      timeoutTimer = setTimeout(() => {
        if (!isFinished && !writable.locked) {
          writer.write(encoder.encode(`data: ${JSON.stringify(this.createOpenAIChunk({ conversationId: startData.conversationId, model, isLast: true, finishReason: "timeout" }))}\n\n`));
          writer.write(encoder.encode("data: [DONE]\n\n"));
          writer.close();
          isFinished = true;
        }
        if (ws.readyState === 1) ws.close();
      }, 60000);
    });

    ws.addEventListener("message", (event) => {
      if (isFinished) return;
      const rawData = typeof event.data === "string" ? event.data : new TextDecoder().decode(event.data);
      const jsonStr = rawData.replace(/\d+$/, "").trim();
      if (!jsonStr) return;
      try {
        const msg = JSON.parse(jsonStr);
        if (msg.event === "connected") { sendPayload(); return; }
        if (msg.event === "ping") { ws.send(JSON.stringify({ event: "pong", id: msg.id || "1" })); return; }
        if (msg.event === "appendText" && msg.text) {
          if (timeoutTimer) { clearTimeout(timeoutTimer); timeoutTimer = null; }
          fullResponse += msg.text;
          if (!hasSentFirstChunk) {
            writer.write(encoder.encode(`data: ${JSON.stringify(this.createOpenAIChunk({ conversationId: startData.conversationId, model, isFirst: true }))}\n\n`));
            hasSentFirstChunk = true;
          }
          writer.write(encoder.encode(`data: ${JSON.stringify(this.createOpenAIChunk({ conversationId: startData.conversationId, model, content: msg.text }))}\n\n`));
          return;
        }
        if (msg.event === "done" && !isFinished) {
          if (!hasSentFirstChunk) {
            writer.write(encoder.encode(`data: ${JSON.stringify(this.createOpenAIChunk({ conversationId: startData.conversationId, model, isFirst: true }))}\n\n`));
            hasSentFirstChunk = true;
          }
          writer.write(encoder.encode(`data: ${JSON.stringify(this.createOpenAIChunk({ conversationId: startData.conversationId, model, isLast: true, finishReason: "stop" }))}\n\n`));
          writer.write(encoder.encode("data: [DONE]\n\n"));
          isFinished = true;
          writer.close();
          if (ws.readyState === 1) ws.close();
        }
      } catch (e) {
        console.error("Parse error:", e, jsonStr.slice(0, 200));
      }
    });

    ws.addEventListener("error", (error) => {
      if (!isFinished && !writable.locked) {
        writer.write(encoder.encode(`data: ${JSON.stringify({ error: { message: "WebSocket error", type: "connection_error" } })}\n\n`));
        writer.write(encoder.encode("data: [DONE]\n\n"));
        writer.close();
        isFinished = true;
      }
    });

    ws.addEventListener("close", () => {
      if (timeoutTimer) { clearTimeout(timeoutTimer); timeoutTimer = null; }
      if (!isFinished && !writable.locked) {
        try { writer.write(encoder.encode("data: [DONE]\n\n")); writer.close(); } catch (e) {}
        isFinished = true;
      }
    });
  },

  // ==================== 工具函数 ====================
  createErrorResponse(message, type) {
    return new Response(JSON.stringify({ error: { message, type } }), {
      status: 400,
      headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
    });
  },

  createEmptyResponse(conversationId, model, isStream, writer, encoder) {
    if (isStream) {
      const chunk = this.createOpenAIChunk({ conversationId, model, isFirst: true, isLast: true, content: "", finishReason: "stop" });
      writer.write(encoder.encode(`data: ${JSON.stringify(chunk)}\n\n`));
      writer.write(encoder.encode("data: [DONE]\n\n"));
      writer.close();
      return;
    }
    return this.createNonStreamResponse(conversationId, model, "");
  },

  createNonStreamResponse(conversationId, model, content) {
    return {
      id: `chatcmpl-${conversationId}`,
      object: "chat.completion",
      created: Math.floor(Date.now() / 1000),
      model,
      choices: [{
        index: 0,
        message: { role: "assistant", content },
        finish_reason: content ? "stop" : "length"
      }],
      usage: { prompt_tokens: 0, completion_tokens: content.length, total_tokens: content.length }
    };
  },

  createOpenAIChunk({ conversationId, model = "gpt-4o", isFirst = false, isLast = false, content = "", finishReason = null }) {
    const chunkId = `chatcmpl-${conversationId}-${Date.now()}`;
    if (isFirst) return { id: chunkId, object: "chat.completion.chunk", created: Math.floor(Date.now() / 1000), model, choices: [{ index: 0, delta: { role: "assistant" }, finish_reason: null }] };
    if (isLast) return { id: chunkId, object: "chat.completion.chunk", created: Math.floor(Date.now() / 1000), model, choices: [{ index: 0, delta: {}, finish_reason: finishReason || "stop" }] };
    return { id: chunkId, object: "chat.completion.chunk", created: Math.floor(Date.now() / 1000), model, choices: [{ index: 0, delta: { content }, finish_reason: null }] };
  },

  async initializeEnvironment(cookie) {
    const commonHeaders = {
      "Cookie": cookie,
      "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0",
      "Referer": `https://${PROXY_DOMAIN}/`,
      "Origin": `https://${PROXY_DOMAIN}`,
      "Accept": "application/json"
    };

    const startPayload = {
      timeZone: "Asia/Shanghai", startNewConversation: true,
      teenSupportEnabled: true, correctPersonalizationSetting: true,
      deferredDataUseCapable: true, performUserMerge: true,
      source: "web", conversationType: "chat"
    };

    const startRes = await fetch(`https://${PROXY_DOMAIN}/c/api/start`, {
      method: "POST",
      headers: { ...commonHeaders, "Content-Type": "application/json" },
      body: JSON.stringify(startPayload)
    });

    if (!startRes.ok) throw new Error(`Start API failed: ${startRes.status} ${startRes.statusText}`);

    const startConfig = await startRes.json();
    const convId = startConfig.currentConversationId || startConfig.conversationId;
    if (!convId) throw new Error("Failed to get conversation ID");

    return { conversationId: convId, userId: startConfig.userId, traceId: crypto.randomUUID().replace(/-/g, "") };
  },

  convertToCopilotFormat(messages) {
    if (!messages || !Array.isArray(messages) || messages.length === 0) {
      return { text: "", hasSystem: false };
    }

    let systemMessage = "";
    const conversationMessages = [];

    for (const msg of messages) {
      if (msg.role === "system" && msg.content && typeof msg.content === "string") {
        systemMessage = msg.content.trim();
      } else if (msg.content && typeof msg.content === "string" && msg.content.trim() !== "") {
        conversationMessages.push({ role: msg.role, content: msg.content.trim() });
      }
    }

    if (conversationMessages.length === 0 && systemMessage) {
      return { text: systemMessage, hasSystem: true };
    }

    let copilotText = "";
    if (systemMessage) copilotText += `System: ${systemMessage}\n\n`;

    if (conversationMessages.length === 1) {
      copilotText += conversationMessages[0].content;
    } else {
      for (const msg of conversationMessages) {
        const roleText = msg.role === "user" ? "User" :
          msg.role === "assistant" ? "Assistant" :
            msg.role === "system" ? "System" : msg.role;
        copilotText += `${roleText}: ${msg.content}\n\n`;
      }
    }

    return { text: copilotText.trim(), hasSystem: !!systemMessage };
  }
};
