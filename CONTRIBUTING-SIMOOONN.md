# Simooonn 低耦合开发规范

本文档约定 Simooonn fork 的分支使用方式和定制开发边界，目标是让定制能力与官方代码保持低耦合，尽量减少合并上游更新时的冲突。

定制开发只在 fork 的开发 checkout 中进行，不要把定制代码写入运行用的官方 checkout。

## 1. 分支规范

| 分支 | 用途 | 规则 |
| --- | --- | --- |
| `main` | 跟踪上游 `NousResearch/hermes-agent:main` | 官方代码镜像，只允许 `simooonn-upstream-watch.yml` 一个例外文件 |
| `release` | 定制代码的基础分支和生产发布分支 | 所有定制改动最终汇入此分支 |
| `feature/xxx` | 新功能开发 | 从最新 `release` 切出 |
| `fix/xxx` | 问题修复 | 从最新 `release` 切出 |

个人功能分支使用小写、简短且能表达用途的名称，例如 `feature/simooonn-upstream-watch` 或 `fix/simooonn-adapter-timeout`。

### `main` 上唯一的例外：上游监控 workflow

`.github/workflows/simooonn-upstream-watch.yml` 必须同时存在于 `main`：GitHub Actions 的 `schedule` 触发器只读取仓库默认分支（本仓库默认分支是 `main`），只放在 `release` 上永远不会定时运行。

因此：

- 该文件是 `main` 上唯一的 Simooonn 自有文件，其他任何定制代码一律不得进入 `main`。
- `main` 与 `release` 上的这份文件内容必须保持一致；改动时两边都要更新，否则 `git merge origin/main` 到 `release` 时会冲突。
- 该文件只应有 `schedule` 和 `workflow_dispatch` 两个触发器，不引入任何需要 fork 侧秘密以外前提的逻辑，保证它能长期留在 `main` 上而很少需要改动。

### 合并上游

- `origin` 指向 Simooonn fork，`upstream` 指向官方仓库。
- 合并前确保 `origin/main` 已与上游 `main` 同步。
- 在 `release` 上执行 `git merge origin/main`，不要使用 rebase，以保留分支和合并历史。
- 除上游监控 workflow 外，不要把 `release` 的定制提交合并或推送到 `main`。

```bash
git fetch upstream
git switch main
# main 带有 workflow 例外文件，无法快进合并，必须产生一个合并提交
git merge upstream/main -m "chore: merge upstream main"
git push origin main
git switch release
git merge origin/main
```

`main` 上出现合并提交是预期结果，由 workflow 例外文件导致。除该文件外，`main` 不应有任何本地改动；可通过下面的命令确认：

```bash
git diff --name-only upstream/main main   # 只应输出 .github/workflows/simooonn-upstream-watch.yml
```

## 2. 双 checkout 安全边界

本开发流程涉及两个完全独立的 checkout：

- 开发用 fork 仓库：`/Users/wmm/wmm-code/GitHub-Code/Simooonn/hermes-agent`。
- 运行用官方 partial clone：`~/.hermes/hermes-agent`。

两者有各自独立的工作树、分支和远端配置。开始开发或同步前，分别核对远端：

```bash
git remote -v
git -C ~/.hermes/hermes-agent remote -v
```

禁止把定制代码直接写进运行 checkout。所有定制改动只在开发用 fork 仓库中完成，并按分支规范提交。

fork/release 的改动不会自动部署到实际运行环境，需要手动同步或通过 `hermes update` 机制更新。不要把开发仓库的提交、推送或合并操作视为运行环境已经更新。

## 3. 低耦合开发原则

### 新增功能优先使用独立扩展

- 优先新建独立文件，不修改官方现有文件。
- 新 skill 放在 `skills/` 下的独立子目录中。
- 新 plugin 放在 `plugins/platforms/` 下的独立子目录中；需要独立发布或维护时，优先使用单独 plugin repo。
- 新 workflow 文件使用 `simooonn-` 前缀，例如 `.github/workflows/simooonn-upstream-watch.yml`。
- 新文档放在 `docs/simooonn/` 或 `website/docs/simooonn/`。
- 优先使用项目已有的配置、CLI、skill 和 plugin 扩展机制，不为单个定制需求扩大核心代码修改面。

