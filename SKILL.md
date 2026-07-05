---
name: "chenmo-video-kb"
description: "尘墨·视频知识采集：下载B站/YouTube等视频→提取音频→Whisper转录→AI提炼→结构化归档。触发：视频下载/转文字/提炼。"
---

# 尘墨·视频知识采集 Skill

> 🏷️ **尘墨出品** — 从视频到知识的结构化采集闭环

当大哥发送视频链接（B站/YouTube/其他平台）并要求下载、转文字、提炼、归档时，自动触发本 Skill。也适用于已有视频文件需要做知识提取的场景。

---

## 设计哲学（与老章 z-skills 的关键差异）

| 维度 | 老章 z-video-downloader | 尘墨·视频知识采集 |
|------|------------------------|-------------------|
| 独立性 | 依赖 z-web-pack 的清单文件 | **完全独立**，直接接收链接 |
| 处理链 | 只下载视频文件 | 下载 → 音频 → 转录 → **AI提炼** → 归档 |
| 输出 | 本地视频文件 | 结构化知识文档（.md）+ 封面图 + 可选视频文件 |
| 平台覆盖 | yt-dlp 全平台 | yt-dlp 全平台 + B站API深度集成 |
| 归档 | 无 | 自动归入 knowledge/学习资源/ 含分类 |
| 品牌 | 无 | 尘墨标识 + 统一命名规范 |
| 鲁棒性 | 弱（硬编码） | 强（28项工程优化） |

---

## 安全红线（开始前必须检查）

⚠️ **以下内容拒绝处理：**
- 付费/会员专属内容（即使 yt-dlp 有能力下载）
- 明确标注"禁止转载/下载"的版权内容
- 涉及个人隐私的视频（如他人未公开的私人视频）
- 色情/违规内容

遇到以上情况 → 直接告知大哥：「尘墨不能处理该内容，涉及<原因>。」

---

## 前置依赖

```bash
# yt-dlp（多平台视频下载）
pip install yt-dlp

# ffmpeg（音视频转换，通常已预装）
brew install ffmpeg

# openai-whisper（语音转录，使用 base 模型）
pip install openai-whisper
```

---

## 第零步：依赖自检 + 模型预下载 + 磁盘检查 + 错误日志初始化 + 计时开始

### 0.1 依赖自检

```bash
echo "=== 尘墨依赖自检 ==="
command -v yt-dlp && echo "✅ yt-dlp" || echo "❌ yt-dlp 未安装，执行 pip install yt-dlp"
command -v ffmpeg && echo "✅ ffmpeg" || echo "❌ ffmpeg 未安装，执行 brew install ffmpeg"
command -v whisper && echo "✅ whisper" || echo "❌ whisper 未安装，执行 pip install openai-whisper"
```

如果有 ❌，自动安装缺失依赖后再继续。安装失败 → 告知大哥缺少哪个工具、手动安装命令。

### 0.2 Whisper base 模型预下载

避免首次转录时网络超时卡死：

```bash
if [ ! -d ~/.cache/whisper ] || [ ! "$(ls -A ~/.cache/whisper/*base* 2>/dev/null)" ]; then
    echo "📥 尘墨：首次使用，正在下载 Whisper base 模型（~140MB）..."
    python3 -c "import whisper; whisper.load_model('base')" && echo "✅ 模型就绪"
else
    echo "✅ Whisper base 模型已缓存"
fi
```

### 0.3 磁盘空间预检 ✅ P1-26

```bash
# 检查工作磁盘可用空间（至少需要 1GB 安全余量）
AVAIL_GB=$(df -g /tmp | awk 'NR==2 {print $4}')
if [ "$AVAIL_GB" -lt 1 ]; then
    echo "❌ 尘墨：磁盘可用空间不足 1GB（当前 ${AVAIL_GB}GB），无法安全处理视频。"
    echo "[$(date +%H:%M:%S)] 磁盘空间不足: ${AVAIL_GB}GB" >> "$CM_ERROR_LOG"
    exit 1
fi
echo "✅ 磁盘可用空间: ${AVAIL_GB}GB"
```

### 0.4 错误日志初始化

```bash
CM_ERROR_LOG="/tmp/cm_error_$$.log"
echo "=== 尘墨采集日志 PID=$$ 开始 $(date) ===" > "$CM_ERROR_LOG"
```

