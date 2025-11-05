## 前端连接 WebSocket 服务器核心方法

### 核心流程概览
- 页面初始化后调用 `connectWs`，与 `ws://12.34.56.78:9999/` 建立长链接，可改成自己的服务器地址。
- 服务器在首次连接时返回一个 `clientId`，前端保存后生成二维码供 DG-LAB App 扫码绑定。
- 完成绑定后，网页与 App 在后续通讯中都要携带 `clientId` 和 `targetId`，服务端据此校验来源是否合法并转发消息。
- 页面通过 `sendWsMsg` 统一封装发包；服务器下发的 `msg`、`bind`、`break`、`error`、`cpuAuto` 等类型在 `onmessage` 中分支处理。
- 新增的 CPU 同步模式会根据 Windows 的 CPU 利用率自动调节强度，页面实时展示采样结果。

### 关键代码示例
```javascript
var connectionId = "";        // 浏览器在本次通信中的唯一标识
var targetWSId = "";          // 已绑定 App 的唯一标识
var wsConn = null;             // 全局 WebSocket 实例
let cpuSyncEnabled = false;    // CPU 同步开关状态
const cpuSyncOptions = {       // CPU 同步默认参数，可按需覆盖
    channel: "both",           // 推送到 A、B 通道，可选 "A"、"B"
    intervalMs: 1000,          // 采样间隔（毫秒）
    minStrength: 10,           // 映射的最小强度
    maxStrength: 120,          // 映射的最大强度
    smoothing: 0.3             // 指数平滑系数，越大越平滑
};

function connectWs() {
    wsConn = new WebSocket("ws://12.34.56.78:9999/");

    wsConn.onopen = () => console.log("WebSocket 连接已建立");

    wsConn.onmessage = (event) => {
        let message;
        try {
            message = JSON.parse(event.data);
        } catch (e) {
            console.log("收到非 JSON 数据:", event.data);
            return;
        }

        switch (message.type) {
            case "bind":
                // 首次拿到 clientId，生成二维码给 App 扫码
                if (!message.targetId) {
                    connectionId = message.clientId;
                    qrcodeImg.clear();
                    qrcodeImg.makeCode(`https://www.dungeon-lab.com/app-download.php#DGLAB-SOCKET#ws://12.34.56.78:9999/${connectionId}`);
                } else {
                    // 已绑定成功，保存 targetId 并更新 UI
                    if (message.clientId !== connectionId) {
                        alert("收到不匹配的 target 消息");
                        return;
                    }
                    targetWSId = message.targetId;
                    document.getElementById("status").innerText = "已连接";
                    document.getElementById("status-btn").innerText = "断开";
                    hideqrcode();
                }
                break;
            case "msg":
                // App 回传的强度信息与反馈提示
                if (message.message.includes("strength")) {
                    const numbers = message.message.match(/\d+/g).map(Number);
                    document.getElementById("channel-a").innerText = numbers[0];
                    document.getElementById("channel-b").innerText = numbers[1];
                    document.getElementById("soft-a").innerText = numbers[2];
                    document.getElementById("soft-b").innerText = numbers[3];
                } else if (message.message.includes("feedback")) {
                    showSuccessToast(feedBackMsg[message.message]);
                }
                break;
            case "cpuAuto":
                handleCpuAutoMessage(message); // 处理 CPU 同步的 started / tick / stopped 反馈
                break;
            case "break":
                showToast("对方已断开，code:" + message.message);
                resetCpuSyncUI();
                location.reload();
                break;
            case "error":
                showToast(message.message);
                break;
            case "heartbeat":
                flashHeartbeatLight();
                break;
            default:
                console.log("收到其他消息:", message);
                break;
        }
    };

    wsConn.onclose = () => {
        showToast("连接已断开");
        resetCpuSyncUI();
    };
}
```

### 常用发送方法
```javascript
function sendWsMsg(payload) {
    if (!wsConn) {
        showToast("WebSocket 尚未建立");
        return;
    }
    payload.clientId = connectionId;
    payload.targetId = targetWSId;
    if (!payload.type) {
        payload.type = "msg"; // 默认作为普通消息
    }
    wsConn.send(JSON.stringify(payload));
}

function toggleCpuSync() {
    if (!connectionId || !targetWSId) {
        showToast("请先完成绑定再开启 CPU 同步");
        return;
    }

    if (cpuSyncEnabled) {
        sendWsMsg({ type: "cpuAuto", action: "stop" });
    } else {
        sendWsMsg({
            type: "cpuAuto",
            action: "start",
            channel: cpuSyncOptions.channel,
            intervalMs: cpuSyncOptions.intervalMs,
            minStrength: cpuSyncOptions.minStrength,
            maxStrength: cpuSyncOptions.maxStrength,
            smoothing: cpuSyncOptions.smoothing
        });
    }
}
```

## 后端 WebSocket 服务核心方法

### 服务端职责
- 为每一个新连接分配唯一 `clientId`，负责维护 `clients`（连接实例）与 `relations`（绑定关系）。
- 校验消息来源是否合法（是否是绑定的一方），并在前端与 App 之间转发 `bind`/`msg`/`clientMsg` 等业务报文。
- 管理心跳、错误、断线等情况，确保双方能得到即时反馈。
- 新增 `cpuAuto` 指令处理，定时采样主机 CPU 利用率并映射为强度指令推送给 App。

### 关键代码示例
```javascript
const WebSocket = require('ws');
const { v4: uuidv4 } = require('uuid');
const os = require('os');

