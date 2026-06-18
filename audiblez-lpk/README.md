# Audiblez 微服 LPK

将 [audiblez](https://github.com/santinic/audiblez) 封装为懒猫微服 LPK 应用（FastAPI Web + Docker + 分阶段网盘集成）。

## 文档

| 文档 | 说明 |
|------|------|
| [docs/PRD.md](docs/PRD.md) | 产品需求 |
| [docs/ARCH.md](docs/ARCH.md) | 架构设计 |
| [docs/CASES.md](docs/CASES.md) | 用例与验收 |

## 状态

- [x] M0 文档（PRD / ARCH / CASES）
- [ ] M1 骨架（Dockerfile + FastAPI 空 Web + 三件套）
- [ ] M2 核心转换（上传 EPUB → M4B 下载）
- [ ] M3 网盘集成（file_handler 右键 EPUB + inject + 网盘读写）
- [ ] M4 build + install

## 已拍板决策

- Web 框架：FastAPI + 简单 HTML/HTMX
- 包名：`cloud.lazycat.app.audiblez`，dev 包名 `cloud.lazycat.app.audiblez.dev`
- MVP 边界：M2 只做上传转书；M3 再做网盘右键、inject 与写回
- 模型策略：Kokoro / spaCy 构建时 bake，运行时默认离线
- ffmpeg：使用 Debian 默认支持的原生 `aac` 编码器

## 开发

```bash
cd lzc
lzc-cli project deploy   # 开发迭代（待 M1）
lzc-cli project build    # 发布包（待 M4）
```

## 与 monorepo 关系

- 上游 Python 源码：`../audiblez/`（audiblez 包）
- 本目录：LPK 包装层，符合 `AGENTS.md` 不与 `skills/` 混放构建文件
