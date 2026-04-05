# Skills 聚合仓规范

## 文档类型

Reference + How-to。

## 适用对象

- 维护自有 skills 聚合仓的开发者
- 需要用 GitHub Actions 统一同步、发布与安装 skills 的团队

## 目标

定义一个可长期维护的聚合仓规范，使之满足：

1. 可从多个上游 repo 收集 skills。
2. 聚合后仓库自身仍是标准 skills 仓。
3. 本机或新机器都能用同一命令全局安装。
4. 命名冲突、删除清理、版本发布均有明确规则。

## 仓库目录规范

推荐目录：

```text
your-skills-repo/
  .github/
    workflows/
      sync-skills.yml
  docs/
  scripts/
    sync-skills.mjs
  skills/
    internal-my-skill/
      SKILL.md
    vercel-frontend-design/
      SKILL.md
  sources.yaml
```

各目录职责：

- `skills/`
  - 最终对外暴露的技能目录
  - `skills cli` 直接从这里发现 skill
- `sources.yaml`
  - 上游源声明
  - 是聚合动作的唯一配置入口
- `scripts/sync-skills.mjs`
  - 读取 `sources.yaml`
  - 拉取上游源
  - 复制或覆盖聚合结果到 `skills/`
- `.github/workflows/sync-skills.yml`
  - 定时或手动触发同步
- `docs/`
  - 规则、模板、变更说明

## `sources.yaml` 规范

### 顶层字段

- `schemaVersion`
  - 当前建议值：`1`
- `defaults`
  - 全局默认值
- `sources`
  - 上游源数组

### `defaults` 字段

- `ref`
  - 默认分支或 tag
- `targetRoot`
  - 默认写入目录，建议固定为 `skills`
- `naming`
  - 命名策略，建议固定为 `namespace-prefix`

### `sources[]` 字段

- `id`
  - 源唯一标识
  - 建议只用小写字母、数字、短横线
- `enabled`
  - 是否启用
- `repo`
  - Git URL 或 GitHub HTTPS URL
- `ref`
  - 源分支、tag 或 commit
- `namespace`
  - 聚合后命名前缀
- `discover`
  - 要扫描的相对目录列表
  - 常见值：`skills`
- `include`
  - 允许同步的 skill 名列表
  - 空数组表示全部
- `exclude`
  - 要排除的 skill 名列表
  - 优先级高于 `include`

## 命名规范

重点是：**外部同步 skill 一律加命名空间前缀。**

推荐规则：

- 外部 skill：`<namespace>-<skill-name>`
- 自写内部 skill：`internal-<skill-name>`

示例：

- `vercel-frontend-design`
- `openai-prompt-upgrade`
- `internal-release-notes`

这样做的原因：

1. 避免不同上游同名冲突。
2. 一眼可见来源。
3. 后续清理与归档更容易。

勿依赖“上游永不重名”。逻辑有问题。

## 来源标记规范

所有由同步脚本生成的 skill 目录，建议在目录内写入：

```text
.skill-source.json
```

建议字段：

```json
{
  "sourceId": "vercel",
  "repo": "https://github.com/vercel-labs/agent-skills.git",
  "ref": "main",
  "originalName": "frontend-design",
  "targetName": "vercel-frontend-design"
}
```

用途：

- 标识该目录由哪个 source 管理
- 清理 stale skill
- 审计同步来源

此文件以 `.` 开头，安装到 agent 时通常不会被复制进入最终 skill 目录，适合作为仓内元数据。

## 同步规则

同步脚本应遵守以下规则：

1. 只修改带 `.skill-source.json` 的受管目录。
2. 不触碰手工维护的 skill 目录。
3. 若上游 skill 已删除，则删除对应受管目录。
4. 若上游 `SKILL.md` 缺少 `name` 或 `description`，则跳过并报错。
5. 若命名冲突，直接失败，不做静默覆盖。

## 验证规则

每次同步后，至少做以下检查：

1. `skills/` 下每个 skill 目录均含 `SKILL.md`
2. `SKILL.md` frontmatter 至少有：
   - `name`
   - `description`
3. 仓库根目录执行：

```bash
npx skills add . --list
```

若此命令失败，则本次聚合结果不得发布。

## 安装规则

正式安装源应为 GitHub 仓，而非本地 clone。

推荐命令：

```bash
npx skills add yourorg/your-skills-repo -g --all
```

换新电脑时也相同。

## 更新规则

滚动更新：

```bash
npx skills update
```

若你需要强一致版本，使用 tag：

```bash
npx skills add yourorg/your-skills-repo#v2026.04 -g --all
```

## 发布规范

推荐两种发布策略：

### 1. 滚动发布

- GitHub Action 合并后即为最新
- 适合个人环境

### 2. 版本发布

- 每次同步完成后打 tag
- tag 命名建议：
  - `v2026.04`
  - `v2026.04.1`

适合：

- 新机器快速复现
- 团队统一装机
- 回滚

## 建议的最小实施步骤

1. 建立聚合仓。
2. 加入 `sources.yaml`。
3. 加入 `scripts/sync-skills.mjs`。
4. 加入 `sync-skills.yml` workflow。
5. 先本地运行一次同步脚本。
6. 再执行：

```bash
npx skills add . --list
```

7. 提交并推送。
8. 在另一台机器上用 GitHub 源安装验证。
