# Audiblez 微服 LPK — 用例与验收（CASES）

| 字段 | 内容 |
|------|------|
| 文档版本 | 0.1.0 |
| 关联 PRD | `docs/PRD.md` v0.1.0 |
| 关联 ARCH | `docs/ARCH.md` v0.1.0 |
| 状态 | 待执行 |

---

## 1. 用例编写说明

- **优先级**：P0（MVP 必过）/ P1（第二迭代）/ P2（提审或增强）
- **验收**：可客观判断通过/失败
- **测试数据**：统一使用 Daisy 无障碍测试 EPUB（小体积、可重复）
- **阶段边界**：M2 只验证上传 EPUB → M4B 下载；M3 再验证 `file_handler`、inject 与网盘读写。

```bash
# 标准测试 EPUB（约 1～3 万字量级）
wget -O test-book.epub \
  "https://github.com/daisy/epub-accessibility-tests/releases/download/fundamental-2.0/Fundamental-Accessibility-Tests-Basic-Functionality-v2.0.0.epub"
```

**环境前置（所有用例）：**

```bash
lzc-cli box default          # 确认默认微服可达
cd audiblez-lpk/lzc
```

---

## 2. 用户故事（User Stories）

### US-01 家庭用户听书

> 作为微服用户，我想在浏览器里把网盘 EPUB 转成 M4B，以便在手机播放器里听。

**验收：** US-01-A（P1 网盘）或 US-01-B（P0 上传）任一通过。

### US-02 开发者本地验证

> 作为开发者，我想 `project build` 后 `lpk install` 到自家微服，确认与用户安装体验一致。

**验收：** CASE-DEPLOY-01～03。

### US-03 长书进度可见

> 作为用户，转换需要较长时间时，我能看到进度百分比，知道还要等多久。

**验收：** CASE-JOB-03。

---

## 3. 功能用例

### 3.1 部署与启动（CASE-DEPLOY）

| ID | 优先级 | 场景 | 步骤 | 预期结果 |
|----|--------|------|------|----------|
| CASE-DEPLOY-01 | P0 | 开发态部署 | `lzc-cli project deploy` | 命令成功，无 manifest 校验错误 |
| CASE-DEPLOY-02 | P0 | 发布包构建 | `lzc-cli project build` | 产出 `*.lpk`，体积 >100MB（含镜像） |
| CASE-DEPLOY-03 | P0 | 正式安装 | `lzc-cli lpk install ./<pkg>.lpk` | 安装成功，打印 HTTPS 入口 URL |
| CASE-DEPLOY-04 | P0 | 首页可访问 | 浏览器打开 `https://audiblez.your-box-name.heiyu.space` | 200，显示 Web UI，需微服登录态 |
| CASE-DEPLOY-05 | P0 | 冷启动时间 | 安装后首次打开（模型已 bake） | 首屏可交互 ≤ 5 分钟 |
| CASE-DEPLOY-06 | P0 | 容器日志 | `lzc-cli project log -f` | 无持续 crash loop |
| CASE-DEPLOY-07 | P1 | lpk info | `lzc-cli lpk info <lpk>` | 显示 package、version、镜像信息 |
| CASE-DEPLOY-08 | P1 | 元数据完整 | 检查 `package.yml` 和 `lzc-build.yml` | 有 icon、zh-CN/en-US locales、name/description |

### 3.2 Web 界面（CASE-WEB）

| ID | 优先级 | 场景 | 步骤 | 预期结果 |
|----|--------|------|------|----------|
| CASE-WEB-01 | P0 | Voice 列表 | 打开首页 | 展示多语言 voice，含 `af_sky`、`zf_xiaoxiao` 等 |
| CASE-WEB-02 | P0 | 语速调节 | 设置 speed=1.5 并提交 | 任务 meta 中 speed=1.5 |
| CASE-WEB-03 | P0 | 上传 EPUB | 上传 `test-book.epub` | 创建任务，返回 job_id |
| CASE-WEB-04 | P0 | 非法文件 | 上传 `.txt` | 4xx，友好错误，无任务创建 |
| CASE-WEB-05 | P0 | 超大文件 | `dd if=/dev/zero of=large.epub bs=1M count=$((AUDIBLEZ_MAX_UPLOAD_MB+1))` 后上传 | 拒绝并提示 |
| CASE-WEB-06 | P1 | 章节选择 | 勾选部分章节后提交 | 仅选中章节生成 WAV/M4B |
| CASE-WEB-07 | P0 | 任务列表 | 提交后查看列表 | 显示 queued/running/done |

### 3.3 任务与转换（CASE-JOB）