后续每一步的关键失败信息都追加到 `$CM_ERROR_LOG`：
```bash
echo "[$(date +%H:%M:%S)] <步骤描述> 失败: <原因>" >> "$CM_ERROR_LOG"
```

### 0.5 计时开始 ✅ P1-25

```bash
CM_START_TIME=$(date +%s)
```

处理完成后，在汇报中显示总耗时：
```bash
CM_END_TIME=$(date +%s)
CM_ELAPSED=$((CM_END_TIME - CM_START_TIME))
# 格式化输出：X分Y秒
```

---

## 第½步：网络环境预检

```bash
echo "=== 网络预检 ==="
curl -s -o /dev/null -w "%{http_code}" --connect-timeout 5 "https://api.bilibili.com" && echo " ✅ B站API可达" || echo " ⚠️ B站API不可达"
curl -s -o /dev/null -w "%{http_code}" --connect-timeout 5 "https://www.youtube.com" && echo " ✅ YouTube可达" || echo " ⚠️ YouTube不可达（可能需要代理）"
curl -s -o /dev/null -w "%{http_code}" --connect-timeout 5 "https://github.com" && echo " ✅ GitHub可达" || echo " ⚠️ GitHub不可达"
```

目标平台不可达时 → 提前告知大哥，避免后续全流程失败。

---

## 第一步：存量去重

在开始下载前，先检查 knowledge/学习资源/ 下是否已有同名归档：

```bash
find /Users/jinhanyu/.openclaw/workspace/knowledge/学习资源/ -name "尘墨-*" -type f 2>/dev/null
```

如果发现已有同视频归档 → 跳到「增量更新」场景处理。
如果无 → 继续正常流程。

---

## 第二步：判断平台与策略

| 平台 | 识别方式 | 下载策略 |
|------|---------|---------|
| B站 | `bilibili.com/video/BV` | B站 API 获取 dash 音频流 + curl 下载 |
| YouTube | `youtube.com/watch` 或 `youtu.be` | yt-dlp（含 cookie 兜底） |
| 其他 | 抖音/Vimeo/X/m3u8 等 | yt-dlp 通用策略 |

---

## 第三步：获取视频元信息（B站 API 错误码处理版）✅ P0-23

B站 API 通用返回格式：`{"code": 0, "message": "0", "data": {...}}`。必须先检查 `code`。

**B站专用：**
```bash
curl -s "https://api.bilibili.com/x/web-interface/view?bvid=<BV号>" | python3 -c "
import json,sys
resp = json.load(sys.stdin)
code = resp.get('code', -1)
msg = resp.get('message', '未知错误')

# B站错误码映射
ERROR_MAP = {
    -400: '请求错误，请检查BV号是否正确',
    -403: '权限不足，可能是大会员专属内容',
    -404: '视频不存在或已被删除/下架',
    62002: '视频不可见（被UP主设为私密）',
    62003: '视频审核中',
    62004: '视频已被删除',
}
err_desc = ERROR_MAP.get(code, f'未知错误(code={code}): {msg}')

if code != 0:
    print(f'BILIBILI_API_ERROR|{code}|{err_desc}')
    sys.exit(1)

d = resp['data']
print(f\"标题: {d['title']}\")
print(f\"UP主: {d['owner']['name']}\")
print(f\"封面: {d['pic']}\")
print(f\"播放: {d['stat']['view']}\")
print(f\"日期: {d['pubdate']}\")
print(f\"时长: {d['duration']}秒\")
print(f\"分P数: {d['videos']}\")
for i,p in enumerate(d['pages']):
    print(f\"  P{i+1}: {p['part']} (cid={p['cid']}, {p['duration']}秒)\")
"
```

若输出以 `BILIBILI_API_ERROR` 开头 → 解析错误信息，告知大哥具体原因，流程终止。

**YouTube 专用：**
```bash
yt-dlp --print "%(title)s|%(uploader)s|%(view_count)s|%(upload_date)s|%(duration)s|%(thumbnail)s" "<URL>"
```

若 yt-dlp 返回非0或 stderr 包含 `age restricted` → 特殊处理：告知大哥该视频有年龄限制，需手动确认。

### 3.1 超短视频判定

获取时长后判断：

| 时长 | 策略 |
|------|------|
| < 60秒 | 超短视频 — 只归档元信息+封面，标注「短视频，未提炼」，结束流程 |
| 60秒 - 3分钟 | 短视频 — 走完整流程，但提炼精简（每节1-2个要点即可） |
| > 3分钟 | 正常流程 |

