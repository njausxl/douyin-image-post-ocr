# douyin-image-post-ocr

> ### 🔀 这是 fork 修改版的配套技能包仓库
>
> | 仓库 | 地址 |
> | --- | --- |
> | **完整 fork**（含上游源码 + 我的修改，可直接 diff） | <https://github.com/njausxl/douyin-favorites-to-knowledge> |
> | **上游原仓库** | [tars1230/douyin-favorites-to-knowledge](https://github.com/tars1230/douyin-favorites-to-knowledge) |
> | 上游 Gitee 镜像 | [gitee.com/tars123](https://gitee.com/tars123/douyin-favorites-to-knowledge) |
>
> 本仓库是把修改内容**单独抽出来**的轻量技能包：**不想改动上游代码、只想给已有安装加上 OCR / 长文能力**的，直接装这个即可。
> 想要完整可运行的修改版代码，请用上面的**完整 fork** 仓库。

> 给 [`douyin-favorites-to-knowledge`](https://github.com/tars1230/douyin-favorites-to-knowledge) 补上两条被忽略的正文通道：
> **图文帖的图片文字（OCR）** 与 **抖音长文的文章正文**。

上游工具把抖音收藏同步成 Obsidian 笔记时，只有**语音识别（ASR）一条通道**。但收藏里的内容其实分三类，其中两类完全抓不到正文 —— 笔记里只剩标题，还混着一堆背景音乐跑出来的幻觉噪声。

本仓库提供完整的补丁、技能脚本和踩坑记录，让这两类内容也能被自动读进知识库。

| 项目 | 说明 |
|---|---|
| 上游项目 | [tars1230/douyin-favorites-to-knowledge](https://github.com/tars1230/douyin-favorites-to-knowledge)（本仓库为其 fork 修改版的配套技能包） |
| 补丁改动 | 4 个文件，+437 / −9 行 |
| 技能版本 | v1.4.0（10 个脚本 + 统一路径解析） |
| 实测规模 | 932 条收藏 → 154 条图文帖 + 38 条长文 |
| 补全结果 | 知识库 306 → **366** 条笔记，提取约 **24 万字** |
| 视觉模型 | `Qwen/Qwen3-VL-30B-A3B-Instruct`（SiliconFlow） |

---

## 一、它解决什么问题

抖音收藏里混着三类内容，按 `aweme_type` 区分：

| 类型 | aweme_type | 正文在哪 | 上游的处理 |
|---|---|---|---|
| 普通视频 | 0 | 音轨里 | ✅ 语音识别，正常 |
| 图文帖 | 68 | **图片里** | ❌ 完全丢失 |
| 长文（文章） | 163 | `article_info` 里 | ❌ 完全丢失 |

### 坑一：图文帖的「视频地址」是背景音乐

图文帖没有真正的视频流，抖音把 `video.play_addr` 填成了**背景音乐 MP3**。上游不知情，照常把这条 MP3 送去 ASR，于是产出：

```
🎼你生看。
🎼嗯。。🎼的都荡。The number you dialed is not in service.
```

状态却标成 `success`。**结果是正文一个字没抓，反而塞进一堆噪声。**

### 坑二：长文正文被截断在 499 字

长文正文在 `article_info.article_content.markdown`，但**收藏列表接口返回的这份是截断在 499 字的预览**。完整正文要走 `aweme/detail` 接口。

---

## 二、快速开始

### 0. 前置条件

- Python 3.11+
- 上游项目 `douyin-favorites-to-knowledge`
- 一个 SiliconFlow API Key（只有图文帖 OCR 会花钱；长文通道零成本）
- 首次采集需扫码登录抖音，之后复用本地浏览器 profile

### 1. 装上游 + 打补丁

```bash
# 方式一（推荐）：直接 clone 完整 fork，已含全部修改，无需打补丁
git clone https://github.com/njausxl/douyin-favorites-to-knowledge.git

# 方式二：clone 上游原版，自己打补丁
git clone https://github.com/tars1230/douyin-favorites-to-knowledge.git
git apply <本仓库>/patches/0001-ocr-and-article-support.patch

cd douyin-favorites-to-knowledge
python -m venv .venv
.venv/Scripts/pip install -e .        # POSIX: .venv/bin/pip
```

补丁改动的 4 个文件：

| 文件 | 改动 |
|---|---|
| `browser_collector.py` | 采集时多取 `item.images`；识别长文 `article_id` 并调 `aweme/detail` 取全文 |
| `siliconflow.py` | 新增图片 OCR 通道；遇长文直接短路，不做 ASR |
| `workflow.py` | 新增 `## 图片文字 (OCR)` 与 `## 文章正文` 两段渲染 |
| `cli.py` | 已入库的图文帖／长文从本轮剔除，跳过重复处理 |

> ⚠️ **`src/` 与 `.venv/Lib/site-packages/` 下各有一份同名文件，两份都要打补丁**，否则重装依赖后改动会丢。

### 2. 装技能包

把 `skill/` 整个目录拷进技能目录：

```bash
cp -r skill ~/.workbuddy/skills/douyin-image-post-ocr
```

### 3. 配路径与环境变量

脚本**不含任何硬编码机器路径**，全部由 `scripts/_paths.py` 解析：环境变量优先，未设置时自动探测默认布局。

先自检（打印 6 个路径，标注 `OK` / `MISS`）：

```bash
<venv>/python <skills目录>/douyin-image-post-ocr/scripts/_paths.py
```

| 环境变量 | 含义 | 默认值 |
|---|---|---|
| `DOUYIN_OCR_ROOT` | 工作根目录 | `~/OneDrive/douyin`，退回 `~/douyin` |
| `DOUYIN_UPSTREAM` | 上游项目目录 | `$DOUYIN_OCR_ROOT/douyin-favorites-to-knowledge` |
| `DOUYIN_SITE_PACKAGES` | 打过补丁的 site-packages | `$DOUYIN_UPSTREAM/.venv/**/site-packages` |
| `DOUYIN_OCR_WORK` | 中间产物目录 | `$DOUYIN_OCR_ROOT/ocr_test` |
| `DOUYIN_OCR_KB` | Obsidian 知识库目录 | `~/OneDrive/Apps/Obsidian library/抖音知识库` |
| `DOUYIN_OCR_LEDGER` | 账本 sqlite3 文件 | `%APPDATA%/douyin-favorites-to-knowledge/state/ledger.sqlite3` |

`MISS` 的项用对应环境变量覆盖即可，不用改脚本。

### 4. 运行时环境

```bash
export SILICONFLOW_API_KEY=你的密钥        # 只读环境变量，不落盘

# 代理没开时必须清掉，否则报 ECONNREFUSED 127.0.0.1:7890
unset HTTP_PROXY HTTPS_PROXY http_proxy https_proxy
```

---

## 三、四种用法

### 方式 A：日常同步（已内置，零手工）

打完补丁后，`sync` / `daily` / `scan` 三条命令**自动分流**：

- 有 `images` → 走视觉模型 OCR
- 有 `article` → 直接取正文，**不消耗 API 额度**
- 都没有 → 照旧走 ASR

开关：`DOUYIN_OCR_ENABLED=0` 关闭自动 OCR；`DOUYIN_OCR_MODEL` 换模型；`DOUYIN_OCR_CONCURRENCY` 调并发。

### 方式 B：存量图文帖批量补全

处理补丁上线前入库的历史笔记：

```bash
python scripts/collect_images.py                          # 1. 扩展采集
python scripts/ocr_images.py all \                        # 2. 批量 OCR（可断点续跑）
       Qwen/Qwen3-VL-30B-A3B-Instruct
python scripts/merge_ocr.py                               # 3. 预演
python scripts/merge_ocr.py --apply                       #    写入（自动整体备份）
```

### 方式 C：误筛图文帖回捞

筛选口径（如「去掉游戏娱乐」）很容易连坐误伤。把被误筛的知识类图文帖补进库，**直接复用主管线**：

```bash
python scripts/backfill_misfiltered.py plan   # 只打印名单，不发请求
python scripts/backfill_misfiltered.py run    # 执行，自动 OCR
python scripts/backfill_report.py             # 生成报告
```

### 方式 D：长文正文补全

```bash
python scripts/fetch_article_full.py          # 走 aweme/detail 取全文
python scripts/backfill_articles.py plan      # 预演
python scripts/backfill_articles.py apply     # 回填 + 入库（自动备份知识库与账本）
```

> 长文通道**零 API 成本** —— 取的是抖音自己的正文，不调模型。

---

## 四、关键设计：三个必须绕开的坑

1. **OCR / 文章正文不参与 `content_sha256`**。它们都在哈希算完之后才挂上去。否则换个模型、换个措辞重跑，就会被判定「条目已变更」而报错。
2. **已入库的图文帖必须从本轮剔除**。补丁上线前入库的图文帖，哈希是基于**背景音乐跑出来的噪声转录**算的；上线后跳过 ASR 就再也复现不出那个哈希，保留只会报 `promoted item changed`。
3. **已入库笔记只能合并回填，不能重推**。`promote` 有「untracked note conflict」保护：笔记已存在且内容不一致就直接拒绝。删账本重推行不通，必须把正文合并进已有笔记。

另外两个：

- **`assert_safe_value` 会拦正文**：`security.py` 拦截疑似密钥（`sk-…`、`gh?_…`）、`<think>` / `<analysis>`、U+FFFD。技术长文里可能有代码示例，**入库前先扫一遍**。
- **图片下载要带 Referer**：图片走抖音 CDN，必须带 `Referer: https://www.douyin.com/`，否则 403。

---

## 五、实测数据（2026-09-10）

### 图文帖 OCR

| 项目 | 结果 |
|---|---|
| 验证取样 | 6 条 / 145 张图 |
| 识别出文字 | 135 张（其余为纯风景实拍照，正常） |
| 产出字数 | 30,386 字 |
| 端到端耗时 | 6 分 57 秒（约 2.9 秒/张） |
| 分流正确率 | 6 条全部走图片通道，无一误送 ASR |

最大一条是 100 图的贴吧长帖：OCR 还原 **24,533 字**，连楼中楼评论都抽了出来。

误筛回捞批次：**49 条 / 252 张图**，25 分钟跑完，提取 73,844 字。

### 长文补全

| 项目 | 结果 |
|---|---|
| 长文总数 | 38 条 |
| 回填正文（已入库） | 24 条 |
| 新增入库 | 11 条 |
| 正文合计 | 136,850 字 |

### 模型选型

| 模型 | 表现 |
|---|---|
| `PaddlePaddle/PaddleOCR-VL-1.5` | ❌ 社交截图场景反复退化，输出一长串 `0000` 或 emoji 直到撞满上限 |
| `Qwen/Qwen3-VL-30B-A3B-Instruct` | ✅ 干净还原正文、楼中楼评论、表格结构 |

**别迷信「专用 OCR 模型」** —— 在社交截图这个场景上，通用视觉模型反而更稳。

---

## 六、常见问题

| 现象 | 原因 / 处理 |
|---|---|
| `promoted item changed` | 已入库条目哈希对不上。图文帖走剔除逻辑；其他情况说明笔记被手改过而账本未同步，需人工迁移 |
| `connect ECONNREFUSED 127.0.0.1:7890` | 环境残留了没在运行的代理配置，`unset` 所有 proxy 变量 |
| 图片下载 `IncompleteRead` | 抖音 CDN 偶发断流，重跑该条即可（已缓存图片会跳过） |
| `No module named '_paths'` | 不是以脚本文件路径调用的。用 `<venv>/python <skills目录>/scripts/xxx.py` |
| 怎么判断是不是图文帖 | `play_url` 指向 `obj/ies-music` 且以 `.mp3` 结尾 ⇒ 几乎一定是 |

---

## 七、仓库结构

```
.
├── README.md                    本文件
├── patches/
│   └── 0001-ocr-and-article-support.patch   对上游 4 文件的改动
├── skill/                       技能包（拷进 skills 目录即可用）
│   ├── SKILL.md
│   └── scripts/
│       ├── _paths.py            统一路径解析（换机器必读）
│       ├── collect_images.py    采集图文帖图片列表
│       ├── ocr_images.py        批量图片 OCR
│       ├── merge_ocr.py         回填进已有笔记
│       ├── backfill_misfiltered.py  误筛图文帖补录
│       ├── backfill_report.py   误筛补录报告
│       ├── collect_articles.py  采集长文列表
│       ├── fetch_article_full.py    取长文全文
│       ├── backfill_articles.py 长文回填 + 入库
│       ├── report_articles.py   长文补全报告
│       └── test_integration.py  离线回归测试
├── docs/                        实测报告与架构图
│   ├── 抖音知识库-架构说明.html
│   ├── 图文帖处理验证报告.md
│   ├── 疑似误筛补录报告.md
│   └── 长文正文补全报告.md
└── posts/
    ├── 帖子正文.md                      完整流程长文（Markdown）
    ├── 抖音知识库图文帖与长文补全-公众号版.html   微信公众号排版版
    └── selfcheck.py                     排版自查脚本（逐字比对 + 红线检查）
```

## 八、配套长文

`posts/` 下是完整的落地过程记录：从发现笔记正文为空，到定位两个字段语义误判，再到补丁实现、灰度验证、批量回填，含全部踩坑与实测数据。

- `帖子正文.md` —— 通用 Markdown 版
- `抖音知识库图文帖与长文补全-公众号版.html` —— 公众号排版版（可直接粘贴进后台）

---

## 九、许可与致谢

- 本仓库的补丁与脚本：**MIT**
- 上游项目 `douyin-favorites-to-knowledge` 版权归其作者（MIT，© 2026 Cheng Chen / tars1230）
- 本仓库仅包含补丁与独立脚本，不包含上游源码

## 免责声明

本项目仅用于**个人备份与学习**用途。请遵守抖音用户协议与相关法律法规，不要用于批量抓取、商业分发或任何侵犯他人权益的场景。抓取频率请自行控制。
