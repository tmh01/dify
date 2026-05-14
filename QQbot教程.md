# 基于 NoneBot2 + NapCatQQ + Dify 的学术助手 QQ 机器人搭建教程

本教程将指导你从零开始搭建一个接入大语言模型（Dify）的 QQ 机器人。机器人可读取用户指令，调用 Dify 工作流/应用检索学术论文，并返回总结报告。

> 注意：本文中的示例以 OneBot V11 + NapCatQQ + NoneBot2 为主。请勿把 Dify API Key 提交到公共仓库。

---

## 目录

- [第一阶段：准备工作](#第一阶段准备工作)
- [第二阶段：配置 Dify（AI 大脑）](#第二阶段配置-difyai-大脑)
- [第三阶段：配置 NapCatQQ（机器人躯体）](#第三阶段配置-napcatqq机器人躯体)
- [第四阶段：配置 NoneBot2（神经中枢）](#第四阶段配置-nonebot2神经中枢)
- [第五阶段：核心代码编写（含抗超时设计）](#第五阶段核心代码编写含抗超时设计)
- [第六阶段：运行与测试](#第六阶段运行与测试)
- [常见排坑指南（Troubleshooting）](#常见排坑指南troubleshooting)
- [安全建议](#安全建议)

---

## 第一阶段：准备工作

### 1. 环境准备

- 操作系统已安装 **Python 3.10 或以上**（建议 3.10/3.11）。
- 建议使用虚拟环境（`venv` / Anaconda）。

示例（venv）：

```bash
python -m venv .venv
# Windows
.venv\Scripts\activate
# macOS/Linux
source .venv/bin/activate
```

### 2. 账号准备

- 注册并登录 **Dify**（云端版即可）。
- 准备一个用于作为机器人的 **QQ 小号**。

### 3. 安装基础依赖

后续插件会用到网络请求库 `httpx`：

```bash
pip install httpx
```

---

## 第二阶段：配置 Dify（AI 大脑）

Dify 负责处理业务逻辑（例如：搜索论文、调用大模型总结、返回最终文本）。

### 1. 创建应用

在 Dify 工作区创建一个：

- “聊天助手 (Chat App)” 或
- “工作流 (Chatflow)”

两者都可以，只要最终能通过 API 调用 `/v1/chat-messages`。

### 2. 清理入参变量（极其关键）

> 这一点是为了避免 API 调用时出现参数冲突，常见报错为 **400 Bad Request**。

操作步骤：

1. 进入你的 Dify 应用/工作流的“编排/Workflow”页面。
2. 点击最左侧的 **「开始」(Start)** 节点。
3. 检查节点中的输入字段：如果预设了 `query`、`files` 等输入变量，请 **全部删除**。
4. 删除后，务必点击右上角 **「发布」(Publish)**。

说明：
- 如果不删除这些自定义变量，后续通过 API 调用时可能会出现“同名参数重复/冲突”，从而导致 400。

### 3. 获取 API Key

操作步骤：

1. 进入应用页面左侧菜单。
2. 点击 **「API 访问」 -> 「API 密钥」**。
3. 点击 **生成新的密钥**。
4. 复制以 `app-` 开头的字符串备用。

> 强烈建议：把 Key 放到 `.env.prod`（或 `.env`）里，不要写死在代码中，更不要提交到 GitHub。

---

## 第三阶段：配置 NapCatQQ（机器人躯体）

NapCatQQ 负责监听 QQ 消息，并通过 WebSocket 转发给 NoneBot（OneBot V11 适配器）。

### 1. 登录 QQ 小号

1. 启动 NapCatQQ 主程序。
2. 出现二维码后，使用准备好的 QQ 小号扫码登录。

### 2. 配置「反向 WebSocket」到 NoneBot

> 目标：NapCatQQ 作为 WebSocket Client，连接到 NoneBot 提供的 WS 服务。

操作步骤：

1. 登录成功后，打开 NapCat 的 WebUI 管理后台（通常是 `http://127.0.0.1:6099/webui`）。
2. 点击左侧菜单 **「网络配置」**。
3. 点击右上角 **「新建」**。
4. 选择 **Websocket Client**。
5. 填写：
   - URL：`ws://127.0.0.1:8080/onebot/v11/ws`
   - 勾选：**启用**
   - 消息格式：**Array**
6. 点击保存。

> 提醒：
> - `127.0.0.1:8080` 需要与你的 NoneBot 启动端口一致（下面 FastAPI 默认是 8080）。
> - 如果你的 NoneBot 部署在另一台机器上，要把 `127.0.0.1` 改成对应机器的局域网 IP / 公网地址。

---

## 第四阶段：配置 NoneBot2（神经中枢）

NoneBot 负责接收 NapCat 转发的消息，执行 Python 插件逻辑，并调用 Dify API。

### 1. 初始化项目

安装脚手架并创建项目：

```bash
pip install nb-cli
nb create
```

交互式选择：

- 项目名称：`qq_bot`（或你喜欢的名字）
- Driver：选择 **FastAPI**
- Adapter：选择 **OneBot V11**

### 2. 配置 COMMAND_START（支持斜杠指令）

打开项目根目录的 `.env.prod`（或 `.env`），确保包含：

```env
COMMAND_START=["/", ""]
```

### 3. 配置 Dify Key（推荐用环境变量）

同样在 `.env.prod`（或 `.env`）中加入：

```env
DIFY_API_URL=https://api.dify.ai/v1/chat-messages
DIFY_API_KEY=app-xxxxxxxxxxxxxxxxxxxxxxxx
```

> 注意：URL 必须以 `/chat-messages` 结尾。

### 4. 创建插件文件

在 `src/plugins/` 目录下新建文件：

- `src/plugins/dify_academic.py`

---

## 第五阶段：核心代码编写（含抗超时设计）

将以下代码完整复制到 `src/plugins/dify_academic.py`。

本代码包含两项关键优化：

1. **Streaming 流式请求**：防止 Dify 搜索/检索耗时过长触发网关 60 秒断开导致 **504 Gateway Timeout**。
2. **NoneBot 异常隔离**：避免把 NoneBot 的 `finish()` 结束信号当成真实异常捕获。

```python
# src/plugins/dify_academic.py

import json
import os
import httpx

from nonebot import on_command
from nonebot.adapters.onebot.v11 import MessageEvent, Message
from nonebot.params import CommandArg
from nonebot.log import logger

# ================= 配置区 =================
# 注意：URL 必须以 /chat-messages 结尾
DIFY_API_URL = os.getenv("DIFY_API_URL", "https://api.dify.ai/v1/chat-messages")
# 替换为你的 Dify 应用 API Key（推荐放在 .env / .env.prod）
DIFY_API_KEY = os.getenv("DIFY_API_KEY", "")
# ==========================================

user_conversations = {}
academic_bot = on_command("学术", aliases={"查论文", "解析"}, priority=5, block=True)


@academic_bot.handle()
async def handle_dify_request(event: MessageEvent, args: Message = CommandArg()):
    if not DIFY_API_KEY:
        await academic_bot.finish("未检测到 DIFY_API_KEY，请在 .env/.env.prod 中配置后重试。")

    user_query = args.extract_plain_text().strip()
    if not user_query:
        await academic_bot.finish("请输入你要查询的内容，例如：/学术 帮我找关于大模型的论文")

    user_id = str(event.user_id)
    await academic_bot.send("收到。学术大脑正在检索，长篇解析可能需要30-60秒，请稍候...")

    headers = {
        "Authorization": f"Bearer {DIFY_API_KEY}",
        "Content-Type": "application/json",
    }

    payload = {
        # 说明：这里把 query 放在 inputs 里，同时也放在 query 字段中，是为了兼容不同应用形态
        # 但前提是你已经在 Dify 的开始节点删除了自定义 inputs 变量（否则可能冲突 400）
        "inputs": {"query": user_query, "files": []},
        "query": user_query,
        "response_mode": "streaming",  # 使用流式接收防止 504 超时
        "user": user_id,  # 必须为字符串类型
    }

    if user_id in user_conversations:
        payload["conversation_id"] = user_conversations[user_id]

    final_output = ""
    error_msg = ""

    try:
        # 设置合理的超时时间（流式情况下整体可更长）
        async with httpx.AsyncClient(timeout=300.0) as client:
            full_answer = ""
            new_conv_id = None

            async with client.stream(
                "POST", DIFY_API_URL, headers=headers, json=payload
            ) as response:
                response.raise_for_status()

                async for line in response.aiter_lines():
                    if not line.startswith("data: "):
                        continue

                    try:
                        data = json.loads(line[6:])
                    except json.JSONDecodeError:
                        continue

                    if data.get("event") == "message":
                        full_answer += data.get("answer", "")
                    elif data.get("event") == "message_end":
                        new_conv_id = data.get("conversation_id")

            if new_conv_id:
                user_conversations[user_id] = new_conv_id

            final_output = full_answer

    except httpx.TimeoutException:
        error_msg = "处理超时，请尝试缩小查询范围。"
    except Exception as e:
        logger.error(f"Dify 接口调用真实异常: {e}")
        error_msg = f"连接学术大脑失败，请检查后台。错误: {e}"

    # 在 try-except 代码块外部执行发送逻辑，防止拦截 NoneBot 正常结束异常
    if error_msg:
        await academic_bot.finish(error_msg)

    if final_output:
        await academic_bot.finish(final_output)
    else:
        await academic_bot.finish("学术大脑没有返回任何内容，请重试。")


# ================= 清空记忆功能 =================
clear_bot = on_command("清空记忆", priority=5, block=True)


@clear_bot.handle()
async def handle_clear(event: MessageEvent):
    user_id = str(event.user_id)
    if user_id in user_conversations:
        del user_conversations[user_id]
        await clear_bot.finish("学术大脑已重置，可以开始新的话题。")
    else:
        await clear_bot.finish("目前没有需要清空的记忆。")
```

---

## 第六阶段：运行与测试

在 NoneBot 项目目录运行：

```bash
nb run
```

如果看到类似输出：

- `Loaded adapters: OneBot V11`
- 应用已启动且无报错日志

表示 NoneBot 已准备就绪。

测试：用任意 QQ 号向机器人发送：

- `/学术 检索 2 篇深度学习的相关论文`

等待 30～60 秒左右，应能收到 Dify 返回的总结长文。

---

## 常见排坑指南（Troubleshooting）

- **401 Unauthorized**：API Key 错误。请重新在 Dify 生成以 `app-` 开头的密钥；检查 `Bearer` 后是否多空格。
- **404 Not Found**：接口路径错误。确保 `DIFY_API_URL` 以 `/chat-messages` 结尾。
- **400 Bad Request**：Payload 不符合要求。常见原因：
  - Dify 工作流开始节点手动添加了其他变量（未删除 `query/files` 等），导致参数冲突。
  - `user` 参数未转成字符串（必须 `str()`）。
- **504 Gateway Timeout**：长连接超时。使用 `client.stream` 流式逐行接收，避免 60 秒网关强制断开。
- **FinishedException()**：NoneBot 的正常结束信号，不是实际报错。把 `finish()` 放到 `try...except` 外侧执行可避免误捕获。

---

## 安全建议

- 不要把 `DIFY_API_KEY` 写死在代码中。
- 推荐使用 `.env.prod` / `.env` 注入环境变量，并把 `.env*` 加入 `.gitignore`。
- 如需给他人参考，建议提供 `.env.example`（仅占位不含密钥）。