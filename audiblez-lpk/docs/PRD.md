# Audiblez 微服 LPK — 产品需求文档（PRD）

| 字段 | 内容 |
|------|------|
| 文档版本 | 0.1.0 |
| 产品代号 | audiblez-lpk |
| 上游应用 | [audiblez](https://github.com/santinic/audiblez) v0.4.9 |
| 目标平台 | 懒猫微服（LazyCat MicroServer）LPK V2 |
| 状态 | 待开发 |

---

## 1. 背景与问题

### 1.1 背景

Audiblez 是一款开源 EPUB → 有声书工具，核心能力为：

- 解析 EPUB 章节与正文
- 使用 Kokoro-82M 进行高质量 TTS
- 输出章节 WAV，经 ffmpeg 合并为 M4B

上游仅提供 **CLI** 与 **wxPython 桌面 GUI**，无法在微服浏览器环境中直接使用。

### 1.2 问题陈述

懒猫微服用户希望在 NAS/微服上：

1. 通过浏览器选择网盘 EPUB，生成有声书
2. 无需在本机安装 Python、PyTorch、ffmpeg
3. 生成结果可保存到懒猫网盘，供手机/播放器使用
4. 长书转换耗时长，需可查看进度、失败可重试

### 1.3 产品定位

将 Audiblez **封装为懒猫微服 LPK 应用**：内嵌 Docker 镜像 + FastAPI Web 操作界面 + 分阶段网盘集成，支持本地 `project build` 与 `lpk install` 部署验证。

**不在 MVP 范围：** 上架提审、GPU 算力舱、多用户队列调度、wxPython GUI 移植。

---

## 2. 目标用户

| 角色 | 诉求 |
|------|------|
| 微服家庭用户 | 把收藏的 EPUB 转成 M4B，在手机听 |
| 开发者（本仓库维护者） | 可复现 build/install，后续可提审 |
| AI 智能体 | 通过技能包文档理解移植边界与验收标准 |

---

## 3. 产品目标（Goals）

| 优先级 | 目标 | 可衡量指标 |
|--------|------|------------|
| P0 | 浏览器可完成「选书 → 转书 → 拿 M4B」 | 测试 EPUB 全流程成功 |
| P0 | 本地 LPK 可 build + install 到默认微服 | `lzc-cli lpk install` 无报错 |
| P0 | 数据持久化 | 重启后 `/lzcapp/var` 任务记录与缓存仍在 |
| P1 | 网盘右键 EPUB 打开应用 | `file_handler` 出现在网盘打开方式 |
| P1 | 网盘选 EPUB、M4B 写回网盘 | inject + document 权限链路通 |
| P1 | 长任务不阻塞 Web | 提交后 3s 内返回任务 ID |
| P2 | GPU 加速 | CPU 50 字/秒 → GPU 500 字/秒（可选） |
| P2 | 提审就绪 | 免密登录、locales、icon 完整 |

---

## 4. 非目标（Out of Scope — MVP）

- 不支持 PDF、TXT、Markdown 直转（上游 audiblez 亦以 EPUB 为主）
- 不支持在线书城、订阅、多人协作
- 不在浏览器内跑 TTS（算力在容器内）
- 不把 `lazycat-skills` 技能文档仓库与 LPK 构建产物混为一目录根
- 不实现 qwen-tts 双引擎切换（依赖已存在但上游未使用）

---

## 5. 功能需求

### 5.1 Web 界面（F-Web）

| ID | 需求 | 优先级 | 说明 |
|----|------|--------|------|
| F-Web-01 | 应用首页可访问 | P0 | `https://audiblez.your-box-name.heiyu.space` |
| F-Web-02 | 选择旁白 voice | P0 | 至少英/中各 1 个默认 voice |
| F-Web-03 | 调节语速 | P0 | 0.5～2.0，默认 1.0 |
| F-Web-04 | 上传 EPUB 或指定路径 | P0/P1 | M2 表单上传；M3 网盘路径 |
| F-Web-05 | 章节勾选 | P1 | 对应 CLI `-p` 交互能力 |
| F-Web-06 | 提交转换任务 | P0 | 异步，立即返回 |
| F-Web-07 | 任务列表与进度 | P0 | 状态：queued/running/done/failed |
| F-Web-08 | 下载 M4B / 写回网盘 | P0/P1 | 完成后提供下载链接或网盘保存 |
| F-Web-09 | 失败原因展示 | P0 | 可读错误信息，可重试 |

### 5.2 后台任务（F-Job）

| ID | 需求 | 优先级 | 说明 |
|----|------|--------|------|
| F-Job-01 | 单容器内任务队列 | P0 | MVP 串行或有限并发（建议 1） |
| F-Job-02 | 调用 audiblez.core | P0 | 不 fork  TTS 逻辑，薄包装 |
| F-Job-03 | 中间 WAV 与最终 M4B 落盘 | P0 | 目录规范见 ARCH |
| F-Job-04 | 任务可取消 | P2 | 运行中进程优雅停止 |
| F-Job-05 | 断点续跑 | P2 | 已存在章节 WAV 跳过（core 已支持） |

### 5.3 网盘集成（F-Netdisk）

| ID | 需求 | 优先级 | 说明 |
|----|------|--------|------|
| F-Netdisk-01 | 从网盘读取 EPUB | P1 | M3 先用浏览器侧 `/_lzc/files/home` fetch，再上传给后端创建任务 |
| F-Netdisk-02 | 写 M4B 回网盘 | P1 | M3 完成后通过浏览器侧 PUT 或明确的保存 API 写回 |
| F-Netdisk-03 | `lzc-file-picker` inject | P1 | 浏览器选文件，传路径给后端 |
| F-Netdisk-04 | 大文件不走浏览器上传 | P1 | >50MB 推荐网盘路径 |

### 5.4 应用关联 / 文件类型（F-FileHandler）

懒猫网盘右键「用此应用打开」依赖 `lzc-manifest.yml` 的 **`application.file_handler`**（官方：[应用关联](https://developer.lazycat.cloud/advanced-mime.html)）。**工具类应用提审也要求与网盘文件类型关联**（见 store-publish 审核指南）。

| ID | 需求 | 优先级 | 说明 |
|----|------|--------|------|
| F-FileHandler-01 | 声明 EPUB 类型 | **P1（M3 / 提审必需）** | 网盘右键 EPUB → 打开本应用 |
| F-FileHandler-02 | 声明 M4B 类型 | **P1** | 网盘右键 M4B → 打开播放/管理 |
| F-FileHandler-03 | `open` 路由解析 `file` | P1 | `actions.open` 中 `%u` 替换为 WebDAV 路径 |
| F-FileHandler-04 | EPUB 打开 → 转换流程 | P1 | 自动创建转换任务或预填表单 |
| F-FileHandler-05 | M4B 打开 → 播放页 | P1 | HTML5 `<audio>` 或跳转播放器 UI |
| F-FileHandler-06 | 多应用时出现在选择器 | P1 | 用户网盘可见「Audiblez」选项 |

**建议声明的 MIME（v1.3.8+）：**

| 文件 | MIME / 扩展名声明 |
|------|-------------------|
| EPUB | `application/epub+zip`、`x-lzc-extension/epub` |
| M4B | `audio/mp4`、`audio/x-m4a`、`x-lzc-extension/m4b`（M4B 常被识别为 `audio/mp4`） |

### 5.5 打包与部署（F-LPK）

| ID | 需求 | 优先级 | 说明 |
|----|------|--------|------|
| F-LPK-01 | LPK V2 三件套 | P0 | `package.yml` / `lzc-manifest.yml` / `lzc-build.yml` |
| F-LPK-02 | 内嵌 Docker 镜像 | P0 | `embed:audiblez` |
| F-LPK-03 | 构建时预烘焙模型 | P0 | spaCy + Kokoro，冷启动 <5min |
| F-LPK-04 | `project deploy` 开发态 | P0 | `lzc-build.dev.yml` |
| F-LPK-05 | `project build` + `lpk install` | P0 | 发布验证路径 |
| F-LPK-06 | icon + locales | P1 | zh-CN / en-US |

### 5.6 权限与安全（F-Sec）

| ID | 需求 | 优先级 | 说明 |
|----|------|--------|------|
| F-Sec-01 | 微服登录态访问 | P0 | 走 ingress 鉴权，不 public_path |
| F-Sec-02 | 权限最小声明 | P0/P1 | M2 不声明文稿权限；M3 增加 `document.read/write` |
| F-Sec-03 | 任务文件仅本应用可见 | P0 | 落在 `/lzcapp/var` |
| F-Sec-04 | 免密登录 inject | P2 | 提审前 |

---

## 6. 非功能需求（NFR）

| 类别 | 要求 |
|------|------|
| 性能 | 短篇测试 EPUB（<3 万字）CPU 下 20 分钟内完成 |
| 镜像体积 | 接受 4～6 GB（含 PyTorch CPU + 模型） |
| 可用性 | 容器崩溃后，已完成任务 M4B 仍可下载 |
| 可维护性 | audiblez 源码与 LPK 包装分层，便于跟进上游版本 |
| 兼容性 | Python 3.12；微服 LPK v1.5.0+ |
| 审核 | 冷启动/首屏可交互 ≤ 5 分钟（模型预烘焙） |

---

## 7. 依赖与环境约束

| 依赖 | 用途 | 处理方式 |
|------|------|----------|
| Python 3.12 | 运行时 | Dockerfile |
| ffmpeg | M4B 合并 | apt 安装；M2 使用原生 `aac` 编码器，避免依赖 `libfdk_aac` |
| espeak-ng | Kokoro 音素 | apt + espeakng-loader |
| Kokoro-82M | TTS | 构建时下载到镜像 |
| xx_ent_wiki_sm | 分句 | 构建时 `spacy download` |
| PyTorch | 推理 | CPU 版默认；GPU 可选 |

---

## 8. 里程碑

| 阶段 | 交付物 | 验收 |
|------|--------|------|
| M0 文档 | PRD / ARCH / CASES | 评审通过 |
| M1 骨架 | Dockerfile + FastAPI 空 Web + 三件套 | `deploy` 后首页可开 |
| M2 核心 | 上传 EPUB → M4B 下载 | 上传类 P0 CASES 全绿 |
| M3 网盘与应用关联 | `file_handler` 右键 EPUB + inject + 网盘读写 | CASE-FH 与 CASE-NETDISK 绿 |
| M4 发布包 | `project build` + `lpk install` | 发布包部署 CASES 绿 |
| M5 提审准备 | 免密、资料、GPU 可选 | publish-checklist |

---

## 9. 风险登记

| 风险 | 影响 | 缓解 |
|------|------|------|
| 镜像过大 build 超时 | 无法本地验证 | 分层 Dockerfile；远程 build 耐心等待 |
| CPU 转长篇过慢 | 用户失望 | MVP 文案说明；P2 GPU |
| 长任务 OOM | 容器被杀 | 限制并发；监控内存 |
| 上游 audiblez 升级破坏 API | 包装层失效 | 薄包装 + 版本锁定 0.4.9 |
| 网盘 inject 兼容性 | 选书失败 | 保留本地上传兜底 |
| ffmpeg 编码器不可用 | M4B 合成失败 | 使用 Debian 默认 ffmpeg 支持的原生 `aac`，M2 前验证 `ffmpeg -encoders` |

---

## 10. 已拍板决策

| # | 决策 | 说明 |
|---|------|------|
| D-1 | Web 框架使用 FastAPI + 简单 HTML/HTMX | `/api/jobs`、`/open`、下载接口和路径安全更直接 |
| D-2 | 正式包名 `cloud.lazycat.app.audiblez`，开发包名 `cloud.lazycat.app.audiblez.dev` | subdomain 固定 `audiblez` |
| D-3 | M2 只做上传 EPUB → M4B 下载；M3 再做 `file_handler`、inject、网盘写回 | 避免网盘问题阻塞核心 TTS 验证 |
| D-4 | M3 网盘读取先走浏览器侧 `/_lzc/files/home` fetch，再提交给后端 | 不让后端猜 WebDAV 到容器路径 |
| D-5 | CPU-only MVP，GPU 作为 P2；Kokoro/spaCy 构建时 bake | 冷启动可控，兼容普通微服 |
| D-6 | ffmpeg 统一使用原生 `aac` | 避免 Debian 默认 ffmpeg 不含 `libfdk_aac` |

---

## 11. 术语

| 术语 | 含义 |
|------|------|
| LPK | 懒猫应用安装包 |
| Kokoro-82M | 82M 参数 TTS 模型 |
| M4B | 有声书常用容器格式 |
| inject | 微服向页面注入 JS（网盘选文件等） |
| embed 镜像 | 构建时打进 LPK 的 Docker 镜像 |