超短视频归档格式（简化版）：
```markdown
# <视频标题>

> 🏷️ 尘墨·视频知识采集 | 归档日期：YYYY-MM-DD | 🎬 短视频

![封面](<相对路径>)

## 📋 元信息
| 项目 | 内容 |
|------|------|
| 平台 | <B站/YouTube> |
| UP主 | <名称> |
| 时长 | <秒数>秒 |
| 链接 | <URL> |

> ⚡ 超短视频（<60秒），仅作索引归档，未做知识提炼。

---
*本文档由 尘墨·视频知识采集 Skill 自动生成*
```

直接跳到第十步生成文档。

---

## 第四步：下载封面图并归入归档目录

### 4.1 下载封面

**B站封面：** 第三步已获取 `d['pic']` 字段，直接下载。
```bash
curl -L -o "/tmp/cm_cover_$$.jpg" "<封面URL从第三步获取>"
```

**YouTube封面：**
```bash
yt-dlp --get-thumbnail "<URL>" | xargs curl -L -o "/tmp/cm_cover_$$.jpg"
```

### 4.2 将封面复制到归档目录

在确定归档路径后（第十步），将封面复制到归档文档旁：
```bash
COVER_DIR="knowledge/学习资源/<分类>/assets"
mkdir -p "$COVER_DIR"
cp "/tmp/cm_cover_$$.jpg" "$COVER_DIR/尘墨-<sanitized标题>-cover.jpg"
```

文档中引用相对路径：
```markdown
![封面](assets/尘墨-<sanitized标题>-cover.jpg)
```

---

## 第五步：检测视频语言

在转录前，先判断视频实际语言，避免用错模型：

**B站字幕语言检测（优先）：**
```bash
curl -s "https://api.bilibili.com/x/player/v2?bvid=<BV号>&cid=<cid>" | python3 -c "
import json,sys
resp = json.load(sys.stdin)
if resp.get('code') != 0:
    print('NO_SUBTITLE_DATA')
    sys.exit(0)
d = resp['data']
sub = d.get('subtitle', {}).get('subtitles', [])
if sub:
    for s in sub:
        print(f\"{s['lan']}: {s['lan_doc']}\")
else:
    print('NO_SUBTITLE')
"
```

**YouTube 语言检测：**
```bash
yt-dlp --print "%(language)s" "<URL>"
```

**语言→Whisper参数映射：**
| 检测到的语言 | whisper --language 参数 |
|-------------|----------------------|
| 中文/zh/zho/chi | `Chinese` |
| 英语/en | `English` |
| 日语/ja/jpn | `Japanese` |
| 韩语/ko/kor | `Korean` |
| 其他 | 先尝试 `auto` 检测，失败则用 `Chinese` |

⚠️ **外语视频归档策略：** 转录结果保留原文 + 追加中文翻译摘要，不全文翻译（保留原始精度）。

---

## 第六步：获取字幕/文本

### 6.1 检查是否有内置字幕

**B站（含 API 错误处理）：**
```bash
curl -s "https://api.bilibili.com/x/player/v2?bvid=<BV号>&cid=<cid>" | python3 -c "
import json,sys
resp = json.load(sys.stdin)
if resp.get('code') != 0:
    print('NO_SUBTITLE')
    sys.exit(0)
d = resp['data']
sub = d.get('subtitle', {}).get('subtitles', [])
if sub:
    print('HAS_SUBTITLE')
    for s in sub:
        print(f\"  LAN={s['lan']} URL={s['subtitle_url']}\")
else:
    print('NO_SUBTITLE')
"
```

**YouTube：**
```bash
yt-dlp --list-subs "<URL>" 2>&1 | head -20
```

### 6.2 下载字幕文本（兼容多种JSON结构）✅ P1-28

**B站字幕下载与解析：**
B站字幕JSON结构可能有变体（`body` / `subtitles` / `content`），兼容处理：

