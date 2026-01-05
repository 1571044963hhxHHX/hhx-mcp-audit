# hhx-mcp-audit（MCP）

一个用于**前端工程依赖安全审计**的 MCP Server：在 Cursor 里调用 `auditPackage` 工具，即可对指定项目（本地路径或 GitHub 仓库）做依赖漏洞审计，并输出一份可直接阅读/分享的 Markdown 报告。

## 功能

- **本地项目审计**：传入项目根目录（绝对路径），分析其直接与间接依赖的漏洞风险。
- **远程仓库审计**：传入 GitHub 仓库 URL，拉取仓库根目录的 `package.json` 后进行审计。
- **报告输出**：将审计结果渲染为 Markdown 并写入你指定的 `savePath`。
- **Cursor 集成**：按 MCP 方式配置后，可直接在对话中触发审计。

## 安装

本项目发布为 npm 包：`@hhxhhxhhx/hhx-mcp-audit`。

- **方式 A（推荐）**：使用 `npx` 运行（无需全局安装）
- **方式 B**：全局安装后直接使用 `hhx-mcp-audit`

## 在 Cursor 中配置 MCP

你可以在**项目级**或**用户级**配置 MCP Server（两者选其一即可）。

### 项目级配置（推荐）

在你的项目根目录创建文件：`.cursor/mcp.json`
cursor - 首选项 - Cursor Setting - Tools & MCP 进行配置

```json
{
  "mcpServers": {
    "hhx-mcp-audit": {
      "command": "npx",
      "args": ["-y", "@hhxhhxhhx/hhx-mcp-audit"]
    }
  }
}
```

### 用户级配置（可选）

在你的用户目录创建：`~/.cursor/mcp.json`（macOS/Linux）

内容与上面一致。

> 配置完成后，重启 Cursor（或刷新 MCP Servers）使其生效。

## 如何使用（在 Cursor 对话中）

该 MCP Server 暴露一个工具：

- **tool 名称**：`auditPackage`
- **入参**：
  - `projectRoot`：本地工程根路径（绝对路径）或 GitHub 仓库 URL
  - `savePath`：报告保存路径（**必须是绝对路径**），例如：`/abs/path/to/audit.md`

### 审计本地项目

在 Cursor 里对我说类似：

> 对 `/abs/path/to/your-project` 做安全审计，输出到 `/abs/path/to/your-project/audit.md`

### 审计 GitHub 仓库

示例（仓库根目录存在 `package.json`）：

> 对 `https://github.com/owner/repo` 做安全审计，输出到 `/abs/path/to/audit.md`

也支持 `tree` URL（会转换为 `tags/<name>` 形式去拉取 `package.json`）：

> `https://github.com/owner/repo/tree/v1.2.3`

## 输出报告长什么样

报告为 Markdown，包含：

- 漏洞总数与严重性分布（critical/high/moderate/low）
- 每个漏洞包的：
  - 漏洞标题、npm advisory 编号、链接、受影响版本范围
  - 依赖链（从当前工程到漏洞包的路径）
  - 漏洞包在 lock/node_modules 解析中的位置（nodes）

## 工作原理（实现概览）

整体流程（对应 `auditPackage(projectRoot, savePath)`）：

1. **创建临时工作目录**：在项目内部的 `work/` 下创建一次性目录
2. **解析目标项目**：
   - 本地：读取 `projectRoot/package.json`
   - 远程：仅支持 `github.com`，从 GitHub 拉取仓库根目录的 `package.json`
3. **生成 lock 文件**：在工作目录写入 `package.json`，执行：
   - `npm install --package-lock-only --force`
4. **执行审计**：
   - `npm audit --json`
   - 并额外对“当前工程包本身”（`name@version`）调用 npm 安全审计接口补充结果
5. **结果规范化 & 依赖链计算**：把 `npm audit` 的结构整理为按严重性分组的统一格式，并计算依赖链
6. **渲染 Markdown**：通过 EJS 模板渲染为最终报告
7. **清理临时目录**：删除工作目录，避免污染
8. **写入文件**：把 Markdown 写到 `savePath`

## 注意与限制

- **远程审计限制**：
  - 仅支持 `github.com`
  - 仅拉取**仓库根目录**的 `package.json`（不支持 monorepo 子目录 package）
- **网络访问**：
  - 生成 lock 与审计过程需要访问 npm registry
  - 远程审计需要访问 GitHub API 与 raw 内容地址
- **不会执行你的项目脚本**：
  - 工具不会运行你的 `start/build` 等脚本；它只在临时目录内运行 `npm install --package-lock-only` 与 `npm audit`

## 本地开发

```bash
npm install
npm run start
```

然后按 MCP 的方式用 stdio 连接（Cursor 配置时也会通过 stdio 启动）。

## License

ISC


