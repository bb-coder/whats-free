---
name: stitch-sdk
description: >-
  Generates and iterates Google Stitch UI mockups from natural-language prompts
  using the official @google/stitch-sdk (Node.js), including batch generation
  across prompts, edits, variants, and fetching HTML/screenshot URLs. Use when
  the user wants Stitch designs, programatic screen generation, or workflows
  aligned with https://stitch.withgoogle.com/docs/sdk/tutorial/.
---

# Google Stitch 官方 SDK（设计稿生成）

使用 **[@google/stitch-sdk](https://www.npmjs.com/package/@google/stitch-sdk)**（官方 npm 包，Apache-2.0）通过 Node.js 调用 Stitch，从文本需求生成界面稿并拿到 HTML / 截图下载链接。可与 [官方 SDK 教程](https://stitch.withgoogle.com/docs/sdk/tutorial/) 对照阅读。

## 安装与环境

- **运行时**：Node.js **≥ 18**。
- **安装依赖**：

  ```bash
  npm install @google/stitch-sdk
  ```

- **鉴权（切勿把密钥写入仓库或 Skill）**：设置环境变量 `STITCH_API_KEY`（或 OAuth 场景下的 `STITCH_ACCESS_TOKEN` + `GOOGLE_CLOUD_PROJECT`）。可选：`STITCH_HOST`、`STITCH_TIMEOUT_MS`。

  ```bash
  export STITCH_API_KEY="…"
  ```

## 最小示例（ESM，单文件可跑）

将下列保存为 `stitch-once.mjs`，在项目目录已 `npm install @google/stitch-sdk` 后执行 `node stitch-once.mjs`。

```javascript
import { stitch, StitchError } from "@google/stitch-sdk";

async function main() {
  const project = await stitch.createProject("Agent sandbox");
  const screen = await project.generate(
    "极简深色主题的「免费资源」列表页，含筛选与卡片",
    "DESKTOP",
  );
  const htmlUrl = await screen.getHtml();
  const imageUrl = await screen.getImage();
  console.log(
    JSON.stringify(
      {
        projectId: project.projectId,
        screenId: screen.screenId,
        htmlUrl,
        imageUrl,
      },
      null,
      2,
    ),
  );
}

main().catch((e) => {
  if (e instanceof StitchError) {
    console.error(e.code, e.message);
  } else {
    console.error(e);
  }
  process.exit(1);
});
```

`project.generate` 的第二参数为设备类型：`"MOBILE"` \| `"DESKTOP"` \| `"TABLET"` \| `"AGNOSTIC"`。第三参数可选 `modelId`：`"GEMINI_3_PRO"` \| `"GEMINI_3_FLASH"`（以当前 SDK 类型定义为准）。

## 使用已有项目

```javascript
import { stitch } from "@google/stitch-sdk";

const projects = await stitch.projects();
const project = stitch.project("你的 projectId"); // 无需先 list，已知 id 可直接引用
const screens = await project.screens();
```

## 批量：多条需求 → 多屏

对多个 prompt **顺序**调用 `project.generate`（便于控制速率与错误处理）：

```javascript
import { stitch } from "@google/stitch-sdk";

const project = await stitch.createProject("Batch run");
const prompts = [
  "移动端登录页，邮箱+密码",
  "设置页：通知开关与深色模式",
];

const out = [];
for (const prompt of prompts) {
  const screen = await project.generate(prompt, "MOBILE");
  out.push({
    prompt,
    screenId: screen.screenId,
    htmlUrl: await screen.getHtml(),
    imageUrl: await screen.getImage(),
  });
}
console.log(JSON.stringify(out, null, 2));
```

需要并行时可用 `Promise.all`，但注意 **API 速率限制**；批量更稳妥的方式通常是有限并发或队列。

## 编辑已有屏（可循环多屏）

```javascript
const project = stitch.project(projectId);
const screen = await project.generate("仪表盘，含三张统计卡", "DESKTOP");
const edited = await screen.edit("整体改为深色侧栏 + WCAG 对比度更高的主按钮");
const htmlUrl = await edited.getHtml();
```

对多个 `screenId`：逐个 `await project.getScreen(id)` 再 `edit(prompt)` 即可。

## 多方案 Variants

```javascript
const screen = await project.generate("品牌着陆页 hero 区", "DESKTOP");
const variants = await screen.variants("三种更轻的品牌色、更大留白", {
  variantCount: 3,
  creativeRange: "EXPLORE",
  aspects: ["COLOR_SCHEME", "LAYOUT"],
});

for (const v of variants) {
  console.log(v.screenId, await v.getHtml());
}
```

`creativeRange`：`"REFINE"` \| `"EXPLORE"` \| `"REIMAGINE"`。`aspects` 可选 `"LAYOUT"`、`"COLOR_SCHEME"`、`"IMAGES"`、`"TEXT_FONT"`、`"TEXT_CONTENT"`。

## Agent / 底层工具调用（可选）

需要直接调 MCP 工具名与参数时，使用 `StitchToolClient`（用完 `close()`）：

```javascript
import { StitchToolClient } from "@google/stitch-sdk";

const client = new StitchToolClient({ apiKey: process.env.STITCH_API_KEY });
const { tools } = await client.listTools();
await client.close();
```

也可用全局 `stitch` 上的工具封装（见包内 README 的 `stitchTools()` / ADK 集成），按编排框架选用。

## 连通性自检

无单独「doctor」命令时，可调用 `stitch.projects()` 或 `stitch.listTools()`；失败时捕获 `StitchError`，常见 `code`：`AUTH_FAILED`、`RATE_LIMITED`、`NETWORK_ERROR` 等。

## Agent 执行约定

1. **永远不要把 `STITCH_API_KEY` 写进仓库、Skill、Issue、commit**；用环境变量或本机密钥管理。  
2. 解析输出时优先 **`JSON.stringify` 结构化日志**，便于 Agent 读取 `projectId` / `screenId` / URL。  
3. 免费额度与条款以 **Stitch 官方现行说明** 为准；SDK 免责声明见包 README（非官方支持产品声明等）。  
4. HTML / 图片 URL 一般为 **临时下载链接**，需要落盘时及时 `fetch`，避免链接受限。

## 故障排查速查

| 现象 | 处理 |
|------|------|
| `AUTH_FAILED` | 轮换 Key 或检查 OAuth + `GOOGLE_CLOUD_PROJECT` |
| `RATE_LIMITED` | 降低并发、重试退避 |
| 超时 | 增大 `STITCH_TIMEOUT_MS` 或拆分 prompt |
