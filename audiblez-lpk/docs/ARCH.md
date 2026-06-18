# Audiblez 微服 LPK — 架构设计（ARCH）

| 字段 | 内容 |
|------|------|
| 文档版本 | 0.1.0 |
| 关联 PRD | `docs/PRD.md` v0.1.0 |
| 状态 | 待实现 |

---

## 1. 架构总览

```text
┌─────────────────────────────────────────────────────────────────┐
│ 懒猫微服 Ingress（OIDC / 登录态）                                │
└────────────────────────────┬────────────────────────────────────┘
                             │ HTTPS
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ LPK: cloud.lazycat.app.audiblez (subdomain: audiblez)           │
│  application.routes: /=http://audiblez:7860                      │
│  application.file_handler / injects: M3 网盘入口                 │
├─────────────────────────────────────────────────────────────────┤
│ Service: audiblez (embed 镜像)                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │ Web Layer    │  │ Job Manager  │  │ audiblez (upstream)    │ │
│  │ FastAPI+HTML │→ │ 队列+状态机  │→ │ core.py / voices.py    │ │
│  └──────────────┘  └──────────────┘  └────────────────────────┘ │
│         │                  │                    │                │
│         └──────────────────┴────────────────────┘                │
│                            │                                     │
│  ┌─────────────────────────┴─────────────────────────┐           │
│  │ 持久化                                             │           │
│  │  /lzcapp/var/{jobs,huggingface,cache}            │           │
│  │  /lzcapp/documents  (M3 网盘 EPUB 输入 / M4B 输出)│           │
│  └───────────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

**设计原则：**

1. **薄包装**：不重写 TTS；`JobRunner` 调用现有 `audiblez.core.main()` 或等价函数。
2. **异步任务**：HTTP 只负责提交与查询，TTS 在后台线程/进程执行。
3. **模型内置**：构建镜像时下载模型，运行时默认不依赖 HuggingFace。
4. **仓库隔离**：`audiblez-lpk/` 独立目录，不污染 `lazycat-skills/skills/`。

---

## 2. 目录结构（目标）

```text
audiblez-lpk/
├── docs/
│   ├── PRD.md
│   ├── ARCH.md
│   └── CASES.md
├── lzc/                          # lzc-cli 工作目录（build/deploy 在此执行）
│   ├── package.yml
│   ├── lzc-manifest.yml
│   ├── lzc-build.yml
│   ├── lzc-build.dev.yml
│   └── icon.png
├── docker/
│   └── Dockerfile
├── web/
│   ├── app.py                    # 入口：启动 FastAPI / Uvicorn
│   ├── job_manager.py            # 任务 CRUD、状态、持久化
│   ├── job_runner.py             # 包装 core.main，进度回调
│   ├── paths.py                  # /lzcapp 路径常量
│   ├── templates/                # 简单 HTML / HTMX
│   └── static/                   # CSS / JS
├── content/                      # inject 资源（lzc-build contentdir）
│   └── lazycat-injects/
│       └── lzc-file-chooser-inject.js
├── pyproject.toml                # web 层额外依赖（若与 vendor 分离）
└── README.md
```

**与当前 monorepo 关系：**

```text
lazycat-skills/audiblez/audiblez/ # 上游 Python 包源码（已存在）
audiblez-lpk/                     # LPK 包装层；Docker build context 直接复制仓库根的 audiblez/
```

---

## 3. 组件设计

### 3.1 Web Layer（`web/app.py`）

**职责：**

- 提供 HTTP UI 与 REST API
- 校验上传文件（扩展名 `.epub`，大小上限可配置）
- 创建任务、查询任务、下载结果

**技术选型（已拍板）：**

MVP 使用 **FastAPI + Uvicorn + 简单 HTML/HTMX**。理由：`/api/jobs`、`/open?file=%u`、下载接口、路径安全校验和后续网盘接入更直接；避免在文档中同时维护两套 Web 框架心智模型。

**监听：**

- `0.0.0.0:7860`
- manifest `routes` 与容器 `EXPOSE` 保持一致

### 3.2 Job Manager（`web/job_manager.py`）

**任务状态机：**

```text
queued → running → done
              ↘ failed
              ↘ cancelled (P2)