| ID | 优先级 | 场景 | 步骤 | 预期结果 |
|----|--------|------|------|----------|
| CASE-JOB-01 | P0 | 端到端转换 | 上传 test-book.epub，voice=`af_sky`，等待完成 | status=done，存在 `output.m4b` |
| CASE-JOB-02 | P0 | M4B 可播放 | 下载 M4B，本地 ffprobe/ffmpeg 或播放器 | 有音频流，时长 >0 |
| CASE-JOB-03 | P0 | 进度更新 | 转换过程中轮询 `GET /api/jobs/{id}` | `processed_chars` 递增，`percent` 变化 |
| CASE-JOB-04 | P0 | 异步提交 | POST 创建任务后 3s 内响应 | 不阻塞至 TTS 完成 |
| CASE-JOB-05 | P0 | 失败可诊断 | 故意提交损坏 epub | status=failed，`error` 字段非空 |
| CASE-JOB-06 | P1 | 中文 voice | voice=`zf_xiaoxiao`，中文 EPUB | 成功生成，无 espeak 崩溃 |
| CASE-JOB-07 | P1 | 断点续跑 | 任务中途中断容器，重启后同 job 目录已有 wav | 重跑跳过已有章节（core 行为） |
| CASE-JOB-08 | P2 | 取消任务 | 运行中点击取消 | status=cancelled，进程停止 |
| CASE-JOB-09 | P2 | 并发限制 | 同时提交 2 个任务 | 第二个 queued，第一个完成后第二个开始 |

### 3.4 持久化（CASE-PERSIST）

| ID | 优先级 | 场景 | 步骤 | 预期结果 |
|----|--------|------|------|----------|
| CASE-PERSIST-01 | P0 | 任务数据持久 | 完成任务后 `lzc-cli project restart`（或微服内重启应用） | `/lzcapp/var/jobs/<id>/` 仍在，M4B 可下载 |
| CASE-PERSIST-02 | P0 | HF 缓存持久 | 第二次转换同 voice | 不重新下载 Kokoro（或明显快于首次） |
| CASE-PERSIST-03 | P1 | 卸载保留策略 | 卸载应用（若平台支持） | 按平台策略；文档说明用户数据路径 |

### 3.5 应用关联 / 网盘右键打开（CASE-FILEHANDLER）

