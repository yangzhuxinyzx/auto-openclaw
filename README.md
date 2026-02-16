# auto-openclaw

让 OpenClaw 拥有操作电脑的能力 — 基于 UI-TARS 视觉语言模型的桌面自动化集成。

OpenClaw 是一个开源自托管 AI Agent，但它本身没有 GUI 操作能力。本项目通过将 [UI-TARS Desktop](https://github.com/bytedance/UI-TARS-desktop) 的 CLI 作为 OpenClaw 的 skill 接入，让 OpenClaw 能够像人一样看屏幕、点鼠标、打键盘，完成各种桌面自动化任务。

## 工作原理

```
用户 → OpenClaw（AI Agent）→ ui-tars skill → UI-TARS CLI → 截屏 → VLM 推理 → 执行操作 → 循环
                                                                                          ↓
                                                              JSON Lines 反馈 ← 最终截图 ← 完成
```

1. 用户给 OpenClaw 下达 GUI 任务（如"打开浏览器访问抖音"）
2. OpenClaw 识别到需要 GUI 操作，调用 `ui-tars` skill
3. UI-TARS CLI 启动 agent 循环：截屏 → 视觉语言模型推理 → 执行鼠标键盘操作
4. 每轮操作以 JSON Lines 实时输出，任务完成后返回操作摘要和最终截图
5. OpenClaw 解读结果，向用户汇报

## 前置要求

- Node.js >= 20
- pnpm 9.10.0
- [OpenClaw](https://github.com/anthropics/openclaw) 已安装
- VLM API（如火山引擎 Doubao-Seed-1.6-VL）

## 安装

### 1. 构建 UI-TARS CLI

```bash
# 克隆 UI-TARS Desktop（或使用已有的项目目录）
git clone https://github.com/bytedance/UI-TARS-desktop.git
cd UI-TARS-desktop
git checkout v0.3.0-beta.11
pnpm install

# 用本项目的改动文件覆盖原文件
cp <auto-openclaw>/src/cli/start.ts packages/ui-tars/cli/src/cli/start.ts
cp <auto-openclaw>/src/cli/commands.ts packages/ui-tars/cli/src/cli/commands.ts
cp <auto-openclaw>/src/operator-nut-js/index.ts packages/ui-tars/operators/nut-js/src/index.ts

# 按顺序构建
cd packages/ui-tars/shared && pnpm run build && cd -
cd packages/ui-tars/sdk && pnpm run build && cd -
cd packages/ui-tars/operators/nut-js && pnpm run build && cd -
cd packages/ui-tars/cli && pnpm run build && cd -

# 全局链接 CLI
cd packages/ui-tars/cli && npm link
```

验证安装：

```bash
ui-tars --version   # 应输出 1.2.3
ui-tars start --help  # 应显示 --output 选项
```

### 2. 配置 VLM 模型

创建 `~/.ui-tars-cli.json`：

```json
{
  "baseURL": "https://ark.cn-beijing.volces.com/api/v3",
  "apiKey": "<你的 API Key>",
  "model": "doubao-seed-1-6-251015",
  "useResponsesApi": true
}
```

支持任何 OpenAI 兼容的视觉语言模型 API。推荐使用火山引擎的 Doubao-Seed-1.6-VL。

### 3. 安装 OpenClaw Skill

将 `skills/ui-tars/SKILL.md` 复制到 OpenClaw workspace：

```bash
mkdir -p ~/.openclaw/workspace/skills/ui-tars
cp skills/ui-tars/SKILL.md ~/.openclaw/workspace/skills/ui-tars/
```

OpenClaw 会自动发现 `workspace/skills/` 下的 skill，无需额外配置。

## 使用

安装完成后，在 OpenClaw 对话中直接下达 GUI 任务即可：

```
> 打开浏览器访问抖音
> 打开微信搜索某某发一条消息
> 打开记事本输入 Hello World
```

OpenClaw 会自动调用 ui-tars skill，执行完成后汇报结果。

### 手动测试 CLI

```bash
# 简单测试
ui-tars start --target nut-js --query "点击桌面空白处" --output json

# 复杂任务
ui-tars start --target nut-js --query "打开浏览器访问抖音" --output json
```

### 输出格式（JSON Lines）

每行一个 JSON 对象：

| event | 含义 | 关键字段 |
|-------|------|----------|
| `screenshot` | 截屏完成 | `loop`, `width`, `height` |
| `prediction` | 模型决策 | `loop`, `action_type`, `thought`, `action_inputs` |
| `error` | 出错 | `message`, `status` |
| `done` | 任务结束 | `status`, `loops`, `summary`, `screenshotPath` |

`done` 事件包含：
- `status` — `end`（成功）/ `error`（失败）/ `call_user`（需人工）
- `summary` — 每步操作摘要（含模型思考过程）
- `screenshotPath` — 最终截图路径（PNG）

退出码：`0` 成功 / `1` 出错 / `2` 需人工 / `3` 用户中止

## 对 UI-TARS Desktop 的改动

基于 `v0.3.0-beta.11`，本项目做了以下改动：

### CLI 结构化输出（`packages/ui-tars/cli/`）

- `commands.ts` — 新增 `--output <text|json>` 选项
- `start.ts` — 启用 `onData` 回调输出 JSON Lines，从原始预测文本提取 thought（修复解析器丢失中文 thought 的问题），任务结束后保存最终截图并输出操作摘要，根据最终状态设置退出码

### NutJS Operator 修复（`packages/ui-tars/operators/nut-js/`）

- `type` 操作：剪贴板访问失败时 fallback 到 `keyboard.type()` 逐字输入，解决非交互式环境下 clipboardy 崩溃问题
- scroll 改进：Windows 下滚动前先点击目标位置，滚动量 500→1200，支持水平滚动
- wait 时间：5s→2s

## 项目结构

```
auto-openclaw/
├── README.md
├── skills/
│   └── ui-tars/
│       └── SKILL.md              # OpenClaw skill 定义
└── src/
    ├── cli/
    │   ├── start.ts              # CLI 主逻辑（JSON 输出、退出码、截图）
    │   └── commands.ts           # CLI 命令定义（--output 选项）
    └── operator-nut-js/
        └── index.ts              # NutJS 桌面操作器（剪贴板 fallback、scroll 修复）
```

## License

Apache-2.0