### 必须修改官方文件时

修改官方文件属于高风险操作。确认无法通过独立扩展实现后，再按以下约束执行：

1. 将文件路径、修改类型、耦合原因和冲突风险记录到本文档的“已知耦合点”章节。
2. 修改尽量集中在入口点，例如 `__init__.py` 或 `config.py`，不要深入改动官方内部实现。
3. 每处定制修改都添加 `# SIMOOONN: <English explanation>` 注释，便于合并时搜索和识别；非 `#` 注释语法的文件使用对应注释符，但保留 `SIMOOONN` 标记。
4. 注释使用英文，简要说明定制目的，以及为什么不能通过独立扩展实现。
5. PR 描述中明确列出所有受影响的上游文件，并说明合并上游时需要重点检查的位置。
6. 能解决同一问题时，优先把定制逻辑提取到独立文件或函数，官方文件中只保留最小入口调用。

### 合并上游前 checklist

1. 获取上游提交并查看待合并数量：

   ```bash
   git fetch upstream && git log --oneline release..upstream/main
   ```

2. 对照“已知耦合点”，检查新提交是否涉及已耦合的官方文件：

   ```bash
   git diff --name-only release...upstream/main -- <受影响文件>
   ```

   如果需要查看具体改动：

   ```bash
   git diff release...upstream/main -- <受影响文件>
   ```

3. 解决冲突时优先保留上游逻辑，把定制内容提取到独立文件或函数，避免在冲突区域继续叠加实现细节。
4. 合并后运行完整检查：

   ```bash
   python scripts/check
   ```

5. 检查失败时先确认是否由本次合并或定制改动引入；修复后再将结果用于生产发布。

## 4. 已知耦合点

只记录对官方文件的定制修改。新增耦合点时同步更新此表，移除耦合点后再删除对应记录。

| 文件路径 | 修改类型（新增/修改） | 耦合原因 | 冲突风险（高/中/低） |
| --- | --- | --- | --- |
| 暂无 | — | 当前无已知耦合点（零定制） | — |

## 5. 推荐的低耦合模式

### Skill 扩展（最低风险）

```text
skills/<名称>/SKILL.md
```

将说明、脚本和辅助资源放在同一独立目录中，不为单个 skill 修改核心代码。

### Plugin 扩展（低风险）

```text
plugins/platforms/<名称>/adapter.py
```

通过项目公开的 plugin API 和既有适配器约定接入；需要独立维护时使用单独 plugin repo。

### 配置扩展（低风险）

```text
~/.hermes/config.yaml  # 用户配置，不入仓库
```

用户级行为优先通过配置实现，不把个人环境配置或密钥提交到仓库。

### 核心代码钩子（高风险，尽量避免）

如必须修改 `gateway/`、`hermes_cli/`、`tools/` 等核心区域，只在入口位置插入最小调用，不改内部逻辑；定制实现放在独立文件中，并登记为已知耦合点。

## 6. 文件命名约定

- 所有 Simooonn 独有文件使用 `simooonn-` 前缀，或放在 `simooonn/` 子目录中。
- 根目录规范文档 `CONTRIBUTING-SIMOOONN.md` 是固定例外，不要求添加 `simooonn-` 前缀。
- skill、plugin 等必须使用固定目录结构的扩展，将其独立目录命名为 `simooonn-<名称>`，内部文件遵循上游扩展约定。
- workflow 统一命名为 `.github/workflows/simooonn-<用途>.yml`。
- 文档统一放在 `docs/simooonn/` 或 `website/docs/simooonn/`。
- 代码中的定制标记统一使用大写 `SIMOOONN`，便于全文搜索。

使用以下命令快速检查命名范围内的定制内容：

```bash
git diff main release -- '*simooonn*'
```

命名过滤不能替代完整审查；合并或发布前仍需检查“已知耦合点”中列出的官方文件。
