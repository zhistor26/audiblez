# 🐱 Lazycat Skills (懒猫微服 AI 技能包)

这是一套专为 **懒猫微服 (LazyCat MicroServer)** 平台开发者打造的 AI 智能体技能包 (Agent Skills)。

> [!IMPORTANT]
> **分支与版本说明：**
> - **`main` / `v2` 分支 (当前)**: 支持最新的 **LPK V2 (v1.5.0+)** 规范（元数据与运行结构分离，支持 `package.yml` 和 `permissions` 声明）。
> - **`v1` 分支**: 包含旧版的 LPK V1 规范。

通过安装本技能包，AI 助手（Cursor、Windsurf、Cline 等）可自动编写 `package.yml`、`lzc-manifest.yml`、打包 LPK、处理高级路由、网盘集成与微服认证开发。

## 📦 技能列表

### 总控入口

| Skill | 说明 |
| --- | --- |
| `lazycat-developer-expert` | 总控路由：按需求分发到下方各垂直 skill |

### 移植与手册（社区增强）

| Skill | 说明 |
| --- | --- |
| `lazycat-porting` | **Docker 移植三阶段流水线**：配置 → `build` + `lpk install` 验证 → 提审；含 `porting-handbook.md`、`publish-checklist.md` |
| `lazycat-library` | **59 篇开发者手册**按需检索（需克隆 [lazycat-developer-manual](https://github.com/lazycat-cloud/lazycat-developer-manual) 或设 `LAZYCAT_MANUAL_ROOT`） |
| `lazycat-lpk-netdisk` | 懒猫网盘打开/保存、COEP/`ffmpeg.wasm` 桥接方案 |

### 垂直领域（上游核心）

| Skill | 说明 |
| --- | --- |
| `lazycat-lpk-builder` | Docker/源码 → `.lpk` 打包；自包含 YAML 规范（`references/`） |
| `lazycat-dynamic-deploy` | `lzc-deploy-params.yml`、Go 模板渲染、`application.injects` |
| `lazycat-advanced-routing` | 多域名、ingress 四层转发、upstreams、app-proxy |
| `lazycat-auth-integration` | OIDC 免密、HTTP Header 身份、`public_path`、API Auth Token |
| `lazycat-aipod-developer` | AI 算力舱应用（Traefik、GPU、`-ai` 域名） |

## 🔀 推荐分工

```
移植任务     → lazycat-porting（流水线 + checklist）
字段规范     → lazycat-lpk-builder（references/）
深度专题     → lazycat-library（meta/index.json → 单篇 .md）
网盘/COEP    → lazycat-lpk-netdisk
路由/OIDC/…  → 对应垂直 skill
不确定从哪开始 → lazycat-developer-expert
```

## 🚀 安装

```bash
# 推荐：安装本 fork（含社区增强 skill）
npx skills add zhistor26/lazycat-skills

# 或上游原版
npx skills add whoamihappyhacking/lazycat-skills
```

使用 `lazycat-library` 时，额外克隆开发者手册并（可选）设置环境变量：

```bash
git clone https://github.com/lazycat-cloud/lazycat-developer-manual.git
# LAZYCAT_MANUAL_ROOT=/path/to/lazycat-developer-manual/lazycat-developer-manual
```

安装后可以对 AI 说：**「帮我把当前的 Docker 项目打包成懒猫 lpk 并本地 install 验证」**。

## 📂 项目结构

```text
lazycat-skills/
├── AGENTS.md
├── README.md
└── skills/
    ├── lazycat-developer-expert/   # 总控
    ├── lazycat-porting/            # 移植流水线（社区）
    ├── lazycat-library/            # 手册检索（社区）
    ├── lazycat-lpk-netdisk/        # 网盘集成（社区）
    ├── lazycat-lpk-builder/
    ├── lazycat-dynamic-deploy/
    ├── lazycat-advanced-routing/
    ├── lazycat-auth-integration/
    └── lazycat-aipod-developer/
```

## 🤝 贡献

本 fork（`zhistor26/lazycat-skills`）在 upstream 基础上合并了社区移植流水线、完整手册检索协议与网盘集成 skill。欢迎 PR。
