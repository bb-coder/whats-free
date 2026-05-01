---
name: stitch-cli
description: >-
  Generates and iterates Google Stitch UI mockups from natural-language prompts
  using the Stitch CLI (stitch-design-cli) with JSON output, including batch
  generation, multi-screen fetch, edits, and variants. Use when the user wants
  Stitch designs, screens from prompts, HTML/screenshot artifacts, or automated
  design workflows alongside official Stitch SDK docs at stitch.withgoogle.com.
---

# Google Stitch CLI（设计稿生成）

面向 **终端 / Agent** 的流程：用 **官方 Stitch API**（底层为 `@google/stitch-sdk` / MCP）在仓库外生成界面稿，并通过 **CLI** 稳定拿到 JSON、截图与 HTML 链接。

## 文档与工具关系

| 资源 | 说明 |
|------|------|
| [Stitch SDK 教程](https://stitch.withgoogle.com/docs/sdk/tutorial/) | Google 官方入门与概念 |
| [@google/stitch-sdk](https://www.npmjs.com/package/@google/stitch-sdk) | 官方 Node SDK（编程调用） |
| [stitch-design-cli](https://www.npmjs.com/package/stitch-design-cli) | **本 Skill 默认 CLI**（社区维护，面向稳定 JSON/批量；与官方 SDK 同一 API 面） |

若仅需在代码里调用，优先直接读官方 SDK README；若需在 shell 里可解析输出、批量跑屏，用下面的 `stitch` 命令。

## 安装

```bash
npm install -g stitch-design-cli
# 或单次执行
npx -y stitch-design-cli <subcommand> ...
```

## 鉴权（切勿把密钥写进仓库）

1. **推荐**：仅对环境变量赋值，不落盘密钥：

   ```bash
   export STITCH_API_KEY="…"   # 从 Stitch / Google 控制台获取，勿提交 Git
   npx -y stitch-design-cli doctor --json
   ```

2. **本机缓存**（可选）：`stitch auth set` 或 `printf '%s' "$STITCH_API_KEY" | stitch auth set --stdin`  
   密钥仍只应保存在用户机器上，**不要**放进项目目录或 Skill 文件。

可选环境变量（与官方 SDK 一致）：`STITCH_API_KEY`、`STITCH_ACCESS_TOKEN`、`GOOGLE_CLOUD_PROJECT`、`STITCH_HOST`、`STITCH_TIMEOUT_MS`。

## 健康检查

```bash
npx -y stitch-design-cli doctor --json
npx -y stitch-design-cli auth status --json
```

`tool list` 正常但 `project list` 报 `AUTH_FAILED` 时：轮换 API Key 或改用 OAuth + `GOOGLE_CLOUD_PROJECT`。

## 单屏：从需求生成设计稿

1. 选定或新建项目：

   ```bash
   npx -y stitch-design-cli project list --json
   npx -y stitch-design-cli project create --title "Whats Free concepts" --json
   ```

   从 JSON 中取出 `projectId`（或等价字段），记为 `PROJECT_ID`。

2. 用自然语言生成一屏（`--json` 便于 Agent 解析）：

   ```bash
   npx -y stitch-design-cli screen generate \
     --project-id "$PROJECT_ID" \
     --prompt "极简深色主题的「免费资源」列表页，含筛选与卡片" \
     --device-type DESKTOP \
     --include-image \
     --include-html \
     --json
   ```

   `device-type`：`DESKTOP` | `MOBILE` | `TABLET` | `AGNOSTIC`（默认 `DESKTOP`）。  
   可选 `--model-id`（以当前 CLI `--help` 为准）。

3. 从返回 JSON 收集 `screenId`，拉取截图/HTML URL：

   ```bash
   npx -y stitch-design-cli screen get \
     --project-id "$PROJECT_ID" \
     --screen-id "$SCREEN_ID" \
     --include-image \
     --include-html \
     --json
   ```

将 URL 交给用户下载，或在本机 `curl -O` 落盘（路径**不要**指向含密钥的目录再误提交）。

## 批量：多句需求 → 多屏

**模式 A — 多次 `screen generate`（每个需求一屏）**

对 `prompts.txt` 每行一条描述：

```bash
PROJECT_ID="…"
while IFS= read -r line; do
  [ -z "$line" ] && continue
  npx -y stitch-design-cli screen generate \
    --project-id "$PROJECT_ID" \
    --prompt "$line" \
    --device-type MOBILE \
    --include-image \
    --json
done < prompts.txt
```

**模式 B — 一次拉取多屏资产**

生成阶段已得到多个 `screenId` 后，可**重复** `--screen-id` 或逗号列表：

```bash
npx -y stitch-design-cli screen get \
  --project-id "$PROJECT_ID" \
  --screen-id "$ID1" --screen-id "$ID2" --screen-id "$ID3" \
  --include-image \
  --include-html \
  --json
```

**模式 C — 多屏统一改版（批量 edit）**

```bash
npx -y stitch-design-cli screen edit \
  --project-id "$PROJECT_ID" \
  --screen-id "id1,id2" \
  --prompt "统一更大的留白与 WCAG 对比度更高的主按钮" \
  --include-image \
  --json
```

**模式 D — 基于现有屏做多方案 variants**

```bash
npx -y stitch-design-cli screen variants \
  --project-id "$PROJECT_ID" \
  --screen-id "$SCREEN_ID" \
  --prompt "三种更轻的品牌色方向，留白更大" \
  --variant-count 3 \
  --creative-range EXPLORE \
  --aspect COLOR_SCHEME \
  --aspect LAYOUT \
  --include-image \
  --json
```

`--creative-range`：`REFINE` | `EXPLORE` | `REIMAGINE`。`--aspect` 可取：`LAYOUT`、`COLOR_SCHEME`、`IMAGES`、`TEXT_FONT`、`TEXT_CONTENT`（可重复）。

## Agent 执行约定

1. **总是加 `--json`**，方便解析 `screenId` 与 artifact URL。  
2. **永远不要**在 Skill、README、issue、commit 中写入 `STITCH_API_KEY`；提示用户用环境变量或本机密钥管理。  
3. 免费额度与条款以 **Stitch 官方当前政策** 为准；生成内容版权与使用范围遵循用户与 Google/Stitch 的协议。  
4. 若用户要「和官方教程一致」的代码集成，引导其阅读 [SDK 教程](https://stitch.withgoogle.com/docs/sdk/tutorial/) 并使用 `import { stitch } from "@google/stitch-sdk"`；CLI 是自动化补充而非替代文档。

## 故障排查速查

| 现象 | 处理 |
|------|------|
| `AUTH_FAILED` | 检查 Key 是否过期、是否有项目权限；尝试轮换 Key 或 OAuth |
| 仅部分命令失败 | `doctor --json` 与 `tool list --json` 分项确认 |
| 超时 | 提高 `STITCH_TIMEOUT_MS`（大 prompt / 多 variants） |