```bash
# 下载字幕JSON
curl -L -H "Referer: https://www.bilibili.com" -o "/tmp/cm_subtitle_$$.json" "<字幕URL>"

# 兼容多种JSON结构解析
python3 -c "
import json

with open('/tmp/cm_subtitle_$$.json') as f:
    data = json.load(f)

# 尝试多种可能的key
items = None
if isinstance(data, dict):
    items = data.get('body') or data.get('subtitles') or data.get('content')
elif isinstance(data, list):
    items = data

if not items:
    print('SUBTITLE_PARSE_ERROR: 未知的字幕格式')
    sys.exit(1)

for item in items:
    # 兼容不同的时间戳字段名
    start = item.get('from') or item.get('start') or item.get('begin') or 0
    text = item.get('content') or item.get('text') or item.get('value') or ''
    mm, ss = divmod(int(start), 60)
    hh, mm = divmod(mm, 60)
    ts = f'[{int(hh):02d}:{int(mm):02d}:{int(ss):02d}]'
    print(f'{ts} {text}')
" > "/tmp/cm_subtitle_text_$$.txt"
```

若输出 `SUBTITLE_PARSE_ERROR` → 错误日志记录，回退到 Whisper 转录方案。

**YouTube 字幕下载：**
```bash
yt-dlp --write-subs --sub-langs "zh-Hans,en" --skip-download --convert-subs txt -o "/tmp/cm_subtitle_$$" "<URL>"
```

- 有内置字幕且解析成功 → 跳过 Whisper 转录
- 无内置字幕或解析失败 → 进入第七步

---

## 第七步：下载音频并转录（无字幕时）

### 7.1 创建隔离临时目录

```bash
CM_TMP="/tmp/cm_video_$$"
mkdir -p "$CM_TMP"
```

### 7.2 B站音频获取（codec 兼容版 + 备用URL重试）

⚠️ **B站CDN音频链接有时效性，获取后必须立即下载！**

```bash
# 1. 获取 dash 音频流所有可用URL（含备用）
ALL_URLS=$(curl -s "https://api.bilibili.com/x/player/playurl?bvid=<BV号>&cid=<cid>&fnval=4048&fourk=1" | \
  python3 -c "
import json,sys
resp = json.load(sys.stdin)
if resp.get('code') != 0:
    print(f'BILIBILI_PLAYURL_ERROR|{resp.get(\"code\")}|{resp.get(\"message\",\"\")}')
    sys.exit(1)
d = resp['data']['dash']['audio']
urls = []
for a in d:
    url = a.get('baseUrl') or a.get('base_url') or a.get('url', '')
    if url:
        codec_info = a.get('codecs', 'unknown')
        backup_urls = a.get('backupUrl') or a.get('backup_url') or []
        if not isinstance(backup_urls, list):
            backup_urls = [backup_urls]
        urls.append({'url': url, 'codec': codec_info, 'backups': backup_urls})
for u in urls:
    print(json.dumps(u, ensure_ascii=False))
" 2>/dev/null)

# 2. 按优先级尝试下载：mp4a优先 > 第一个可用 > 备用URL
echo "$ALL_URLS" | python3 -c "
import json,sys
urls = [json.loads(line) for line in sys.stdin if line.strip()]
if not urls:
    sys.exit(1)
# mp4a优先
urls.sort(key=lambda u: 0 if 'mp4a' in u['codec'] else 1)
for u in urls:
    print(u['url'])
    for bu in u.get('backups', []):
        print(bu)
" > "$CM_TMP/audio_urls.txt"

# 3. 逐个尝试下载，直到成功
while IFS= read -r url; do
    [ -z "$url" ] && continue
    echo "尝试下载: ${url:0:80}..."
    if curl -L -H "Referer: https://www.bilibili.com" --connect-timeout 10 --max-time 120 -o "$CM_TMP/audio.m4s" "$url" 2>/dev/null; then
        FILE_SIZE=$(stat -f%z "$CM_TMP/audio.m4s" 2>/dev/null || echo 0)
        if [ "$FILE_SIZE" -gt 1024 ]; then
            echo "✅ 下载成功 (${FILE_SIZE} bytes)"
            break
        fi
    fi
    echo "⚠️ 该URL下载失败，尝试下一个..."
done < "$CM_TMP/audio_urls.txt"

# 4. 检查是否下载成功
if [ ! -f "$CM_TMP/audio.m4s" ] || [ $(stat -f%z "$CM_TMP/audio.m4s" 2>/dev/null || echo 0) -le 1024 ]; then
    echo "[$(date +%H:%M:%S)] B站音频下载失败: 所有URL均不可用" >> "$CM_ERROR_LOG"
    exit 1
fi

# 5. 转 mp3
ffmpeg -y -i "$CM_TMP/audio.m4s" -acodec libmp3lame -ab 128k "$CM_TMP/audio.mp3" 2>/dev/null
```