const clients = new Map();        // 保存所有在线 websocket 连接
const relations = new Map();      // 保存 clientId -> targetId 绑定关系
const clientTimers = new Map();   // 波形发送定时器（按通道区分）
const cpuControllers = new Map(); // CPU 同步定时器

const heartbeatMsg = { type: "heartbeat", clientId: "", targetId: "", message: "200" };
let heartbeatInterval;

const wss = new WebSocket.Server({ port: 9999 });

wss.on('connection', (ws) => {
    const clientId = uuidv4();
    clients.set(clientId, ws);
    ws.send(JSON.stringify({ type: 'bind', clientId, message: 'targetId', targetId: '' }));

    ws.on('message', (message) => {
        let data;
        try {
            data = JSON.parse(message);
        } catch (err) {
            ws.send(JSON.stringify({ type: 'msg', clientId: "", targetId: "", message: '403' }));
            return;
        }

        // 只允许关联的一对连接互相转发消息
        if (clients.get(data.clientId) !== ws && clients.get(data.targetId) !== ws) {
            ws.send(JSON.stringify({ type: 'msg', clientId: "", targetId: "", message: '404' }));
            return;
        }

        switch (data.type) {
            case "bind":
                handleBind(ws, data);
                break;
            case 1:
            case 2:
            case 3:
            case 4:
            case "clientMsg":
                handleStrength(ws, data);
                break;
            case "cpuAuto":
                handleCpuAuto(ws, data);
                break;
            default:
                forwardMessage(data);
                break;
        }
    });

    ws.on('close', () => cleanupConnection(clientId));
    ws.on('error', (err) => notifyError(clientId, err));

    ensureHeartbeat();
});
```

### CPU 同步核心逻辑
```javascript
function handleCpuAuto(ws, data) {
    const { clientId, targetId } = data;
    if (relations.get(clientId) !== targetId) {
        ws.send(JSON.stringify({ type: "cpuAuto", clientId, targetId, message: "402" }));
        return;
    }

    const key = buildCpuKey(clientId, targetId);
    if (data.action && data.action.toLowerCase() === 'stop') {
        stopCpuController(key, true);
        return;
    }

    const options = normaliseCpuOptions(data);
    startCpuController(key, { clientId, targetId, options });
    ws.send(JSON.stringify({ type: "cpuAuto", clientId, targetId, message: "started", options }));
}

function startCpuController(key, context) {
    stopCpuController(key, false);

    const state = {
        clientId: context.clientId,
        targetId: context.targetId,
        options: context.options,
        snapshot: captureCpuSnapshot(),
        smoothedUsage: null
    };

    state.intervalId = setInterval(() => {
        const controlSocket = clients.get(state.clientId);
        const targetSocket = clients.get(state.targetId);
        if (!controlSocket || !targetSocket) {
            stopCpuController(key, false);
            return;
        }

        const nextSnapshot = captureCpuSnapshot();
        const usageRatio = computeCpuUsage(state.snapshot, nextSnapshot);
        state.snapshot = nextSnapshot;
        if (usageRatio === null) return;

        // 指数平滑，避免强度频繁抖动
        state.smoothedUsage = state.smoothedUsage === null
            ? usageRatio
            : state.smoothedUsage * state.options.smoothing + usageRatio * (1 - state.options.smoothing);

        const mapped = mapCpuToStrength(state.smoothedUsage, state.options);
        state.options.channels.forEach((channel) => {
            targetSocket.send(JSON.stringify({
                type: "msg",
                clientId: state.clientId,
                targetId: state.targetId,
                message: `strength-${channel}+2+${mapped}`
            }));
        });

        controlSocket.send(JSON.stringify({
            type: "cpuAuto",
            clientId: state.clientId,
            targetId: state.targetId,
            message: "tick",
            usage: Number((state.smoothedUsage * 100).toFixed(2)),
            strength: mapped
        }));
    }, state.options.intervalMs);

    cpuControllers.set(key, state);
}
```

## CPU 强度同步指令说明
- 前端发送 `{ type: "cpuAuto", action: "start", channel, intervalMs, minStrength, maxStrength, smoothing }` 即可启动 CPU→强度映射；`action: "stop"` 会立刻终止并收到 `message: "stopped"`。
- `channel` 支持 `"both"`、`"A"`、`"B"`；`intervalMs` 控制采样周期（200~5000ms），`minStrength` / `maxStrength` 控制输出区间，`smoothing` (0~0.99) 用于指数平滑。
- 服务端会基于 `os.cpus()` 读取宿主机 CPU 总时间差并计算忙碌率，按设置的区间映射为强度，向 App 下发 `strength-<channel>+2+<value>` 指令，同时向网页返回 `type:"cpuAuto"` 的 `tick` 信息用于更新 UI。
- 未完成绑定会返回 `message:"402"`；任意一侧断线或手动停止都会清理定时器并返回 `message:"stopped"`。
