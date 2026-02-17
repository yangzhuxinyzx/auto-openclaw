---
name: ui-tars
description: GUI 桌面自动化工具 — 通过自然语言指令操控电脑桌面
version: 0.1.0
requires:
  bins:
    - ui-tars
---

# UI-TARS — GUI Desktop Automation

UI-TARS 是一个视觉语言模型驱动的桌面自动化工具。你给它一条自然语言指令，它会自动截屏、识别界面、执行鼠标键盘操作，循环直到任务完成。

## 使用方式

```bash
ui-tars start --target nut-js --query "<你的指令>" --output json
```

参数说明：
- `--target nut-js`：使用桌面操作器（必须）
- `--query "<指令>"`：自然语言任务描述
- `--output json`：输出 JSON Lines 格式，方便解析

## 输出格式（JSON Lines）

每行一个 JSON 对象，按时间顺序输出：

| event | 含义 | 关键字段 |
|-------|------|----------|
| `screenshot` | 截屏完成 | `loop`, `width`, `height` |
| `prediction` | 模型决策 | `loop`, `action_type`, `thought`, `action_inputs` |
| `error` | 出错 | `message`, `status` |
| `done` | 任务结束 | `status`, `loops`, `summary`, `screenshotPath` |

### done 事件详情

最后一行 `done` 事件包含完整反馈：
- `status` — 最终状态（`end` = 成功，`error` = 失败，`call_user` = 需人工）
- `loops` — 总循环次数
- `summary` — 字符串数组，每一步的操作摘要（含模型思考过程）
- `screenshotPath` — 任务完成后的最终截图路径（PNG），可用 Read 工具查看结果

### 如何解读结果

1. 先看 `done` 事件的 `status` 判断成败
2. 读 `summary` 了解 UI-TARS 做了什么、每步在想什么
3. **必须**用 Read 工具读取 `screenshotPath` 的截图来确认最终视觉结果，这是验证任务是否真正完成的唯一方式
4. 将截图内容和 summary 一起反馈给用户

### action_type 常见值

- `click` / `left_double` / `right_single` — 鼠标点击
- `type` — 键盘输入（`action_inputs.content`）
- `hotkey` — 快捷键（`action_inputs.key`，如 `ctrl+c`）
- `scroll` — 滚动（`action_inputs.direction`：up/down/left/right）
- `drag` — 拖拽
- `wait` — 等待
- `finished` — 任务完成
- `call_user` — 需要人工介入

## 退出码

- `0` — 任务成功完成（finished）
- `1` — 出错（error）
- `2` — 需要人工介入（call_user）
- `3` — 用户中止（user_stopped）

## 重要注意事项

1. **执行期间会接管鼠标和键盘**，不要同时手动操作电脑
2. 任务执行可能需要多轮循环（截屏→决策→操作），耐心等待
3. 如果退出码为 2（call_user），说明 UI-TARS 遇到了无法自行解决的问题，需要告知用户
4. 模型配置存储在 `~/.ui-tars-cli.json`，首次使用需要配置 VLM API 地址和密钥
5. 超时建议：设置 `timeout 300` 秒（5分钟），复杂任务可能需要更长时间

## 使用示例

简单任务：
```bash
ui-tars start --target nut-js --query "打开记事本并输入 Hello World" --output json
```

解读输出：读取最后一行 `{"event":"done",...}` 的 `status` 字段判断是否成功，`summary` 查看操作过程，`screenshotPath` 查看最终截图确认结果。