### 7.3 YouTube / 通用（yt-dlp）

```bash
yt-dlp -x --audio-format mp3 --audio-quality 128k -o "$CM_TMP/audio.%(ext)s" "<URL>" 2>&1 | while IFS= read -r line; do
    echo "$line"
    if echo "$line" | grep -qi "age.restricted\|sign in\|login required\|unavailable"; then
        echo "[$(date +%H:%M:%S)] YouTube 下载受限: 可能涉及年龄限制或登录要求" >> "$CM_ERROR_LOG"
    fi
done
```

### 7.4 Whisper 转录（带进度 + 超时检测）

```bash
whisper "$CM_TMP/audio.mp3" --model base --language <第五步检测到的语言> --output_dir "$CM_TMP" --output_format txt --verbose False 2>&1 | \
  while IFS= read -r line; do echo "[尘墨转录] $line"; done
```

⚠️ **转录失败处理：**
- whisper 命令返回非0 → 追加到错误日志，告知大哥
- 音频文件损坏 → 错误日志记录，提示检查原始视频
- 转录结果为空 → 归档时标注「无语音内容」
- 转录超过30分钟无输出 → 认为卡死，kill 进程，错误日志记录

---

## 第八步：AI 提炼知识

基于字幕/转录文本，逐段提炼核心知识点。**不靠标题猜测，必须基于实际内容。**

### 提炼结构：

提炼内容最前面加入 **📌 TL;DR 摘要**，让大哥一眼判断这视频值不值得细看：

```markdown
## 📌 TL;DR 摘要
> **一句话总结：** <用一句话说清这个视频讲什么、听完能收获什么>
> **🔑 三个核心收获：**
> 1. <最核心的可落地收获>
> 2. <第二核心的可落地收获>
> 3. <第三核心的可落地收获>

---

## 第N章/节：<章节名>
> 🕐 时间范围：HH:MM:SS - HH:MM:SS
### 📌 核心观点
### 🔑 关键知识点
### 💡 可落地的方法/技巧
### ⚠️ 注意事项/坑点
### 🔖 关键时刻
- [HH:MM:SS] 关键操作/金句描述
```

### 文件名安全处理

视频标题在用作文件名之前必须 sanitize，去掉非法字符：

```python
import re
def sanitize_filename(title):
    """移除文件名非法字符"""
    title = re.sub(r'[<>:"/\\|?*]', '-', title)
    title = title.strip('. ')
    if len(title) > 120:
        title = title[:117] + '...'
    return title
```

用法：`归档文件名 = "尘墨-" + sanitize_filename(视频标题) + ".md"`

### 提炼质量标准

| 维度 | 标准 |
|------|------|
| TL;DR | 一句话必须有信息增量，不能说"讲了很多关于xx的内容"❌ |
| 章节划分 | 按内容自然转折切分，每节3-8个要点，不宜过碎或过长 |
| 核心观点 | 一句话概括本节主旨，不能是废话 |
| 关键知识点 | 具体、可验证、有操作性的知识点，避免空泛概括 |
| 可落地方法 | 给出实际能用的命令/配置/步骤，不写"可以通过某种方式" |
| 注意事项 | 作者踩过的坑 + 推断的潜在风险，有具体上下文 |
| 关键时刻 | 基于字幕时间戳，标注重要操作/金句/转折点在几分几秒 |

**完整示例（好的提炼）：**
```markdown
## 📌 TL;DR 摘要
> **一句话总结：** 用 Scrapling 的 StealthySession 可以绕过 Cloudflare Turnstile，但必须用非 headless 模式才稳定。
> **🔑 三个核心收获：**
> 1. solve_cloudflare=True 仅对 Turnstile 有效，hCaptcha 需用别的方案
> 2. headless=False + network_idle=True 是稳定绕过的关键组合
> 3. 多实例并发会触发 IP 风控，必须错开或换代理

---

## 第3节：Cloudflare Turnstile 绕过实战
> 🕐 时间范围：12:30 - 18:45
### 📌 核心观点
StealthyFetcher 的 solve_cloudflare 不是万能的，需要配合正确的浏览器指纹和等待策略。
### 🔑 关键知识点
- solve_cloudflare=True 仅对 Turnstile 有效，对 hCaptcha 无效
- 必须设置 headless=False 才能过部分增强型检测
- 等待 network_idle 比固定 sleep 3s 更可靠
### 💡 可落地的方法/技巧
```python
with StealthySession(headless=False, solve_cloudflare=True) as s:
    page = s.fetch(url, network_idle=True, wait=2000)
