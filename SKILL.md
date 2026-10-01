---
name: uusscribe-asr
description: 使用 UUScribe 将本地音频文件或公开的 YouTube 单视频转成文字。用户要求语音识别、音频转写、YouTube 转录、字幕文本或说话人分段时使用。
metadata:
  author: UUScribe
  version: "1.0.0"
  openclaw:
    requires:
      env:
        - UUSCRIBE_API_KEY
      bins:
        - python3
    primaryEnv: UUSCRIBE_API_KEY
    envVars:
      - name: UUSCRIBE_API_BASE
        required: false
        description: Optional self-hosted or development UUScribe API URL.
---

# UUScribe ASR

通过 UUScribe API 创建真实转录任务。核心说明保持 Agent 中立；平台专属元数据放在 `agents/`。

## 适用边界

- 接受本地音频文件，或公开、无需登录的 YouTube 单视频 URL。
- 不接受任意网页 URL、播放列表、频道、私密/会员/付费 YouTube 内容或 Cookie。
- 本地文件会上传到 UUScribe，任务会消耗用户账户额度。用户明确要求转写具体文件或链接即构成提交授权；请求含糊时，先说明这两点再提交。
- 只处理用户提供或确认有权处理的内容。

## 前置检查

1. 运行环境需要 Python 3.9+ 和网络访问；本地音频还需要 `ffprobe`。
2. 从环境变量读取 `UUSCRIBE_API_KEY`，不要要求用户在聊天或命令参数中粘贴密钥，也不要打印它。
3. 默认 API 地址是 `https://scribe.uustudio.cc`。仅在用户明确给出自托管或开发环境时设置 `UUSCRIBE_API_BASE`。
4. 本地音频需要 `ffprobe` 获取真实时长；不要猜测时长。若用户准确提供秒数，可用 `--duration`。
5. API Key 所属账户必须具备 UUScribe API 权限，并受账户余额、并发和上传限制约束。
6. 默认只连接 HTTPS API；仅 localhost 开发地址允许 HTTP，认证请求不会跟随重定向。

## 执行

先将 `scripts/transcribe.py` 解析为相对于本文件的绝对路径，再运行：

```bash
python3 /absolute/path/to/uusscribe-asr/scripts/transcribe.py "/path/to/meeting.mp3"
python3 /absolute/path/to/uusscribe-asr/scripts/transcribe.py "https://www.youtube.com/watch?v=VIDEO_ID"
```

长音频或 Agent 执行时间受限时，优先采用可恢复的两阶段流程。先提交并保存 JSON 中的 `task_id`，随后凭 ID 恢复轮询：

```bash
python3 /absolute/path/to/uusscribe-asr/scripts/transcribe.py "/path/to/meeting.mp3" --submit-only --quiet
python3 /absolute/path/to/uusscribe-asr/scripts/transcribe.py --task-id TASK_ID --quiet
```

常用选项：

```bash
--name "任务名称"       # 可选任务名
--format json           # 默认；保留时间戳、说话人、摘要等结构
--format text           # 仅输出纯文本
--output /path/result   # 同时写入指定文件
--timeout 7200          # 最长轮询秒数；超时不会取消远端任务
--duration 123          # ffprobe 不可用时，由用户提供准确秒数
--submit-only           # 仅提交并输出可保存的任务回执
--task-id 123           # 恢复轮询已有任务，避免重复提交和计费
--quiet                 # stdout 只保留机器可解析结果
```

脚本对本地音频执行分片上传并主动节流，对 YouTube 直接创建远端任务；两者随后轮询到 `finished` 或 `failed`。安全的签名、分片 PUT 和轮询请求遇到临时网络错误、`429` 或 `5xx` 会退避重试；可能创建任务的请求不会自动重试。不要在脚本外另行使用 yt-dlp 下载 YouTube 内容。

## 返回结果

- 成功时向用户返回转录文本、任务 ID、识别语言；用户需要说话人或时间戳时使用 JSON 中的 `segments`。`start`/`end` 单位为毫秒；先查看 `speaker_labels_present` 和覆盖数量，缺失时明确说明，不猜测说话人。
- 不主动展示带签名的分享链接，不回显本地绝对路径或 API Key。
- 轮询超时只表示等待结束，远端任务仍可能继续；报告任务 ID，避免重新创建重复任务。
- 若创建或上传完成请求断线且没有返回任务 ID，结果属于“不确定”，不要自动重提；先让用户在 UUScribe 任务列表核对。服务端目前没有供 Skill 使用的幂等键。
- 本地音频在上传前检查可用余额；YouTube 得到真实时长后若因余额不足停在 `pending`，脚本会报告所需与可用秒数，远端任务仍保留。
- `401`：API Key 无效；`403`：账户权限或每日额度限制；`422`：文件/URL/参数不受支持；`429`：并发或频率限制。以服务端实际消息为准，不编造成功结果。
- 创建请求不要自动重试，以免产生重复计费任务。轮询 GET 遇到短暂 `429` 或 `5xx` 可以在总超时内退避重试。
