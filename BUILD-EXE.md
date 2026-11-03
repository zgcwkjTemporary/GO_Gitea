# Gitea Windows EXE 编译脚本

留存自 2026-11-03，用于在 Codespaces / Linux 环境交叉编译 Gitea Windows amd64 可执行文件。

## 快速使用

```bash
# 方式一：使用 Makefile 目标（完整 release 流程）
make release-windows

# 方式二：直接调用构建脚本（推荐，产物在 dist/ 目录）
RELEASE_VERSION=$(git describe --tags --always) ./tools/build-release.sh windows
```

## 产物说明

| 文件 | 说明 |
| --- | --- |
| `dist/release/gitea-*-windows-amd64.xz` | 构建脚本产出的压缩包 |
| `gitea-windows-amd64.zip`（约 46MB） | 实际留存产物，内含单个 `gitea.exe` |

## 生成 zip + sha256（本次使用的方式）

```bash
# 1. 编译 exe（官方构建脚本关键逻辑：CGO_ENABLED=0 + bindata，生成静态二进制）
VERSION=$(git describe --tags --always)
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 \
  go build -tags "bindata $TAGS" \
  -ldflags "-s -w -X main.version=$VERSION -X 'main.Tags=bindata $TAGS'" \
  -o gitea.exe .

# 2. 打包 zip
zip -j gitea-windows-amd64.zip gitea.exe

# 3. 生成校验文件
sha256sum gitea-windows-amd64.zip > gitea-windows-amd64.sha256
```

## 源码变更包（gitea-go1269-source-changes.zip）

```bash
# 将本地修改的源码打包留存（不含未跟踪文件）
git archive --format=zip -o gitea-go1269-source-changes.zip $(git stash create)
# 或简单方式：打包当前 diff 涉及的文件
git diff --name-only | zip gitea-go1269-source-changes.zip -@
sha256sum gitea-go1269-source-changes.zip > gitea-go1269-source-changes.sha256
```

## 注意事项

- 前端资源需要先 `make frontend`（或 `make generate-bindata`），否则 exe 缺少模板/静态资源
- `TAGS` 默认含 `bindata`（内嵌前端资源）；官方脚本使用 `CGO_ENABLED=0` 生成静态二进制（不带 sqlite）
- 需要内置 SQLite 时请去掉 `CGO_ENABLED=0` 并加 `TAGS="sqlite sqlite_unlock_notify"`
- 构建脚本 `tools/build-release.sh` 需要指定平台参数：`windows` / `linux` / `darwin` / `freebsd`
