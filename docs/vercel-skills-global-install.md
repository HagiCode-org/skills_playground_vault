# 自有 GitHub Skills 仓方案

## 目标

建立一个**由自己控制的单一 skills 源**，满足以下效果：

1. 在 GitHub 维护一个自有聚合仓。
2. 该仓通过 GitHub Actions 同步你纳入管理的 skills。
3. 该仓本身保持为标准 skills 仓，可被 `skills cli` 直接识别。
4. 任意机器只需执行一次安装命令，即可全局安装全部技能。
5. 后续更新由你自己的仓统一控制，而不是依赖零散上游源。

## 核心结论

- 可行。
- 最佳实践是：**GitHub 仓才是真源，本地 clone 只是工作副本。**
- 安装时应优先使用 GitHub 源，而不是本地路径源。
- 原因是：
  - 本地路径可安装。
  - 但 `skills update` 对 `local path` 不做完整自动追踪。
  - GitHub 源才能较好复用 `skills` 的全局锁文件与更新机制。

故推荐链路为：

```text
upstream skills -> your GitHub aggregate repo -> npx skills add yourorg/your-skills-repo -g --all
```

而不是：

```text
upstream skills -> local clone -> npx skills add /local/path -g --all
```

后者能安装，然不宜作为长期更新追踪主路径。

## 你的仓库应如何设计

建议将自有仓设计为标准 skills 仓：

```text
your-skills-repo/
  skills/
    skill-a/
      SKILL.md
    skill-b/
      SKILL.md
    skill-c/
      SKILL.md
```

若有自写 skill，可继续放在同一仓内。

推荐再增加一份来源清单，例如：

```text
your-skills-repo/
  skills/
  sources.yaml
  .github/workflows/sync-skills.yml
```

## `skills cli` 在此方案中的职责

`skills cli` 适合做：

- 初始化 skill 骨架
- 校验你的仓是否能被识别为 skills 仓
- 安装全部 skill
- 全局安装
- 列出已安装 skill
- 执行后续更新

`skills cli` 不适合做：

- 从多个上游 repo 抓取技能
- 解决命名冲突
- 去重与重命名
- 生成聚合 PR
- 决定保留哪个上游版本

这些应由 GitHub Action 或你自己的同步脚本负责。

## 推荐工作流

### 一、维护端

你在 GitHub 上维护一个聚合仓。

建议流程：

1. 用 `sources.yaml` 声明要纳管的上游 skills 来源。
2. GitHub Action 定时或手动运行同步脚本。
3. 同步脚本将上游 skills 归档到本仓 `skills/` 下。
4. 若发现冲突，Action 失败或提交 PR 供你审查。
5. 合并后，这个聚合仓即成为新的唯一可安装源。

### 二、消费端

任意机器上，直接安装你的聚合仓：

```bash
npx skills add yourorg/your-skills-repo -g --all
```

后续更新：

```bash
npx skills update
```

查看全局已装技能：

```bash
npx skills list -g
```

## 本地 clone 的正确用途

本地 clone 有用，但用途应限定为：

- 审查聚合后的内容
- 本地调试 Action 产物
- 人工编辑和提交
- 本地验证仓是否仍可被 `skills cli` 发现

例如：

```bash
git clone git@github.com:yourorg/your-skills-repo.git
cd your-skills-repo
npx skills add . --list
```

或在测试项目中验证：

```bash
cd /tmp/test-repo
npx skills add /path/to/your-skills-repo --all -y
```

但**正式全局安装**仍建议使用 GitHub 源：

```bash
npx skills add yourorg/your-skills-repo -g --all
```

## 为什么不建议把本地 clone 当正式更新源

因为 `skills update` 对 `local path` 的自动追踪能力有限。

简化理解：

- `npx skills add /local/path -g --all`
  - 可安装
  - 但后续自动更新不稳定，不宜作为长期主路径

- `npx skills add yourorg/your-skills-repo -g --all`
  - 可安装
  - 后续 `npx skills update` 才更符合预期

所以，结论不是“不能 clone 到本地”，而是：

- **clone 可以**
- **正式 install/update 源应尽量保持为 GitHub 仓**

