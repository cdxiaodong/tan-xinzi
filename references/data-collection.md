# 素材抓取手册 · 如何持续扩充本 skill

> 本 skill 的方法论全部提炼自博主「布布糕」的公开视频。博主共 **334 条作品**，本 skill 已覆盖 37 条核心视频。本文档记录**经过验证的完整抓取→转录→整合工作流**，供后续持续扩充。
> 所有命令均已在 macOS（Apple Silicon）+ Codex 环境实测通过。

---

## 一、博主档案

| 维度 | 信息 |
|------|------|
| 昵称 | 布布糕 |
| sec_uid | `MS4wLjABAAAAHjFX2fm2paXE3VypOvUav1EpKL5HfkRn7NguTLlnrAI` |
| unique_id | `34166593513` |
| 作品数 | 334 条 |
| 粉丝 | 157 万 |
| 总赞 | 1906 万 |
| IP 属地 | 上海 |
| 个性签名 | 全球领先外资顶猎 亚太区 Top biller |

**主页链接格式**：
- Web：`https://www.douyin.com/user/{sec_uid}`
- 单条视频：`https://www.douyin.com/video/{aweme_id}`

---

## 二、工作流总览（五步）

```
① 获取视频列表 → ② 筛选相关视频 → ③ 下载无水印 → ④ 抽音频转写 → ⑤ 整合进 skill 模块
```

### 第 1 步：获取视频列表

**重要：抖音作品流接口要求登录态（msToken），未登录会话即使签名正确也会被风控判空。** 已穷尽以下未登录路径，**全部失败**：
- Web `/aweme/v1/web/aweme/post/`（含合法 a_bogus 签名）→ 200 + 空 body
- 移动端 `iesdouyin.com/share/user/{sec_uid}` → JS 反爬挑战页
- 旧版 `iesdouyin.com/web/api/v2/aweme/post/` → 空 body
- DOM 抓取主页 → 本人作品网格不加载
- 搜索引擎 → 未有效索引

**可行的获取方式（按推荐顺序）**：
1. **手动收集 aweme_id**：在抖音 App / 网页浏览博主主页，把相关视频的链接或 aweme_id 记录下来（最可靠）。
2. **单条解析**：任意一条视频链接都能用 MCP 工具解析（见第 3 步），不受列表风控限制。
3. **登录态抓取**（进阶）：在已登录抖音的浏览器里调 `/aweme/v1/web/aweme/post/`，需带真实 msToken。本环境未自动化此路径。

> 本 skill 现有的 332 条视频清单（`work/all_videos.json`）是早期通过受限渠道获取的，可作为筛选底库。

### 第 2 步：筛选相关视频

本 skill 聚焦以下主题，按相关度筛选 aweme_id：
- **谈薪**：期望薪资、报价、反价、总包、涨幅、title 和钱
- **面试**：自我介绍、motivation、压力面、被拒自救、追反馈、跨行业、后发制人
- **HR/猎头**：HR 友军、猎头天坑、找猎头、荒诞 JD
- **简历**：制作、红线、可改项
- **离职**：提离职、离职理由、试用期、GAP、竞业
- **裁员**：风声、竞品内推、礼包
- **职业规划**：升职、向上管理、转行时机

筛选结果记录到分析报告（见 `outputs/全部视频筛选_谈薪HR面试分级.md` 范式）。

### 第 3 步：下载无水印视频

用 MCP 工具 `mcp__douyin__get_douyin_download_link` 获取直链，再 curl 下载。

**单条获取下载链**（Codex MCP 调用）：
```
mcp__douyin__get_douyin_download_link(share_link="https://www.douyin.com/video/{aweme_id}")
→ 返回 JSON：{ download_url, title, video_id }
```

**批量下载（shell）**：
```bash
mkdir -p work/vN_batchN/{audio,whisper_out} && cd work/vN_batchN
UA="Mozilla/5.0 (iPhone; CPU iPhone OS 16_6 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/16.6 Mobile/15E148 Safari/604.1"
# 把 get_douyin_download_link 返回的 url 填进来
declare -a V=(
"编号_简短标题|https://aweme.snssdk.com/aweme/v1/play/?video_id=xxxxx&ratio=720p&line=0"
# ... 更多
)
for entry in "${V[@]}"; do
  name="${entry%%|*}"; url="${entry#*|}"
  nohup curl -sL -A "$UA" -o "${name}.mp4" "$url" > /dev/null 2>&1 &
done
wait
for entry in "${V[@]}"; do
  name="${entry%%|*}"; printf "%-24s %s bytes\n" "$name" "$(stat -f%z "${name}.mp4" 2>/dev/null||echo 0)"
done
```

**已知坑**：
- **限流**：同一时间批量请求 `get_douyin_download_link` 偶发"从HTML中解析视频信息失败"。**解法：隔几分钟重试**，通常恢复（已验证：失败的视频过几小时再试就成功）。
- **视频被删除/替换**：aweme_id 可能被博主删除后指向无关视频。**解法：下载后先核实内容**（转写后看开头是否文不对题）。案例：竞业协议（7386172668590050596）原 ID 后来指向一个"狗被杀"社会评论视频。
- **超短视频**：13 秒以内的视频转写只有 3 行，无信息量，直接排除。

### 第 4 步：抽取音频 + Whisper 转写

**抽音频（ffmpeg，16kHz 单声道 PCM，whisper 最佳输入格式）**：
```bash
for f in *.mp4; do
  base="${f%.mp4}"
  nohup ffmpeg -y -i "$f" -vn -acodec pcm_s16le -ar 16000 -ac 1 "audio/${base}.wav" > /dev/null 2>&1 &
done
wait
# 核对时长
for f in audio/*.wav; do
  dur=$(ffprobe -v error -show_entries format=duration -of csv=p=0 "$f" 2>/dev/null|cut -d. -f1)
  printf "%-28s %ss\n" "$(basename "$f")" "$dur"
done
```

