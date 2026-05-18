---
name: "clawpanel-dev"
description: "ClawPanel development and debugging skill. Invoke when starting ClawPanel, fixing Gateway connection issues, or debugging OpenClaw integration."
---

# ClawPanel 开发与调试

本 Skill 包含 ClawPanel + OpenClaw Gateway 开发和调试的经验总结。

## 快速启动

### 前端开发模式（浏览器调试）

```bash
cd c:\Users\xmls\Documents\trae_projects\clawpanel
npm run dev
# 访问 http://localhost:1420/
```

### Tauri 桌面应用模式

```bash
npm run tauri dev
```

## 常见问题排查

### 1. Gateway 显示"未运行"但实际在运行

**症状**：端口 18789 正在监听，但前端显示 Gateway 未运行。

**可能原因**：
- `gateway-owner.json` 中的 PID 与实际进程 ID 不匹配
- 前端轮询未检测到状态变化

**解决方案**：
```powershell
# 重启 Gateway
Stop-Process -Name "node" -Force -ErrorAction SilentlyContinue
Remove-Item "C:\Users\xmls\.openclaw\gateway-owner.json" -Force
Start-Process "C:\Users\xmls\.openclaw\gateway.cmd"
```

### 2. WebSocket 连接失败 `token_missing`

**症状**：浏览器控制台显示 `token_missing` 错误。

**原因**：Vite 开发服务器的 WebSocket 代理未转发 Authorization 头。

**修复**（已应用）：
编辑 `vite.config.js`，在 WebSocket 代理中添加：
```javascript
proxy.on('proxyReqWs', (proxyReq, req, socket) => {
  // 转发 Authorization 头
  const authHeader = req.headers['authorization']
  if (authHeader) {
    proxyReq.setHeader('Authorization', authHeader)
  }
  // ...
})
```

### 3. Gateway 安装失败 `schtasks run failed`

**症状**：运行 `openclaw gateway install` 成功，但 `openclaw gateway start` 失败。

**解决方案**：
```powershell
# 手动执行 gateway.cmd
Start-Process "C:\Users\xmls\.openclaw\gateway.cmd"
```

## 关键路径

| 组件 | 路径 |
|------|------|
| OpenClaw 配置目录 | `C:\Users\xmls\.openclaw\` |
| Gateway 启动脚本 | `C:\Users\xmls\.openclaw\gateway.cmd` |
| Gateway Owner 记录 | `C:\Users\xmls\.openclaw\gateway-owner.json` |
| Gateway 日志 | `C:\Users\xmls\AppData\Local\Temp\openclaw\` |
| OpenClaw CLI | `C:\Users\xmls\AppData\Roaming\npm\openclaw` |

## 诊断命令

```powershell
# 检查端口 18789 状态
Get-NetTCPConnection -LocalPort 18789

# 检查 Gateway 日志
Get-Content "C:\Users\xmls\AppData\Local\Temp\openclaw\openclaw-2026-05-18.log" -Tail 50

# 检查 Gateway Owner
Get-Content "C:\Users\xmls\.openclaw\gateway-owner.json" | ConvertFrom-Json

# 查看 Gateway 状态
openclaw gateway status
```

## OpenClaw 配置示例

`openclaw.json` 中的关键配置：
```json
{
  "gateway": {
    "port": 18789,
    "bind": "loopback",
    "auth": {
      "mode": "token",
      "token": "你的token"
    },
    "controlUi": {
      "allowedOrigins": [
        "http://localhost:1420",
        "http://127.0.0.1:1420"
      ]
    }
  },
  "models": {
    "providers": {
      "minimax": {
        "apiKey": "你的API密钥",
        "baseUrl": "https://api.minimaxi.com/v1",
        "models": [
          {
            "id": "MiniMax-M2.7",
            "maxTokens": 8192
          }
        ]
      }
    }
  }
}
```

## 相关文件

- Vite 配置：`vite.config.js`
- WebSocket 客户端：`src/lib/ws-client.js`
- 应用状态：`src/lib/app-state.js`
- Gateway 检测逻辑：`src-tauri/src/commands/service.rs`