## 版本控制建议

若你希望“换电脑也能完全一致”，建议对聚合仓打 tag。

例如：

```bash
npx skills add yourorg/your-skills-repo#v2026.04 -g --all
```

这样有两个好处：

- 新机器安装时可复现同一批技能版本
- 回滚容易

推荐两种模式：

### 滚动模式

适合你自己日常使用：

```bash
npx skills add yourorg/your-skills-repo -g --all
npx skills update
```

### 发布模式

适合跨机器一致性或团队统一版本：

```bash
npx skills add yourorg/your-skills-repo#v2026.04 -g --all
```

后续升级通过新 tag 进行，而不是直接追主分支最新内容。

## 如何用 `skills cli` 维护这个自有仓

### 1. 初始化自写 skill

若你要在聚合仓中自己新增 skill：

```bash
cd your-skills-repo
npx skills init skills/my-skill
```

然后补充 `SKILL.md` 内容。

### 2. 检查聚合仓是否能被正确识别

在仓库根目录执行：

```bash
npx skills add . --list
```

这可验证：

- skills 目录结构是否正确
- `SKILL.md` frontmatter 是否可识别
- 聚合结果是否完整暴露

### 3. 本地 smoke test

若要本地测试安装效果：

```bash
npx skills add . --all -y
```

更稳妥的做法是在临时测试 repo 中执行，避免污染当前工作目录。

### 4. 正式发布后安装

从 GitHub 源正式安装：

```bash
npx skills add yourorg/your-skills-repo -g --all
```

### 5. 后续更新

```bash
npx skills update
```

### 6. 安装结果审计

```bash
npx skills list -g
```

## GitHub Action 应负责什么

建议 Action 负责以下职责：

1. 读取 `sources.yaml`
2. 拉取上游 repo 或 skill 子路径
3. 将技能写入本仓 `skills/<skill-name>/`
4. 处理重复命名
5. 校验每个 skill 都有合法 `SKILL.md`
6. 运行一次 `npx skills add . --list` 或等价校验
7. 自动提交变更或发起 PR

## 命名冲突策略

聚合仓早晚会碰到 skill 名冲突。

建议明确规则：

- 优先保留你自己的自写 skill
- 上游 skill 冲突时加命名空间前缀
- 或在同步阶段统一改名

示例：

```text
skills/
  openai-prompt-upgrade/
  vercel-react-best-practices/
  internal-release-notes/
```

不要依赖上游原名天然永不冲突。逻辑有问题。

## 实施建议

最稳的最终方案如下：

### 方案主体

1. 建立 `your-skills-repo`
2. 用 GitHub Action 聚合你纳管的 skills
3. 保持该仓本身就是标准 skills 仓
4. 客户端统一安装这个仓

### 客户端命令

首次安装：

```bash
npx skills add yourorg/your-skills-repo -g --all
```

后续更新：

```bash
npx skills update
```

换新电脑：

```bash
npx skills add yourorg/your-skills-repo -g --all
```

或安装某个固定发布版本：

```bash
npx skills add yourorg/your-skills-repo#v2026.04 -g --all
```

## 最终判断

- 你这个思路可行。
- 你的 GitHub 聚合仓应被视为**唯一真源**。
- 本地 clone 应被视为**工作副本**，不是正式更新源。
- `skills cli` 可复用来做初始化、校验、安装、更新。
- 上游聚合、冲突处理、同步编排仍需你自己的 GitHub Action 或脚本实现。

## 当前调研上下文

- 调研对象仓库：`vercel-labs/skills`
- 本地落地路径：`/home/newbe36524/repos/newbe36524/hagicode-mono/repos/local_publishment/.local-publishment/data/codeRefs/skills_playground_vault/repos/vercel-labs-skills`
- 已确认：该仓是 skills CLI 本体仓，不是完整 skills 集合仓

## 相关模板与规范

若要继续落地，请配合以下文档使用：

- `docs/skills-aggregate-repo-spec.md`
- `docs/templates/sources.yaml.example`
- `docs/templates/sync-skills.workflow.yml.example`
- `docs/templates/sync-skills.mjs.example`