**Whisper 转写**：
```bash
WHISPER=/Users/cdxd/Desktop/develop/ai-audit/securevibes/env/bin/whisper
for f in audio/*.wav; do
  base=$(basename "$f" .wav)
  nohup "$WHISPER" "$f" --model small --language zh \
    --output_format txt --output_dir whisper_out --task transcribe \
    > "whisper_out/${base}.log" 2>&1 &
done
wait   # 关键：必须 wait，否则后台进程会被 SIGHUP
for f in whisper_out/*.txt; do
  [ -s "$f" ] && echo "$(basename "$f"): ✅ $(wc -m < "$f")字" || echo "$(basename "$f"): ❌"
done
```

**模型选择（重要）**：
- **用 `small`，不要用 `medium`**。small 单条约 2–4 分钟，medium 要 5–15 分钟，中文识别质量差异不大。
- **CPU 模式会警告 "FP16 not supported on CPU; using FP32"**，正常现象。
- **并行度控制**：7 并行在 CPU 上会严重抢占（load 18+），一批 7 条要 15–25 分钟。建议**控制在 3–4 并行**更稳，或分批跑。
- **whisper 跑完才写 txt**：中途看不到输出是正常的，看 `*.log` 里的 SRT 进度判断是否在推进。
- **DASHSCOPE_API_KEY 未设置**：云 ASR（`extract_douyin_text`）不可用，必须用本地 whisper。

### 第 5 步：整合进 skill 模块

转写完成后，按内容主题归入对应模块文件：

| 视频主题 | 归入文件 |
|----------|----------|
| 谈薪/报价/反价/总包 | `salary-scripts.md` / `total-comp.md` |
| counter offer | `counter-offer.md` |
| 面试（自我介绍/motivation/压力面/被拒/跨行业） | `interview-skills.md` |
| HR/猎头 | `hr-playbook.md` / `recruiter-tactics.md` |
| 简历 | `resume-building.md` |
| 离职（提离职/理由/试用期/GAP） | `resignation-handoff.md` |
| 离职时机 | `resignation-timing.md` |
| 裁员 | `layoff-rebound.md` |

**整合规范**：
1. **提炼而非堆砌**：把口语化转写提炼成结构化要点（表格/清单/话术模板），保留博主原话金句（标注"布布糕原话"）。
2. **每段加引用视频**：章节末尾标注 `《标题》（aweme_id）`。
3. **更新 SKILL.md**：如果是新主题，在路由表加新行；更新 description 的模块描述。
4. **更新 README.md**：视频覆盖数 +N，模块地图补对应行，文件结构如有新文件补充。
5. **写分析报告**：在 outputs/ 下建 `视频分析_第N批_主题.md`，记录本批视频清单、字数、归入模块、内容要点、失败视频。
6. **提交规范**：`git commit -m "扩充第N批N条主题视频内容"`，逐文件说明变更。

---

## 三、目录结构约定

```
工作区/
├── work/
│   ├── all_videos.json              # 332 条视频清单（筛选底库）
│   └── vN_batchN/                   # 每批一个目录
│       ├── 编号_标题.mp4            # 下载的视频
│       ├── audio/编号_标题.wav      # 16kHz 单声道音频
│       └── whisper_out/
│           ├── 编号_标题.txt        # 转写文本
│           └── 编号_标题.log        # whisper 日志（含SRT进度）
└── outputs/
    ├── 全部视频筛选_谈薪HR面试分级.md   # 筛选结果
    └── 视频分析_第N批_主题.md           # 每批分析报告
```

---

## 四、环境依赖

| 工具 | 路径 / 说明 |
|------|------------|
| whisper | `/Users/cdxd/Desktop/develop/ai-audit/securevibes/env/bin/whisper`（Python 3.14 venv） |
| ffmpeg | 系统自带或 brew install |
| ffprobe | 同 ffmpeg |
| 抖音 MCP | Codex 内置 `mcp__douyin__get_douyin_download_link` / `parse_douyin_video_info` |
| gh | GitHub CLI（已认证，scope: gist/read:org/repo/workflow） |

> **whisper 模型文件**首次运行会自动下载到 `~/.cache/whisper/`，small 模型约 461MB。
> 如需在其他机器复现，装 `pip install openai-whisper` 即可，模型自动下载。

---

## 五、快速复制的单条流程（最常用）

抓一条新视频的完整命令序列：

```bash
# 1. 拿下载链（MCP）
#    mcp__douyin__get_douyin_download_link(share_link="https://www.douyin.com/video/{id}")

# 2. 下载 + 抽音频 + 转写（把 URL 填进去）
cd /Users/cdxd/Documents/Codex/2026-08-05/1-7-94-https-v-douyin/work
mkdir -p tmp && cd tmp
UA="Mozilla/5.0 (iPhone; CPU iPhone OS 16_6 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/16.6 Mobile/15E148 Safari/604.1"
curl -sL -A "$UA" -o v.mp4 "DOWNLOAD_URL"
ffmpeg -y -i v.mp4 -vn -acodec pcm_s16le -ar 16000 -ac 1 a.wav -loglevel error
/Users/cdxd/Desktop/develop/ai-audit/securevibes/env/bin/whisper a.wav \
  --model small --language zh --output_format txt --output_dir . --task transcribe
cat a.txt   # 转写结果
```

转写读完 → 提炼要点 → 追加到对应模块 → 更新 SKILL.md/README.md → `git add -A && git commit && git push`。