```

**任务元数据（JSON，`/lzcapp/var/jobs/<job_id>/meta.json`）：**

```json
{
  "id": "20260618-abc123",
  "created_at": "2026-06-18T10:00:00Z",
  "updated_at": "2026-06-18T10:05:00Z",
  "status": "running",
  "voice": "af_sky",
  "speed": 1.0,
  "source": {
    "type": "upload",
    "path": "/lzcapp/var/jobs/.../input.epub"
  },
  "output": {
    "m4b": "/lzcapp/var/jobs/.../output.m4b",
    "chapters_dir": "/lzcapp/var/jobs/.../wav/"
  },
  "progress": {
    "total_chars": 120000,
    "processed_chars": 45000,
    "percent": 37,
    "eta_seconds": 900
  },
  "error": null
}
```

**并发策略（MVP）：**

- 全局 `max_running_jobs = 1`，避免 CPU/内存争抢
- 新任务 `queued`，当前任务结束后自动拉起

### 3.3 Job Runner（`web/job_runner.py`）

**执行流程：**

```text
1. 复制/确认 input.epub 在 job 目录
2. set_espeak_library()（core 内逻辑）
3. KPipeline(lang_code=voice[0])
4. 对每个选中章节调用 gen_audio_segments + soundfile.write
5. 若 ffmpeg 可用：create_index_file + create_m4b
6. 更新 meta.json → done / failed
```

**进度回调：**

`audiblez.core.main(..., post_event=callback)` 将事件映射到 `meta.json.progress`：

| core 事件 | 写入字段 |
|-----------|----------|
| `CORE_STARTED` | `status=running`、`started_at` |
| `CORE_CHAPTER_STARTED` | `current_chapter_index` |
| `CORE_PROGRESS` | `total_chars`、`processed_chars`、`percent`、`eta_seconds` |
| `CORE_CHAPTER_FINISHED` | `finished_chapters[]` |
| `CORE_FINISHED` | `status=done`、`finished_at` |

写入频率：收到事件立即原子写 `meta.json.tmp → rename(meta.json)`；若事件过密，至少每 2 秒写一次。

**线程模型：**

```text
Main thread: FastAPI / Uvicorn
Worker thread: JobRunner.run(job_id)   # MVP 单 worker
```

P2 可改为 `multiprocessing` 以便 cancel 杀进程。

### 3.4 上游 Audiblez 集成边界

| 模块 | 用法 | 不改 |
|------|------|------|
| `audiblez.core.main` | 主转换流程 | 优先直接调；若 CLI argparse 耦合则抽 `run_conversion(...)` |
| `audiblez.core.find_document_chapters_and_extract_texts` | 章节列表 | 是 |
| `audiblez.voices.voices` | voice 列表 UI | 是 |
| `audiblez.ui` | wxPython GUI | **不使用** |
| `audiblez.cli` | argparse | **不使用**（Web 替代） |

**若需最小改动上游（可选后续）：**

在 `audiblez/core.py` 增加无 argparse 的 `run_epub(...)` 函数；MVP 允许直接改本 fork 的根 `audiblez/`，不再引入 `vendor/` 或 submodule。

---

## 4. 存储设计

### 4.1 路径规范

| 路径 | 用途 | bind |
|------|------|------|
| `/lzcapp/var/jobs/<id>/` | 单任务工作区 | `services.audiblez.binds` |
| `/lzcapp/var/jobs/<id>/input.epub` | 输入 | — |
| `/lzcapp/var/jobs/<id>/wav/` | 章节 WAV | — |
| `/lzcapp/var/jobs/<id>/output.m4b` | 输出 | — |
| `/lzcapp/var/jobs/<id>/meta.json` | 任务元数据 | — |
| `/lzcapp/var/huggingface/` | HF 缓存（兜底） | `HF_HOME` |
| `/lzcapp/documents/` | 用户文稿（网盘） | 同 binds |

### 4.2 环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `AUDIBLEZ_HOST` | `0.0.0.0` | Web 绑定 |
| `AUDIBLEZ_PORT` | `7860` | Web 端口 |
| `AUDIBLEZ_JOBS_ROOT` | `/lzcapp/var/jobs` | 任务根目录 |
| `AUDIBLEZ_OUTPUT_ROOT` | `/lzcapp/var/output` | 可选汇总输出 |
| `HF_HOME` | `/lzcapp/var/huggingface` | 模型缓存 |
| `AUDIBLEZ_MAX_UPLOAD_MB` | `200` | 上传限制 |
| `AUDIBLEZ_USE_CUDA` | `0` | 1 时尝试 CUDA |

---

## 5. Docker 镜像

### 5.1 多阶段构建（推荐）

```text
Stage base:     python:3.12-slim-bookworm + ffmpeg + espeak-ng
Stage deps:     pip install audiblez + fastapi + uvicorn + python-multipart + jinja2
Stage models:   设置 HF_HOME=/opt/audiblez-models，spacy download + KPipeline warmup
Stage runtime:  复制应用 + 非 root 用户（可选）
```

**模型预烘焙要求：**

```bash
python -m spacy download xx_ent_wiki_sm
python - <<'PY'
from kokoro import KPipeline
KPipeline(lang_code="a")  # M2 默认英文 voice，例如 af_sky
KPipeline(lang_code="z")  # M3/P1 中文 voice，例如 zf_xiaoxiao
PY
```

运行时默认离线：

```dockerfile
ENV HF_HOME=/opt/audiblez-models
ENV HF_HUB_OFFLINE=1
ENV TRANSFORMERS_OFFLINE=1
```

通过标准：容器启动和 M2 冒烟转换期间日志中不出现 HuggingFace 下载进度；若缺模型应在构建阶段失败，而不是运行时下载。

### 5.2 镜像体积预估

| 层 | 约 |
|----|-----|
| base + apt | 200 MB |
| PyTorch CPU | 500 MB |
| Python 依赖 | 800 MB |
| Kokoro + voices | 300～1100 MB |
| spaCy 模型 | 40 MB |
| **合计** | **~4～6 GB** |

### 5.3 健康检查

```yaml
healthcheck:
  test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:7860/api/health', timeout=5)"]
  interval: 30s
  timeout: 10s
  start_period: 120s
