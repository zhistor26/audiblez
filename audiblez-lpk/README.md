# Audiblez 微服 LPK

将 [audiblez](https://github.com/santinic/audiblez) 封装为懒猫微服 LPK 应用（Web + Docker + 网盘）。

## 文档

| 文档 | 说明 |
|------|------|
| [docs/PRD.md](docs/PRD.md) | 产品需求 |
| [docs/ARCH.md](docs/ARCH.md) | 架构设计 |
| [docs/CASES.md](docs/CASES.md) | 用例与验收 |

## 状态

- [x] M0 文档（PRD / ARCH / CASES）
- [ ] M1 骨架（Dockerfile + Web + 三件套）
- [ ] M2 核心转换
- [ ] M3 网盘集成
- [ ] M4 build + install

## 开发

```bash
cd lzc
lzc-cli project deploy   # 开发迭代（待 M1）
lzc-cli project build    # 发布包（待 M4）
```

## 与 monorepo 关系

- 上游 Python 源码：`../audiblez/`（audiblez 包）
- 本目录：LPK 包装层，符合 `AGENTS.md` 不与 `skills/` 混放构建文件
