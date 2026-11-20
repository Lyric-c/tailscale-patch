# Tailscale 自定义补丁

此目录对应一个独立的补丁仓库，用于保存从 Tailscale 定制分支导出的代码补丁、补丁说明和构建工作流。GitHub Actions 也在这个补丁仓库中运行：它检出指定的 Tailscale 上游版本、应用本仓库保存的补丁、验证并编译，而不需要从 Tailscale fork 的分支历史中推断要 cherry-pick 哪些提交。

> 当前目录包含初始代码补丁和一个手动触发的 GitHub Actions 工作流。更新上游或定制功能后，请重新生成并验证补丁，再提交更新。

## 当前内容

| 文件 | 用途 |
| --- | --- |
| `patches/0001-force-derp.patch` | Force DERP 特性及测试 |
| `patches/0002-default-no-logs-no-support.patch` | 默认禁用日志上传 |
| `patches/0003-show-tailscaled-subcommands.patch` | 显示 `tailscaled` 子命令及 debug 帮助 |
| `patches/0004-system-service-install-and-args.patch` | Linux systemd、安装参数透传及平台服务安装调整 |
| `BASE_COMMIT` | 生成补丁时使用的上游共同基线 commit |
| `SOURCE_COMMIT` | 生成补丁时定制分支的 commit |
| `README.md` | 特性概览、应用方法和维护流程 |
| `.github/workflows/release-custom.yml` | 在本补丁仓库中运行的构建与发布 Action |

该补丁仅包含 `cmd/tailscale/cli`、`cmd/tailscaled`、`ipn/ipnlocal`、`ipn/localapi` 和 `wgengine/magicsock` 下的代码与测试改动。它不包含 README、GitHub Actions workflow 或其他流程文档；补丁仓库自己的说明文档也不会被应用到 Tailscale 源码树。

## 当前补丁包含的特性

- **强制 DERP 中继（`force-derp`）**：新增 `tailscale debug force-derp` 命令，通过本地 API 更新节点行为，控制是否强制经 DERP 中继。具体参数及行为以补丁代码和对应测试为准。
- **默认禁用日志上传与支持数据上传**：`tailscaled` 默认启用 `-no-logs-no-support`，可显式传入 `false` 覆盖。
- **在帮助中列出守护进程子命令**：让 `tailscaled` 帮助信息显示 `debug`、`install-system-daemon` 和 `uninstall-system-daemon` 等子命令。
- **安装服务时支持传递启动参数**：相关平台的安装逻辑可接收并持久化服务启动参数。
- **Linux systemd 服务支持**：新增 Linux 上安装、启动、卸载 systemd 服务的实现及测试。

补丁当前不包含独立 DERP 构建流水线；该流水线属于构建/发布配置，由本仓库的 Action 独立构建 `cmd/derper`。

## 运行构建 Action

`.github/workflows/release-custom.yml` 定义了本仓库的构建与发布流程。提交补丁后，在 GitHub 仓库的 **Actions → Build Custom Tailscale → Run workflow** 手动运行，并输入上游版本 tag，例如 `v1.104.4`。如果希望发布到带自定义标记的 Release，可输入 `v1.104.4-custom`；Action 会从上游检出 `v1.104.4`，但以 `v1.104.4-custom` 命名产物和 Release。

Action 会在各个构建 job 中检出干净的 Tailscale 上游 tag，按文件名顺序依次应用 `patches/*.patch`，并先运行定向 Go 测试，再并行构建：

- `tailscale` 和 `tailscaled`：Linux amd64/arm64、macOS arm64、Windows amd64。
- `derper`：Linux amd64。
- 所有产物的 SHA-256 校验和：`sha256sums.txt`。

构建完成后，产物会发布到**补丁仓库自身**的 GitHub Releases。工作流需要仓库 Actions 设置允许 `GITHUB_TOKEN` 写入 contents/releases。修改工作流时，请检查 `.github/workflows/release-custom.yml`；更新补丁时，Action 会自动使用本仓库 checkout 得到的新补丁。

也可以在本地手动应用并测试补丁。先检出目标 Tailscale 源码版本，再在源码树根目录运行：

```bash
for patch in /path/to/tailscale-patch/patches/*.patch; do
  git apply --check "$patch"
  git apply "$patch"
done
```

然后运行相关测试和目标平台构建。至少建议验证：

```bash
go test ./cmd/tailscale/...
go test ./cmd/tailscaled/...
go test ./ipn/localapi/...
go test ./wgengine/magicsock/...
```

`git apply --check` 只检查补丁能否干净应用，不代表代码在新上游版本上仍然正确；补丁应用成功后仍需测试。遇到冲突时，应针对新上游版本更新定制改动并重新生成补丁，不要通过忽略错误继续发布。

## 补丁来源与再生成

本次补丁的来源信息记录在 `BASE_COMMIT` 和 `SOURCE_COMMIT` 中。补丁是从基线到定制分支指定代码目录的**汇总 diff**，不是逐个提交的 `format-patch` 系列。

在定制仓库中更新时，先选定明确的共同基线和已审查的定制分支 commit，再按 feature 涉及的文件分别导出补丁。例如：

```bash
BASE=<上游共同基线 commit>
SOURCE=<已审查的定制分支 commit>
PATCH_DIR=../tailscale-patch
mkdir -p "$PATCH_DIR/patches"

git diff --binary --full-index "$BASE" "$SOURCE" -- <该 feature 的文件列表> \
  > "$PATCH_DIR/patches/0001-<feature-name>.patch"
# 其他 feature 分别导出为 0002-、0003- 等补丁
printf '%s\n' "$BASE" > "$PATCH_DIR/BASE_COMMIT"
printf '%s\n' "$SOURCE" > "$PATCH_DIR/SOURCE_COMMIT"
```

不要让不同补丁包含同一处改动；若 feature 有依赖关系，按依赖顺序编号。生成后，应在 `BASE` 对应的干净工作树中按文件名顺序逐个运行 `git apply --check` 并应用，检查代码、测试和跨平台编译。新增自定义 feature 时，也要相应更新补丁文件和本说明。

## 维护原则

1. 定制仓库是功能开发和审查的来源；补丁仓库保存已审查、用于构建的交付物，避免两边分别手工修改功能实现。
2. 每个 feature 对应一个编号补丁，按文件名顺序应用；如 feature 之间存在依赖，需保持依赖顺序并在本说明中记录。每次发布记录目标 Tailscale 上游 tag/commit、补丁来源 commit 和构建结果。基线文件记录的是生成补丁时的共同基线，不一定等于每次实际构建的目标 tag。
3. 上游升级后，先在新版本上应用补丁并测试；若需修改，回到定制开发源修复并重新导出，不要只在 Action 临时改源码。
4. 构建 Action 放在本补丁仓库的 `.github/workflows/` 下运行，应固定并校验目标上游版本，依次检出上游源码、应用本仓库中的补丁、执行测试/构建和发布。
5. 补丁更新应一并更新本说明和来源 commit，便于追溯。