```

### 5.4 ffmpeg 编码器约束

上游 `audiblez/core.py` 的临时拼接阶段曾使用 `libfdk_aac`，Debian 默认 ffmpeg 通常不带该编码器。M2 前必须改为原生 `aac`，并在容器内验证：

```bash
ffmpeg -hide_banner -encoders | grep -E '(^| )A.*aac'
```

通过标准：`CASE-JOB-01` 能生成 `output.m4b`，不因 `Unknown encoder 'libfdk_aac'` 失败。

---

## 6. LPK 配置架构

### 6.1 `package.yml`（元数据 + 权限）

M2（上传-only MVP）最小权限：

```yaml
package: cloud.lazycat.app.audiblez
version: 0.1.0
name: Audiblez
permissions:
  optional:
    - net.internet      # 模型未完全 bake 时兜底
    - device.dri.render   # GPU P2
```

M3（网盘 / 应用关联）追加权限：

```yaml
permissions:
  required:
    - document.read
    - document.write
  optional:
    - net.internet
    - device.dri.render
```

若仅使用浏览器侧 `/_lzc/files/home` 读取，后端仍应把文稿权限声明作为 M3 验收项，因为保存 M4B 回网盘需要写权限。

### 6.2 `lzc-manifest.yml`（运行时）

M2 最小 manifest：

```yaml
application:
  subdomain: audiblez
  routes:
    - /=http://audiblez:7860
services:
  audiblez:
    image: embed:audiblez
    binds:
      - /lzcapp/var:/lzcapp/var
