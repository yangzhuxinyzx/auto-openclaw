# auto-openclaw

让 OpenClaw 拥有操作电脑的能力 — 基于 UI-TARS 视觉语言模型的桌面自动化集成。

本项目基于 [UI-TARS Desktop](https://github.com/bytedance/UI-TARS-desktop)（v0.3.0-beta.11）修改，将其 CLI 作为 [OpenClaw](https://github.com/anthropics/openclaw) 的 skill 接入，让 OpenClaw 能够像人一样看屏幕、点鼠标、打键盘。

## 工作原理

```
用户 → OpenClaw → ui-tars skill → UI-TARS CLI → 截屏 → VLM 推理 → 执行操作 → 循环
                                                                                    ↓
                                                        JSON Lines 反馈 ← 最终截图 ← 完成
```

## 前置要求

- Node.js >= 20
- pnpm 9.10.0
- [OpenClaw](https://github.com/anthropics/openclaw) 已安装
- VLM API（推荐火山引擎 doubao-seed-2-0-pro-260215）

## 安装

### 1. 克隆并构建

```bash
git clone https://github.com/yangzhuxinyzx/auto-openclaw.git
cd auto-openclaw
pnpm install

# 按顺序构建
cd packages/ui-tars/shared && pnpm run build && cd -
cd packages/ui-tars/sdk && pnpm run build && cd -
cd packages/ui-tars/operators/nut-js && pnpm run build && cd -
cd packages/ui-tars/cli && pnpm run build && cd -

# 全局链接 CLI
cd packages/ui-tars/cli && npm link
```

验证：

```bash
ui-tars --version        # 应输出 1.2.3
ui-tars start --help     # 应显示 --output 选项
```

### 2. 配置 VLM 模型

创建 `~/.ui-tars-cli.json`：

```json
{
  "baseURL": "https://ark.cn-beijing.volces.com/api/v3",
  "apiKey": "<你的 API Key>",
  "model": "doubao-seed-2-0-pro-260215",
  "useResponsesApi": true
}
```

支持任何 OpenAI 兼容的视觉语言模型 API。

### 3. 接入 OpenClaw

将 skill 文件复制到 OpenClaw workspace：

```bash
mkdir -p ~/.openclaw/workspace/skills/ui-tars
cp skills/ui-tars/SKILL.md ~/.openclaw/workspace/skills/ui-tars/
```

OpenClaw 会自动发现并加载，无需额外配置。

## 使用

在 OpenClaw 对话中直接下达 GUI 任务：

```
> 打开浏览器访问抖音
> 打开微信搜索某某发一条消息
> 打开记事本输入 Hello World
```

### 手动测试

```bash
ui-tars start --target nut-js --query "点击桌面空白处" --output json
```

### 输出格式

JSON Lines，每行一个事件：

| event | 含义 | 关键字段 |
|-------|------|----------|
| `screenshot` | 截屏 | `loop`, `width`, `height` |
| `prediction` | 模型决策 | `action_type`, `thought`, `action_inputs` |
| `error` | 出错 | `message` |
| `done` | 结束 | `status`, `summary`, `screenshotPath` |

退出码：`0` 成功 / `1` 出错 / `2` 需人工 / `3` 用户中止

## 相对原版的改动

基于 [UI-TARS Desktop v0.3.0-beta.11](https://github.com/bytedance/UI-TARS-desktop/tree/v0.3.0-beta.11)：

- `packages/ui-tars/cli/` — 新增 `--output json` 结构化输出、退出码、操作摘要、最终截图保存
- `packages/ui-tars/operators/nut-js/` — type 操作剪贴板 fallback、scroll 改进、wait 缩短

## 致谢

- [UI-TARS Desktop](https://github.com/bytedance/UI-TARS-desktop) — ByteDance 开源的 GUI Agent 桌面应用（Apache-2.0）
- [OpenClaw](https://github.com/anthropics/openclaw) — 开源自托管 AI Agent

## License

Apache-2.0（继承自 UI-TARS Desktop）