```
### ⚠️ 注意事项/坑点
- 频繁请求同一域名会触发 IP 风控，建议换代理
- 不要同时开多个 solve_cloudflare 实例
### 🔖 关键时刻
- [14:02] 演示 Turnstile 实际绕过过程
- [16:30] 作者踩坑：headless=True 导致无限转圈
- [17:50] 给出最终可行方案
```

---

## 第九步：自动分类

根据视频内容和标题自动判断归档分类：

| 特征 | 分类目录 |
|------|---------|
| Agent/OpenClaw/Skill/Cursor/AI编程工具 | `AI工具与Agent/` |
| ComfyUI/Stable Diffusion/Midjourney/AI绘画 | `AI图片生成/` |
| Sora/Runway/AI视频/音频生成 | `AI视频音频/` |
| 提示词/AI写作/ChatGPT技巧 | `AI写作创作/` |
| 纯编程/架构/算法/后端/前端 | `编程开发/` |
| UI/UX/Figma/产品设计 | `产品设计/` |
| 读书/深度长文/哲学 | `读书感悟/` |
| 效率工具/自动化/AppleScript/快捷键 | `基础技能/` |
| 无法判断 | 归入根目录 `学习资源/`，注标「待分类」 |

判断依据排序：视频标题 > 转录内容关键词 > UP主频道定位。

---

## 第十步：生成结构化归档文档 ✅ P0-24

**先确保归档目录存在：**
```bash
mkdir -p "knowledge/学习资源/<第九步分类>/"
mkdir -p "knowledge/学习资源/<第九步分类>/assets/"
```

**文件路径：** `knowledge/学习资源/<第九步分类>/尘墨-<sanitized标题>.md`

**文档模板：**
```markdown
# <视频标题>

> 🏷️ 尘墨·视频知识采集 | 归档日期：YYYY-MM-DD | 语言：<第五步检测结果>

![封面](assets/尘墨-<sanitized标题>-cover.jpg)

## 📋 元信息

| 项目 | 内容 |
|------|------|
| 平台 | B站/YouTube/... |
| UP主/作者 | <名称> |
| 视频链接 | <URL> |
| 发布日期 | YYYY-MM-DD |
| 时长 | <HH:MM:SS> |
| 播放量 | <数字> |
| 转录方式 | 内置字幕 / Whisper base / 无语音 |

## 🎬 视频嵌入

<iframe src="<B站/YouTube嵌入链接>" sandbox="allow-scripts allow-same-origin allow-popups" allowfullscreen="true" width="100%" height="500"></iframe>
🔗 [在新窗口打开观看](<原始链接>)

## 📝 知识提炼

<第八步的提炼内容>

---

*本文档由 尘墨·视频知识采集 Skill 自动生成*
```

---

## 第十一步：清理临时文件 + 计算耗时

```bash
# 计算总耗时
CM_END_TIME=$(date +%s)
CM_ELAPSED=$((CM_END_TIME - CM_START_TIME))
CM_MIN=$((CM_ELAPSED / 60))
CM_SEC=$((CM_ELAPSED % 60))
if [ "$CM_MIN" -gt 0 ]; then
    CM_TIME_STR="${CM_MIN}分${CM_SEC}秒"
else
    CM_TIME_STR="${CM_SEC}秒"
fi

# 如果错误日志中有失败记录 → 保留日志文件路径，告知大哥
if [ -s "$CM_ERROR_LOG" ] && grep -q "失败\|错误\|ERROR" "$CM_ERROR_LOG"; then
    echo "⚠️ 尘墨：处理过程中有错误，日志保留在 $CM_ERROR_LOG"
else
    rm -f "$CM_ERROR_LOG"
fi

# 清理其他临时文件
rm -rf "$CM_TMP" "/tmp/cm_cover_$$.jpg" "/tmp/cm_subtitle_$$.json" "/tmp/cm_subtitle_text_$$.txt" "/tmp/cm_audio_urls.txt" 2>/dev/null
```

---

## 特殊场景处理

### 多P/多集系列视频（含子Agent task 模板）✅ P2-27

