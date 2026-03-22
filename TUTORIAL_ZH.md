---
title: OpenClaw 本地 AI 语音配置教程
date: 2026-03-22
tags:
  - openclaw
  - tts
  - voicebox
  - tutorial
---

# OpenClaw 本地 AI 语音配置完全指南

本教程将指导你使用 Voicebox + 自定义代理实现 OpenClaw 的本地 TTS（文本转语音）功能，支持声音克隆。

## 前置条件

在开始之前，请确保你已经准备好以下内容：

- ✅ **OpenClaw** - AI 助手应用
- ✅ **AI 模型** - 例如 Kimi 等任意模型
- ✅ **Python 环境** - 用于运行代理服务
- ✅ **FastAPI & Uvicorn** - Python web 框架（安装：`pip install fastapi uvicorn requests`）

## 阶段 1: 安装和配置 Voicebox

### 第1步：下载 Voicebox

访问 [Voicebox GitHub](https://github.com/jamiepine/voicebox) 下载最新版本。

### 第2步：下载 TTS 模型

从 [Hugging Face](https://huggingface.co) 手动下载 Qwen 的 TTS 模型。

### 第3步：配置 Voicebox 启动参数

启动 Voicebox 应用后，在设置中勾选以下两项：

1. **Keep server running when app closes** - 保持后台服务运行
2. **Allow network access** - 允许网络访问

> [!warning] 重要提示
> 这两项设置对正常使用至关重要，不可遗漏。

---

## 阶段 2: 声音克隆（可选但推荐）

如果你想使用特定人物的声音，需要准备：

- 📁 **音频文件** - 一段干净、清晰的语音样本
- 📝 **音频台词** - 该音频的完整文字内容

### 获取音频样本

#### 方法一：从 Fish Audio 提取（推荐）

Fish Audio 上有丰富的语音样本库，你可以直接提取音频：

1. 访问 [Fish Audio](https://www.fish-audio.com/) 找到想要的声音片段
2. 打开浏览器开发者工具（按 `F12`）
3. 切换到 **网络（Network）** 标签页
4. **刷新页面**（Ctrl + R 或 Cmd + R）
5. 在网络请求中筛选 **媒体（Media）** 类型
6. 找到对应的 `.mp3` 文件，右键 → **在新标签页中打开** 或 **另存为**

> [!tip] 获取台词
> 复制页面中展示的文字内容作为该音频的台词，这样克隆效果会更好。

#### 方法二：使用本地录音

如果是你自己或朋友的声音，直接录制一段清晰的语音样本（推荐 10-30 秒）。

---

在 Voicebox 中完成声音克隆过程，系统会生成一个唯一的 **Profile ID**。

### 获取 Profile ID

Voicebox 运行在 `http://127.0.0.1:17493` 端口，访问以下地址查看已克隆的模型：

```
http://localhost:17493/profiles
```

> [!caution] 注意
> 记下模型的 **ID**，而不是 Name。后续配置时需要使用 ID。

---

## 阶段 3: 创建 TTS 代理服务

### 第4步：创建 Python 脚本

创建文件 `voicebox_proxy.py`，内容如下：

```python
from fastapi import FastAPI, Request
from fastapi.responses import Response
import requests
import base64
import uvicorn

app = FastAPI()

VOICEBOX_URL = "http://localhost:17493/generate"
PROFILE_ID = "这里填上对应克隆语音的id"  # ← 替换为你的 Profile ID

@app.post("/v1/audio/speech")
async def tts(request: Request):
    data = await request.json()
    text = data.get("input", "")

    payload = {
        "text": text,
        "profile_id": PROFILE_ID,
        "language": "en"  # 可选：支持其他语言，如 "zh"
    }

    resp = requests.post(VOICEBOX_URL, json=payload, timeout=30)
    resp_json = resp.json()

    # Voicebox 返回 BASE64 编码的音频
    audio_base64 = resp_json.get("audio", "")
    audio_bytes = base64.b64decode(audio_base64)

    return Response(content=audio_bytes, media_type="audio/mpeg")


if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8880)
```

### 第5步：启动代理服务

```bash
python voicebox_proxy.py
```

服务将在 `http://localhost:8880` 启动。

---

## 阶段 4: 配置 OpenClaw

### 第6步：配置 TTS 设置

在终端中执行以下命令配置 OpenClaw：

```bash
openclaw config set messages.tts.auto "always"
openclaw config set messages.tts.provider "openai"
openclaw config set messages.tts.openai.apiKey "sk-随便填"
openclaw config set messages.tts.openai.baseUrl "http://localhost:8880/v1"
openclaw config set messages.tts.openai.model "tts-1"
openclaw config set messages.tts.openai.voice "default"
openclaw gateway restart
```

> [!tip] 参数说明
> - `auto: "always"` - 每条回复都生成语音（适合讲故事场景）
> - `auto: "tagged"` - 仅生成标记为 TTS 的内容
> - `apiKey` - 可随意填写，代理会忽略
> - `voice` - 字段会被忽略，使用 Profile ID 控制

### 第7步：在 OpenClaw 中使用 TTS

如果将 `tts.auto` 设置为 `tagged`，使用以下格式标记需要语音的文本：

```
[[tts:text]]这里放你要朗读的文字[[/tts:text]]
```

---

## 阶段 5: 测试和验证

### 第8步：测试音频生成

在终端中执行以下 curl 命令测试代理是否正常工作：

```bash
curl -X POST "http://localhost:8880/v1/audio/speech" \
  -H "Content-Type: application/json" \
  -d "{\"input\":\"Hello world\"}" \
  --output test.mp3
```

### 第9步：验证输出

1. 检查 `test.mp3` 是否已在 Voicebox 处理队列中
2. 在文件资源管理器中打开 `test.mp3`
3. 确认音频能正常播放

> [!success] 成功标志
> 能够正常播放生成的 MP3 文件，说明整个流程配置正确。

---

## 阶段 6: 可选 - 开机自启动

### 第10步：创建启动脚本

创建文件 `start_tts.bat`：

```batch
@echo off
echo 正在启动 Voicebox + 代理...

:: 修改为你的 Voicebox.exe 完整路径
start "" /min "D:\Voicebox\Voicebox.exe"

timeout /t 8 >nul     :: 等待 8 秒让 Voicebox 完全启动

:: 修改为你的 voicebox_proxy.py 完整路径
start "" /min cmd /c "python D:\tts\voicebox_proxy.py"

echo 启动完成！（窗口会自动最小化）
```

> [!note] 路径修改
> 根据你的实际安装位置修改：
> - `D:\Voicebox\Voicebox.exe` → 你的 Voicebox 应用路径
> - `D:\tts\voicebox_proxy.py` → 你的代理脚本路径

### 第11步：添加到系统启动

1. 按 `Win + R` 打开运行对话框
2. 输入 `shell:startup` 并回车（打开"启动"文件夹）
3. 将 `start_tts.bat` 拖入或复制到此文件夹

✅ 完成！系统重启后，Voicebox 和代理服务将自动启动。

---

## 故障排查

| 问题 | 解决方案 |
|------|--------|
| 代理无法连接 Voicebox | 检查 Voicebox 是否运行，确认端口 17493 可访问 |
| 生成的音频质量差 | 在 Voicebox 中重新克隆声音，确保输入音频清晰 |
| OpenClaw 未生成语音 | 运行测试 curl 命令，检查代理是否正常响应 |
| 启动脚本失效 | 检查路径是否正确，确保 Python 环境可用 |

---

## 相关链接

- 🔗 [Voicebox GitHub](https://github.com/jamiepine/voicebox)
- 🔗 [Hugging Face TTS 模型](https://huggingface.co/Qwen/Qwen3-TTS-12Hz-1.7B-Base/tree/main)
- 🔗 [FastAPI 文档](https://fastapi.tiangolo.com/)
- 🔗 [Fish Audio](https://www.fish-audio.com/)
