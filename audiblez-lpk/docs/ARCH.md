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
│  application.injects: browser → lzc-file-picker (P1)            │
├─────────────────────────────────────────────────────────────────┤
│ Service: audiblez (embed 镜像)                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │ Web Layer    │  │ Job Manager  │  │ audiblez (upstream)    │ │
│  │ Gradio/FAPI  │→ │ 队列+状态机  │→ │ core.py / voices.py    │ │
│  └──────────────┘  └──────────────┘  └────────────────────────┘ │
│         │                  │                    │                │
│         └──────────────────┴────────────────────┘                │
│                            │                                     │
│  ┌─────────────────────────┴─────────────────────────┐           │
│  │ 持久化                                             │           │
│  │  /lzcapp/var/{jobs,huggingface,cache}            │           │
│  │  /lzcapp/documents  (网盘 EPUB 输入 / M4B 输出)   │           │
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
├── vendor/
│   └── audiblez/                 # 上游源码拷贝或 submodule（与根 audiblez/ 同步）
├── web/
│   ├── app.py                    # 入口：启动 Gradio / FastAPI
│   ├── job_manager.py            # 任务 CRUD、状态、持久化
│   ├── job_runner.py             # 包装 core.main，进度回调
│   ├── paths.py                  # /lzcapp 路径常量
│   └── static/                   # 可选：自定义 CSS
├── content/                      # inject 资源（lzc-build contentdir）
│   └── lazycat-injects/
│       └── lzc-file-chooser-inject.js
├── pyproject.toml                # web 层额外依赖（若与 vendor 分离）
└── README.md
```

**与当前 monorepo 关系：**

```text
lazycat-skills/audiblez/          # 上游 audiblez 源码（已存在）
audiblez-lpk/vendor/audiblez/     # 构建时复制或 submodule 指向同级
```

---

## 3. 组件设计

### 3.1 Web Layer（`web/app.py`）

**职责：**

- 提供 HTTP UI 与 REST API
- 校验上传文件（扩展名 `.epub`，大小上限可配置）
- 创建任务、查询任务、下载结果

**推荐技术选型（MVP）：**

| 选项 | 优点 | 缺点 | 结论 |
|------|------|------|------|
| Gradio | 快速、依赖已在 kokoro 链 | 定制 inject 稍麻烦 | **MVP 首选** |
| FastAPI + HTMX | 灵活、API 清晰 | 开发量更大 | P2 或并行 |

**监听：**

- `0.0.0.0:7860`（Gradio 默认）或 `8080`（与 nginx 模式统一时用 8080）
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

- 包装 `gen_audio_segments` 中的 `post_event` 或 monkey-patch stats
- 定期写 `meta.json`（每章或每 N 句）

**线程模型：**

```text
Main thread: Gradio / Uvicorn
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

在 `audiblez/core.py` 增加无 argparse 的 `run_epub(...)` 函数；MVP 可在 vendor 内 fork 一行包装，避免改 monorepo 根 `audiblez/`。

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
Stage deps:     pip install audiblez + web extras
Stage models:   spacy download + KPipeline warmup（英+中 voice 各一次可选）
Stage runtime:  复制应用 + 非 root 用户（可选）
```

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
  test: ["CMD", "curl", "-f", "http://127.0.0.1:7860/"]
  interval: 30s
  timeout: 10s
  start_period: 120s
```

MVP 若 Gradio 无轻量 health 路径，可用 `curl` 首页或自建 `/api/health`。

---

## 6. LPK 配置架构

### 6.1 `package.yml`（元数据 + 权限）

```yaml
package: cloud.lazycat.app.audiblez
version: 0.1.0
name: Audiblez
permissions:
  required:
    - document.read
    - document.write
  optional:
    - net.internet      # 模型未完全 bake 时兜底
    - device.dri.render   # GPU P2
```

MVP 若模型全 bake，`net.internet` 可降为 optional 或不声明。

### 6.2 `lzc-manifest.yml`（运行时）

```yaml
lzc-sdk-version: "0.1"
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
    - stage: browser
      files:
        - lazycat-file-chooser-inject.js
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

### 7.1 应用关联（file_handler）— 网盘右键打开

与「应用内 file-picker inject」是**两条入口**，都要支持：

```text
懒猫网盘 / 文件管理器
    │ 用户右键 EPUB 或 M4B →「打开方式」→ Audiblez
    ▼
系统根据 file_handler.mime 匹配应用
    ▼
打开 URL: https://audiblez.<微服>.heiyu.space/open?file=<WebDAV路径>
    ▼
Web /open 处理器:
    ├─ 后缀 .epub → mode=convert → 创建任务或预填转换页
    └─ 后缀 .m4b  → mode=play     → 播放器页（HTML5 audio）
    ▼
读文件:
    浏览器侧 GET /_lzc/files/home + normalize(path)
    或服务端读 /lzcapp/documents（需 document.read + 路径映射）
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

### 7.2 应用内选文件（inject + file-picker）

```text
浏览器主页面
    │ inject: lzc-file-picker
    ▼
用户点击「从网盘选择」→ 得到文稿相对路径
    ▼
POST /api/jobs { "source_type": "document", "path": "..." }
    ▼
JobRunner 从 /lzcapp/documents/<path> 读取
    ▼
完成后：
  方案 A: Web 下载 M4B（GET /api/jobs/<id>/download）
  方案 B: 复制到 /lzcapp/documents/<user_chosen_dir>/book.m4b
  方案 C: 前端 Blob + PUT /_lzc/files/...（见 lazycat-lpk-netdisk）
```

**MVP 优先：** file_handler（右键 EPUB）+ 方案 A；inject 与 M4B 播放为 P1。

---

## 8. API 设计（REST，供 Gradio 内部或前后端分离）

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/health` | 健康检查 |
| GET | `/api/voices` | voice 列表 |
| POST | `/api/jobs` | 创建任务（multipart 或 JSON path） |
| GET | `/api/jobs` | 任务列表 |
| GET | `/api/jobs/{id}` | 任务详情 + 进度 |
| GET | `/api/jobs/{id}/download` | 下载 M4B |
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
| v0.1.0 MVP | 单服务 embed 镜像 + Gradio + 本地上传 |
| v0.2.0 | 网盘 inject + document 读写 |
| v0.3.0 | GPU optional + CUDA torch 镜像变体 |
| v0.4.0 | FastAPI 替换 Gradio；任务 cancel |
| v1.0.0 | 提审：免密 inject、copy-image、商店元数据 |

---

## 13. 技术决策记录（ADR）

| ID | 决策 | 理由 |
|----|------|------|
| ADR-01 | 独立 `audiblez-lpk/` 目录 | 符合 AGENTS.md，技能仓与 LPK 分离 |
| ADR-02 | Python 3.12 in Docker | 匹配 audiblez `requires-python` |
| ADR-03 | 构建时 bake 模型 | 满足审核冷启动、离线可用 |
| ADR-04 | 单 worker 任务队列 | 避免 OOM；MVP 足够 |
| ADR-05 | Gradio MVP | 依赖链已有；最快验证 |
| ADR-06 | CPU PyTorch 默认 | 微服通用性；GPU 作 optional 镜像 |