- 每P独立转录和提炼
- 总文档包含目录表格（编号/标题/时长/提取状态）
- 可并行处理（每批3-4P用子Agent，每批独立 PID 隔离）

**子Agent task 模板（用 `sessions_spawn` 调度）：**
```
【尘墨子任务】处理视频第 <N>P

视频BVID: <BV号>
分P CID: <cid>
分P标题: <part标题>
分P时长: <时长>秒

请完整执行 尘墨·视频知识采集 Skill 中的第五步到第八步：
1. 检测本P语言
2. 检查并获取字幕/文本（如无字幕则下载音频+Whisper转录）
3. AI提炼知识点（含TL;DR摘要）
4. 将提炼结果写入临时文件 /tmp/cm_p<N>_提炼_$$.md

完成后回复「第<N>P 提炼完成」。
```

**合并排序规则：** 按P编号升序排列（P1→P2→P3...），保持视频原始顺序。

**失败隔离：** 某P失败不中断整体系列，跳过继续处理，最后汇总：
- ✅ 成功归档的P
- ⚠️ 失败的P（标注失败原因）
- 总文档标记为「部分完成」

**合并流程：** 所有子Agent完成后，收集 `/tmp/cm_p*_提炼_$$.md`，按P编号顺序合并到同一文档。

### 下载失败回退

1. 首选方案失败 → 尝试 yt-dlp + cookie（`--cookies-from-browser chrome`）
2. yt-dlp 也失败 → 尝试 scrapling stealthy-fetch 获取页面后提取视频URL
3. 全部失败 → 错误日志记录，告知大哥具体失败原因

### 已有本地视频文件

- 跳过下载步骤，直接从第七步（音频提取）开始
- 大哥需提供视频文件路径和基本信息（标题/来源）

### 增量更新（UP主追加新P）

当第一步检测到已有同名归档文档时：

1. 读取已有归档，获取已归档的P列表
2. 对比最新的分P信息，找出新增的P
3. 只处理新P（下载→转录→提炼）
4. 在已有文档末尾追加：
```markdown
---

## 📦 增量更新 YYYY-MM-DD

<新增P的提炼内容>
```
5. 更新元信息中的时长、播放量等数据
6. 如果无新P → 告知大哥：「该视频已完整归档，无需更新」

---

## 尘墨品牌规范

- 归档文件名前缀：「尘墨-」
- 临时文件前缀：`cm_`（尘墨缩写）
- 文档底部标注：「本文档由 尘墨·视频知识采集 Skill 自动生成」
- 转录/提炼完成后，向大哥汇报格式：

```
🎬 尘墨采集完成
📺 <视频标题>
👤 <UP主/作者>
🌐 语言：<中/英/日等> | 转录方式：<字幕/Whisper>
⏱ 视频时长 <HH:MM:SS> | 转录字数 <N> | 处理耗时 <X分Y秒>
📄 归档位置：knowledge/学习资源/<分类>/尘墨-<sanitized标题>.md
🖼 封面已保存
<如有错误日志>⚠️ 处理中有警告：日志见 /tmp/cm_error_XXXXX.log
```

---

## 与现有工具链的衔接

- **Whisper 转录**：沿用 MEMORY.md 中已验证的 `whisper base` 配置，语言参数从第五步动态获取
- **B站 API**：沿用已建立的 B站 player API 调用模式，所有API调用均检查 `code` 字段，CDN 音频 URL 有时效性需立即下载
- **归档规范**：遵循 `archive-review` skill 的文档模板和分类体系
- **并行处理**：多集系列用 `sessions_spawn` 调度子Agent分批并行，子Agent使用上述 task 模板，每批独立 PID 临时目录
- **反爬兜底**：下载失败时可调用 `scrapling-official` 的 stealthy-fetch 获取受限页面

---

## 修订记录

| 日期 | 修订内容 | 累计优化 |
|------|---------|---------|
| 2026-06-29 v1 | 初始创建：基于老章 z-skills 精华 | 基础 |
| 2026-06-29 v2 | 6项P0+4项P1+4项P2 | 14项 |
| 2026-06-29 v3 | 2项P0+3项P1+3项P2 | 22项 |
| 2026-06-29 v4 | P0-23 B站API错误码处理 / P0-24 归档目录mkdrip / P1-25 处理耗时统计 / P1-26 磁盘空间预检 / P1-28 字幕JSON兼容解析 / P2-27 多P子Agent task模板 | **28项** |