```

M3 追加 `file_handler` 与 inject：

```yaml
application:
  subdomain: audiblez
  routes:
    - /=http://audiblez:7860
    - /open=http://audiblez:7860/open
  file_handler:
    mime:
      - application/epub+zip
      - x-lzc-extension/epub
      - audio/mp4
      - audio/x-m4a
      - x-lzc-extension/m4b
    actions:
      open: /open?file=%u
  injects:
    - id: open-save-chooser
      on: browser
      when:
        - /*
      do:
        - src: file:///lzcapp/pkg/content/lazycat-injects/lzc-file-chooser-inject.js
          params:
            diskRoot: /_lzc/files/home
            fallbackMime: application/octet-stream
            locale: auto
            hooks:
              fileInput: true
              fileSystemAccess: true
services:
  audiblez:
    image: embed:audiblez
    binds:
      - /lzcapp/var:/lzcapp/var
      - /lzcapp/documents:/lzcapp/documents
```

`routes` 中的 `/open` 须与 `file_handler.actions.open` 一致；系统打开文件时把 `%u` 替换为 WebDAV 路径（见 [应用关联](https://developer.lazycat.cloud/advanced-mime.html)）。

### 6.3 `lzc-build.yml` vs `lzc-build.dev.yml`

| 文件 | 用途 |
|------|------|
| `lzc-build.yml` | `project build` → 正式 LPK |
| `lzc-build.dev.yml` | `project deploy` → 开发迭代 |

差异示例：dev 可挂载本地 `web/` 热更新（若平台支持）或缩短 healthcheck。

```yaml
# lzc-build.yml
images:
  audiblez:
    dockerfile: ../docker/Dockerfile
    context: ..
contentdir: ../content
pkgout: .
```

---

## 7. 网盘集成架构

### 7.1 应用关联（file_handler）— 网盘右键打开（M3）

与「应用内 file-picker inject」是**两条入口**，都要支持：

```text
懒猫网盘 / 文件管理器
    │ 用户右键 EPUB 或 M4B →「打开方式」→ Audiblez
    ▼
系统根据 file_handler.mime 匹配应用
    ▼
打开 URL: https://audiblez.your-box-name.heiyu.space/open?file=<WebDAV路径>
    ▼
Web /open 处理器:
    ├─ 后缀 .epub → mode=convert → 创建任务或预填转换页
    └─ 后缀 .m4b  → mode=play     → 播放器页（HTML5 audio）
    ▼
权威数据源:
    浏览器侧 GET /_lzc/files/home + normalize(path)
    读到 Blob/File 后 POST /api/jobs
```

**`%u` 路径处理（与 lazycat-lpk-netdisk 一致）：**

```javascript
function normalizeLazyCatPath(path) {
  let normalized = String(path || '').trim().replace(/\.$/, '');
  normalized = normalized.replace(/^\/_lzc\/files\/home(?=\/|$)/, '');
  if (normalized && !normalized.startsWith('/')) normalized = '/' + normalized;
  return normalized;
}
```

### 7.2 应用内选文件（inject + file-picker，M3）

```text
浏览器主页面
    │ inject: lzc-file-picker
    ▼
用户点击「从网盘选择」→ 得到文稿相对路径
    ▼
浏览器 GET /_lzc/files/home + normalize(path) → File/Blob
    ▼
POST /api/jobs multipart 上传给后端
    ▼
完成后：
  方案 A: Web 下载 M4B（GET /api/jobs/<id>/download）
  方案 B: 前端 Blob + PUT /_lzc/files/home/<chosen path>（见 lazycat-lpk-netdisk）
  方案 C: 后端保存 API（仅当前端 PUT 被权限/API 证明确实阻塞时）
```

**M2 不实现网盘读取。M3 优先：** file_handler 右键 EPUB + 浏览器侧 fetch + 方案 A 下载；M4B 播放和写回网盘属于 P1。

---

## 8. API 设计（FastAPI）

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/health` | 健康检查 |
| GET | `/api/voices` | voice 列表 |
| POST | `/api/jobs` | 创建任务（multipart 或 JSON path） |
| GET | `/api/jobs` | 任务列表 |
| GET | `/api/jobs/{id}` | 任务详情 + 进度 |
| GET | `/api/jobs/{id}/download` | 下载 M4B |
| GET | `/open` | M3 file_handler 入口，解析 `file` query 后渲染转换页 |
| GET | `/play` | P1 M4B 播放页 |
| GET | `/api/files/stream` | P1 音频流代理，或明确改用 `/_lzc/files/home` |
| DELETE | `/api/jobs/{id}` | 取消/删除（P2） |

---

## 9. 安全与权限

```text
用户请求 → 微服 Ingress 鉴权 → 仅登录用户可访问
容器内不暴露 public_path（MVP）
任务文件路径禁止目录穿越（规范化 path，禁止 ..）
上传 EPUB 仅允许 .epub 扩展名
```

---

## 10. 可观测性

| 项 | 实现 |
|----|------|
| 应用日志 | stdout → `lzc-cli project log -f` |
| 任务日志 | `/lzcapp/var/jobs/<id>/run.log` |
| 指标（P2） | 任务耗时、字符数、失败率 |

---

## 11. 部署流程

```text
开发迭代:
  cd audiblez-lpk/lzc
  lzc-cli project deploy
  lzc-cli project log -f

发布验证:
  lzc-cli project build
  lzc-cli lpk install ./cloud.lazycat.app.audiblez-0.1.0.lpk
  浏览器验收 CASES P0
```

---

## 12. 演进路线

| 版本 | 架构变更 |
|------|----------|
| v0.1.0 MVP | 单服务 embed 镜像 + FastAPI + 本地上传 |
| v0.2.0 | file_handler + 网盘 inject + 浏览器侧读写 |
| v0.3.0 | GPU optional + CUDA torch 镜像变体 |
| v0.4.0 | 任务 cancel / 多任务管理 |
| v1.0.0 | 提审：免密 inject、copy-image、商店元数据 |

---

## 13. 技术决策记录（ADR）

| ID | 决策 | 理由 |
|----|------|------|
| ADR-01 | 独立 `audiblez-lpk/` 目录 | 符合 AGENTS.md，技能仓与 LPK 分离 |
| ADR-02 | Python 3.12 in Docker | 匹配 audiblez `requires-python` |
| ADR-03 | 构建时 bake 模型 | 满足审核冷启动、离线可用 |
| ADR-04 | 单 worker 任务队列 | 避免 OOM；MVP 足够 |
| ADR-05 | FastAPI + HTML/HTMX MVP | `/api/jobs`、`/open`、下载和路径安全更直接 |
| ADR-06 | CPU PyTorch 默认 | 微服通用性；GPU 作 optional 镜像 |
| ADR-07 | M2 不做网盘，M3 做 file_handler/inject | 先验证核心 TTS，再接微服生态入口 |
| ADR-08 | 网盘读写先走浏览器侧 `/_lzc/files/home` | 避免后端猜 WebDAV 到容器路径 |
| ADR-09 | ffmpeg 使用原生 `aac` | 避免 Debian 默认 ffmpeg 缺少 `libfdk_aac` |