官方文档：[应用关联](https://developer.lazycat.cloud/advanced-mime.html)

| ID | 优先级 | 场景 | 步骤 | 预期结果 |
|----|--------|------|------|----------|
| CASE-FH-01 | P1 | EPUB manifest 声明 | 检查 `file_handler.mime` 含 EPUB | YAML 含 `application/epub+zip`、`x-lzc-extension/epub` |
| CASE-FH-02 | P1 | 网盘识别 EPUB | 懒猫网盘右键某 EPUB | 打开方式列表出现「Audiblez」 |
| CASE-FH-03 | P1 | 右键打开 EPUB | 选择 Audiblez 打开 | 跳转到 `/open?file=...`，进入转换页或自动创建任务 |
| CASE-FH-04 | P1 | `/open` 解析路径 | 带 `file` 参数的 URL | 应用正确解析 `%u` 路径，无目录穿越 |
| CASE-FH-05 | P1 | 网盘识别 M4B | 右键 `.m4b` 文件 | 打开方式列表出现 Audiblez |
| CASE-FH-06 | P1 | 右键打开 M4B | 选择 Audiblez | 进入播放页，`<audio>` 可播放 |
| CASE-FH-07 | P1 | M4B manifest 声明 | 检查 `file_handler.mime` 含 M4B | YAML 含 `audio/mp4`、`audio/x-m4a`、`x-lzc-extension/m4b` |

### 3.6 网盘集成（inject / 读写）（CASE-NETDISK）

| ID | 优先级 | 场景 | 步骤 | 预期结果 |
|----|--------|------|------|----------|
| CASE-NETDISK-01 | P1 | inject 加载 | 打开首页，检查 inject JS | 无 404，控制台无 inject 致命错误 |
| CASE-NETDISK-02 | P1 | 网盘选 EPUB | 点击「从网盘选择」，选 EPUB | 路径回填到表单 |
| CASE-NETDISK-03 | P1 | 服务端读文稿 | 提交网盘路径任务 | 从 `/lzcapp/documents` 正确读取并转换 |
| CASE-NETDISK-04 | P1 | M4B 写回网盘 | 完成后「保存到网盘」 | 网盘可见 M4B，可播放 |
| CASE-NETDISK-05 | P1 | 权限未声明 | 临时去掉 `document.read` 重建 | 读文稿失败，错误明确 |

### 3.7 安全（CASE-SEC）

| ID | 优先级 | 场景 | 步骤 | 预期结果 |
|----|--------|------|------|----------|
| CASE-SEC-01 | P0 | 未登录访问 | 无微服 cookie 访问 URL | 跳转登录或 401/403 |
| CASE-SEC-02 | P0 | 路径穿越 | POST path=`../../../etc/passwd` | 拒绝，无文件读取 |
| CASE-SEC-03 | P2 | 免密登录 | 配置 inject/OIDC 后 | 符合提审 checklist |

---

## 4. 非功能用例（CASE-NFR）

| ID | 优先级 | 场景 | 步骤 | 预期结果 |
|----|--------|------|------|----------|
| CASE-NFR-01 | P0 | 短篇耗时 | test-book.epub，CPU | ≤ 20 分钟完成 |
| CASE-NFR-02 | P1 | 内存稳定 | 转换过程中观察容器内存 | 无 OOM Kill（或 < 容器 limit） |
| CASE-NFR-03 | P0 | ffmpeg 编码器 | 容器内运行 `ffmpeg -hide_banner -encoders | grep -E '(^| )A.*aac'` | 存在原生 `aac` 编码器 |
| CASE-NFR-04 | P2 | ffmpeg 缺失降级 | 镜像故意无 ffmpeg（仅 dev 测） | 仍产出 WAV，UI 提示无 M4B |
| CASE-NFR-05 | P2 | GPU 加速 | `AUDIBLEZ_USE_CUDA=1` + GPU 权限 | 速度显著优于 CPU |

---

## 5. 回归矩阵（与上游 audiblez 对齐）

| 上游能力 | 微服用例 | 说明 |
|----------|----------|------|
| CLI `audiblez book.epub -v af_sky` | CASE-JOB-01 | Web 等价 |
| CLI `-p` 选章 | CASE-WEB-06 | |
| CLI `-s` 语速 | CASE-WEB-02 | |
| CLI `-c` CUDA | CASE-NFR-05 | |
| CLI `-o` 输出目录 | CASE-PERSIST-01 | 映射到 `/lzcapp/var/jobs` |
| `audiblez-ui` GUI | — | **不移植** |
| 无 ffmpeg 警告 | CASE-NFR-04 | core 已有逻辑 |

---

## 6. 自动化测试建议

```text
audiblez-lpk/tests/
├── test_paths.py          # 路径规范化、穿越防护
├── test_job_manager.py    # 状态机、meta 读写
├── test_api.py            # FastAPI TestClient
└── integration/
    └── test_core_smoke.py # 容器内短文本 TTS（CI 可选，耗时长）
```

**CI 策略：**

- PR 门禁：单元测试 + `docker build` 语法检查（不跑完整 TTS）
- 夜间/手动：CASE-JOB-01 在微服或本地 docker 跑通

---

## 7. 验收清单（MVP 发布门槛）

MVP（v0.1.0）上线前 **必须全绿**：

- [ ] CASE-DEPLOY-01～06
- [ ] CASE-WEB-01、03、07
- [ ] CASE-JOB-01～05
- [ ] CASE-PERSIST-01
- [ ] CASE-SEC-01、02
- [ ] CASE-NFR-01、03

P1 迭代门槛：

- [ ] CASE-DEPLOY-08（icon + locales）
- [ ] CASE-FH-01～04（EPUB 网盘右键打开）
- [ ] CASE-FH-05～07（M4B 右键打开与播放）
- [ ] CASE-NETDISK-01～04
- [ ] CASE-WEB-06
- [ ] CASE-JOB-06

---

## 8. 缺陷分级

| 级别 | 定义 | 示例 |
|------|------|------|
| Blocker | 无法安装或无法打开 | build 失败、首页 502 |
| Critical | 核心路径失败 | 合法 EPUB 无法出 M4B |
| Major | 功能缺失但有绕行 | 无网盘只能上传 |
| Minor | 体验问题 | 进度条不刷新 |
| Trivial | 文案/样式 | 按钮错别字 |

---

## 9. 测试记录模板

```markdown
## 运行记录 YYYY-MM-DD

| 用例 ID | 结果 | 执行人 | 备注 |
|---------|------|--------|------|
| CASE-JOB-01 | PASS | | test-book.epub, 12min CPU |
| CASE-DEPLOY-03 | PASS | | lpk 4.2GB |
```

---

## 10. 附录：手动验收脚本（P0 冒烟）

```bash
# 在 audiblez-lpk/lzc 目录
lzc-cli project build
LPK=$(ls -1 *.lpk | head -1)
lzc-cli lpk install "./$LPK"

# 上传测试（需替换 URL 与 cookie，或浏览器手工）
# curl -F "file=@test-book.epub" -F "voice=af_sky" \
#   "https://audiblez.your-box-name.heiyu.space/api/jobs"

lzc-cli project log -f
```

浏览器手工完成 CASE-JOB-01 后，在本文档测试记录区登记结果。
