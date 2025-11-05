## 前端链接 Websocket 服务器核心方法

```JavaScript
var connectionId = ""; // 前端页面在本次通信里的唯一ID

var targetWSId = ""; // app在本次通信里的唯一ID

let followAStrength = false; //跟随AB软上限

let followBStrength = false;

var wsConn = null; // 全局ws链接对象

function connectWs() {
    wsConn = new WebSocket("ws://12.34.56.78:9999/"); // 内容请改成您的ws服务器地址

    // ws是一个长链接，所以官方定义了几个状态方便你处理信息，onopen事件是ws链接建立成功之后自动调用的，这里我们只打印状态
    wsConn.onopen = function (event) {
        console.log("WebSocket连接已建立");
    };

    // 接下来我们定义通信协议,ws的消息是通过长链接在链接双方之间互相发送，所以需要我们主动定义通信协议
    wsConn.onmessage = function (event) {
        var message = null;
        try {
            // 获取消息内容
            message = JSON.parse(event.data);
        }
        catch (e) {
            // 消息不符合JSON格式异常处理
            console.log(event.data);
            return;
        }

        // 根据 message.type 进行不同的处理，我们定义了消息体格式，类型都是字符串{type, clientId, targetId, message}
        switch (message.type) {
            case 'bind':// 链接上第一件事就是绑定，首先让前端页面和服务器绑定一个本次通信的唯一id
                if (!message.targetId) {
                    // 链接创建时，ws服务器生成一个本次通信的id，通过cliendId传给前端（这个clientId是给前端使用的）
                    connectionId = message.clientId; // 获取 clientId
                    console.log("收到clientId：" + message.clientId);
                    qrcodeImg.clear();
                    // 通过qrcode.min.js库生成一个二维码，之后通过app扫描来和服务器创建链接（当app扫描这个二维码之后，服务器端会知道app需要和前端页面进行绑定，生成一个新的targetId返回给app，并在服务器里绑定这一对clientId和targetId，在本次连接中作为唯一通讯合法性鉴权的标志，之后每次二者互发消息时，targetId和clientId都必须携带，服务器会鉴定是否为合法消息，防止他人非法向app和前端发送消息，避免被恶意修改强度
                    qrcodeImg.makeCode("https://www.dungeon-lab.com/app-download.php#DGLAB-SOCKET#ws://12.34.56.78:9999/" + connectionId);
                    //qrcodeImg.makeCode("https://www.dungeon-lab.com/app-download.php#DGLAB-SOCKET#ws://192.168.3.235:9999/" + connectionId);
                }
                else {
                    if (message.clientId != connectionId) {
                        alert('收到不正确的target消息' + message.message)
                        return;
                    }
                    // 当app扫描了二维码之后，服务器完成了targetId的创建，需要通知前端已完成绑定，前端也保存targetId，然后就可以开始正常通信了（记得在每次发送消息的时候携带上targetId和clientId，把波形内容/强度设置等信息放在message里，type设置为msg）
                    targetWSId = message.targetId;
                    console.log("收到targetId: " + message.targetId + "msg: " + message.message);
                    hideqrcode();
                }
                break;
            case 'break':
                // app断开时，服务器通知前端，结束本次游戏
                if (message.targetId != targetWSId)
                    return;
                showToast("对方已断开，code:" + message.message)
                location.reload();
                break;
            case 'error':
                // 服务器出现异常情况和流程错误时，提醒前端
                if (message.targetId != targetWSId)
                    return;
                console.log(message); // 输出错误信息到控制台
                showToast(message.message); // 弹出错误提示框，显示错误消息
                break;
            case 'msg':
                // 正式通讯开始，消息的编码格式请查看socket/readme.md 中的 APP 收信协议
                // 先定义一个空数组来存储结果
                const result = [];
                if (message.message.includes("strength")) {
                    const numbers = message.message.match(/\d+/g).map(Number);
                    result.push({ type: "strength", numbers });
                    document.getElementById("channel-a").innerText = numbers[0]; //解析a通道强度
                    document.getElementById("channel-b").innerText = numbers[1];//解析b通道强度
                    document.getElementById("soft-a").innerText = numbers[2];//解析a通道强度软上限
                    document.getElementById("soft-b").innerText = numbers[3];//解析b通道强度软上限

                    if (followAStrength && numbers[2] !== numbers[0]) {
                        //开启跟随软上限设置  当收到和缓存不同的软上限值时触发自动设置
                        softAStrength = numbers[2]; // 保存 避免重复发信
                        const data1 = { type: 4, message: `strength-1+2+${numbers[2]}` }
                        sendWsMsg(data1);
                    }
                    if (followBStrength && numbers[3] !== numbers[1]) {
                        softBStrength = numbers[3]
                        const data2 = { type: 4, message: `strength-2+2+${numbers[3]}` }
                        sendWsMsg(data2);
                    }
                }
                else if (message.message.includes("feedback")) {
                    showSuccessToast(feedBackMsg[message.message]);
                }
                break;
            case 'heartbeat':
                //心跳包，用于监听本次通信是否网络不稳定而异常断开
                console.log("收到心跳");
                if (targetWSId !== '') {
                    // 已连接上
                    const light = document.getElementById("status-light");
                    light.style.color = '#00ff37';

                    // 1秒后将颜色设置回 #ffe99d
                    setTimeout(() => {
                        light.style.color = '#ffe99d';
                    }, 1000);
                }
                break;
            default:
                console.log("收到其他消息：" + JSON.stringify(message)); // 输出其他类型的消息到控制台
                break;
        }
    };

    wsConn.onerror = function (event) {
        console.error("WebSocket连接发生错误");
        // websocket提供的方法之一，在这里处理连接错误的情况
    };

    wsConn.onclose = function (event) {
        // websocket提供的方法之一，在这里处理连接关闭之后的操作，比如重置页面设置
        showToast("连接已断开");
    };
}
```

## 后端 Websocket 服务核心方法

```JavaScript
// 必须引入的ws链接库
const WebSocket = require('ws');
const { v4: uuidv4 } = require('uuid');

// 储存所有已连接的client，这包括前端和app，在后续消息转发时需要检查连接是否存在
const clients = new Map();

// 存储通讯关系，clientId是key，targetId是value, 本示例中为允许前端和app建立链接，一对一关系
// 如果要设置成一对N关系，请您修改Map的保存策略（将targetId对应的关系设置成数组而不是键值）
const relations = new Map();

const punishmentDuration = 5; //默认发送时间5秒

const punishmentTime = 1; // 默认一秒发送1次

// 存储客户端和发送计时器关系，每个客户端在发送波形消息时都有一个计时器，这个计时器用于波形数据的下发，比如每秒发送1次，一共发送5秒
const clientTimers = new Map();

// 定义心跳消息，告知所有已连接的client，服务器正常工作，若长时间未收到则自动断开（表示网络异常）
const heartbeatMsg = {
    type: "heartbeat",
    clientId: "",
    targetId: "",
    message: "200"
};

// 定义心跳定时器，独立于发信计时器，只要ws服务启动，此定时器就开始持续工作
let heartbeatInterval;

const wss = new WebSocket.Server({ port: 9999 }); // 定义链接端口，根据需求选择自己服务器上的可用端口

wss.on('connection', function connection(ws) {
    // ws提供的方法，链接时自动调用
    // 生成唯一的标识符clientId
    const clientId = uuidv4();

    console.log('新的 WebSocket 连接已建立，标识符为:', clientId);

    //存储这个clientId
    clients.set(clientId, ws);

    // 发送标识符给客户端（格式固定，双方都必须获取才可以进行后续通信：比如浏览器和APP，服务器仅作为一个状态管理者和消息转发工具）
    ws.send(JSON.stringify({ type: 'bind', clientId, message: 'targetId', targetId: '' }));

    // 服务器监听双方发信并处理消息
    ws.on('message', function incoming(message) {
        console.log("收到消息：" + message)
        let data = null;
        try {
            data = JSON.parse(message);
        }
        catch (e) {
            // 非JSON格式数据处理
            ws.send(JSON.stringify({ type: 'msg', clientId: "", targetId: "", message: '403' }))
            return;
        }

        // 非法消息来源拒绝，clientId和targetId并非绑定关系
        if (clients.get(data.clientId) !== ws && clients.get(data.targetId) !== ws) {
            ws.send(JSON.stringify({ type: 'msg', clientId: "", targetId: "", message: '404' }))
            return;
        }

        if (data.type && data.clientId && data.message && data.targetId) {
            // 优先处理clientId和targetId的绑定关系
            const { clientId, targetId, message, type } = data;
            switch (data.type) {
                case "bind":
                    // 服务器下发绑定关系
                    if (clients.has(clientId) && clients.has(targetId)) {
                        // relations的双方都不存在这俩id才能绑定，防止app绑定多个前端
                        if (![clientId, targetId].some(id => relations.has(id) || [...relations.values()].includes(id))) {
                            relations.set(clientId, targetId)
                            const client = clients.get(clientId);
                            const sendData = { clientId, targetId, message: "200", type: "bind" }
                            ws.send(JSON.stringify(sendData));
                            client.send(JSON.stringify(sendData));
                        }
                        else {
                            // 此id已被绑定 拒绝再次绑定
                            const data = { type: "bind", clientId, targetId, message: "400" }
                            ws.send(JSON.stringify(data))
                            return;
                        }
                    } else {
                        const sendData = { clientId, targetId, message: "401", type: "bind" }
                        ws.send(JSON.stringify(sendData));
                        return;
                    }
                    break;
                     // 正式通讯开始，消息的编码格式请查看socket/readme.md 中的 APP 收信协议
                case 1:
                case 2:
                case 3:
                    // clientId请求调节targetId的强度，服务器审核链接合法后下发APP强度调节
                    if (invalidRelation(cliendId, targetId, ws)) return; // 鉴定是否为绑定关系
                        const client = clients.get(targetId);
                        const sendType = data.type - 1;
                        const sendChannel = data.channel ? data.channel : 1;
                        const sendStrength = data.type >= 3 ? data.strength : 1 //增加模式强度改成1
                        const msg = "strength-" + sendChannel + "+" + sendType + "+" + sendStrength;
                        const sendData = { type: "msg", clientId, targetId, message: msg }
                        client.send(JSON.stringify(sendData));
                    break;
                case 4:
                    // clientId请求指定targetId的强度，服务器审核链接合法后下发指定APP强度
                    if (invalidRelation(cliendId, targetId, ws)) return; // 鉴定是否为绑定关系

                        const client = clients.get(targetId);
                        const sendData = { type: "msg", clientId, targetId, message }
                        client.send(JSON.stringify(sendData));

                    break;
                case "clientMsg":
                    // clientId发送给targetId的波形消息，服务器审核链接合法后下发给客户端的消息
                    if (invalidRelation(cliendId, targetId, ws)) return; // 鉴定是否为绑定关系

                    if (!data.channel) {
                        // 240531.现在必须指定通道(允许一次只覆盖一个正在播放的波形)
                        const data = { type: "error", clientId, targetId, message: "406-channel is empty" }
                        ws.send(JSON.stringify(data))
                        return;
                    }

                        //消息体 默认最少一个波形消息
                        let sendtime = data.time ? data.time : punishmentDuration; // AB通道的执行时间
                        const target = clients.get(targetId); //发送到目标app
                        const sendData = { type: "msg", clientId, targetId, message: "pulse-" + data.message }
                        let totalSends = punishmentTime * sendtime;
                        const timeSpace = 1000 / punishmentTime;

                        if (clientTimers.has(clientId + "-" + data.channel)) {
                            // A通道计时器尚未工作完毕, 清除计时器且发送清除APP队列消息，延迟150ms重新发送新数据
                            // 新消息覆盖旧消息逻辑，在多次触发波形输出的情况下，新的波形会覆盖旧的波形
                            console.log("通道" + data.channel + "覆盖消息发送中，总消息数：" + totalSends + "持续时间A：" + sendtime)
                            ws.send("当前通道" + data.channel + "有正在发送的消息，覆盖之前的消息")

                            const timerId = clientTimers.get(clientId + "-" + data.channel);
                            clearInterval(timerId); // 清除定时器
                            clientTimers.delete(clientId + "-" + data.channel); // 清除 Map 中的对应项

                            // 由于App中存在波形队列，保证波形的播放顺序正确，因此新波形覆盖旧波形之前需要发送APP波形队列清除指令
                            switch (data.channel) {
                                case "A":
                                    const clearDataA = { clientId, targetId, message: "clear-1", type: "msg" }
                                    target.send(JSON.stringify(clearDataA));
                                    break;

                                case "B":
                                    const clearDataB = { clientId, targetId, message: "clear-2", type: "msg" }
                                    target.send(JSON.stringify(clearDataB));
                                    break;
                                default:
                                    break;
                            }

                            setTimeout(() => {
                                delaySendMsg(clientId, ws, target, sendData, totalSends, timeSpace, data.channel);
                            }, 150);
                        }
                        else {
                            // 如果不存在未发完的波形消息，无需清除波形队列，直接发送
                            delaySendMsg(clientId, ws, target, sendData, totalSends, timeSpace, data.channel);
                            console.log("通道" + data.channel +"消息发送中，总消息数：" + totalSends + "持续时间：" + sendtime)
                        }

                    break;

                default:
                    // 未定义的其他消息，一般用作提示消息
                   if (invalidRelation(cliendId, targetId, ws)) return; // 鉴定是否为绑定关系

                        const client = clients.get(clientId);
                        const sendData = { type, clientId, targetId, message }
                        client.send(JSON.stringify(sendData));

                    break;
            }
        }
    });

    ws.on('close', function close() {
        // 连接关闭时，清除对应的 clientId 和 WebSocket 实例
        console.log('WebSocket 连接已关闭');
        // 遍历 clients Map，找到并删除对应的 clientId 条目
        let clientId = '';
        clients.forEach((value, key) => {
            if (value === ws) {
                // 拿到断开的客户端id
                clientId = key;
            }
        });
        console.log("断开的client id:" + clientId)
        relations.forEach((value, key) => {
            if (key === clientId) {
                //网页断开 通知app
                let appid = relations.get(key)
                let appClient = clients.get(appid)
                const data = { type: "break", clientId, targetId: appid, message: "209" }
                appClient.send(JSON.stringify(data))
                appClient.close(); // 关闭当前 WebSocket 连接
                relations.delete(key); // 清除关系
                console.log("对方掉线，关闭" + appid);
            }
            else if (value === clientId) {
                // app断开 通知网页
                let webClient = clients.get(key)
                const data = { type: "break", clientId: key, targetId: clientId, message: "209" }
                webClient.send(JSON.stringify(data))
                webClient.close(); // 关闭当前 WebSocket 连接
                relations.delete(key); // 清除关系
                console.log("对方掉线，关闭" + clientId);
            }
        })
        clients.delete(clientId); //清除ws客户端
        console.log("已清除" + clientId + " ,当前size: " + clients.size)
    });

    ws.on('error', function (error) {
        // 错误处理
        console.error('WebSocket 异常:', error.message);
        // 在此通知用户异常，通过 WebSocket 发送消息给双方
        let clientId = '';
        // 查找当前 WebSocket 实例对应的 clientId
        for (const [key, value] of clients.entries()) {
            if (value === ws) {
                clientId = key;
                break;
            }
        }
        if (!clientId) {
            console.error('无法找到对应的 clientId');
            return;
        }
        // 构造错误消息
        const errorMessage = 'WebSocket 异常: ' + error.message;

        relations.forEach((value, key) => {
            // 遍历关系 Map，找到并通知没掉线的那一方
            if (key === clientId) {
                // 通知app
                let appid = relations.get(key)
                let appClient = clients.get(appid)
                const data = { type: "error", clientId: clientId, targetId: appid, message: "500" }
                appClient.send(JSON.stringify(data))
            }
            if (value === clientId) {
                // 通知网页
                let webClient = clients.get(key)
                const data = { type: "error", clientId: key, targetId: clientId, message: errorMessage }
                webClient.send(JSON.stringify(data))
            }
        })
    });

    // 启动心跳定时器（如果尚未启动）
    if (!heartbeatInterval) {
        heartbeatInterval = setInterval(() => {
            // 遍历 clients Map（大于0个链接），向每个客户端发送心跳消息
            if (clients.size > 0) {
                console.log(relations.size, clients.size, '发送心跳消息：' + new Date().toLocaleString());
                clients.forEach((client, clientId) => {
                    heartbeatMsg.clientId = clientId;
                    heartbeatMsg.targetId = relations.get(clientId) || '';
                    client.send(JSON.stringify(heartbeatMsg));
                });
            }
        }, 60 * 1000); // 每分钟发送一次心跳消息
    }
});

function delaySendMsg(clientId, client, target, sendData, totalSends, timeSpace, channel) {
    // 发信计时器 通道会分别发送不同的消息和不同的数量 必须等全部发送完才会取消这个消息 新消息可以覆盖
    // 波形消息由这个计时器来控制按时间发送，波形长度1秒，比如默认输出波形5秒，就需要按顺序向app发送5次，timeSpace设置为1000ms
    // 如果您在前端定义的波形长度不是1秒，那么您就需要控制这个计时器发信的延迟timeSpace，防止波形被覆盖播放

    target.send(JSON.stringify(sendData)); // 计时器开始，立即发送第一次通道的消息
    totalSends--; // 发信总数
    if (totalSends > 0) {
        return new Promise((resolve, reject) => {
            // 按设定频率发送消息给特定的客户端
            const timerId = setInterval(() => {
                if (totalSends > 0) {
                    target.send(JSON.stringify(sendData));
                    totalSends--;
                }
                // 如果达到发信总数，则停止定时器
                if (totalSends <= 0) {
                    clearInterval(timerId);
                    client.send("发送完毕")
                    clientTimers.delete(clientId); // 删除对应的定时器
                    resolve();
                }
            }, timeSpace); // 每隔频率倒数触发一次定时器，一般和波形长度一致

            // 存储clientId与其对应的timerId和波形通道，为了下次收信时确认是否还有正在发送的波形，若有则覆盖，防止多个计时器争抢发信
            clientTimers.set(clientId + "-" + channel, timerId);
        });
    }
}

function invalidRelation(cliendId, targetId, ws) {
    // 关系合法性鉴定，clientId和targetId必须存在于客户端集合中，且在relation集合中绑定了关系
    if (relations.get(clientId) !== targetId) {
        const data = { type: "bind", clientId, targetId, message: "402" }
        ws.send(JSON.stringify(data))
        return true;
    }
    if (!clients.has(clientId) || !clients.has(targetId)) {
        console.log(`未找到匹配的客户端，clientId: ${clientId}`);
        const data = { type: "bind", clientId, targetId, message: "404" }
        ws.send(JSON.stringify(data))
        return true;
    }
    return false;
}

```
### ������CPU ǿ��ͬ��ָ��

- ǰ�˿ɷ��� `{ type: "cpuAuto", action: "start", channel, intervalMs, minStrength, maxStrength, smoothing }` �������������� CPU��ǿ��ӳ�䡣`action` �� `stop` ����ֹͬ����
- `channel` ֧�� `"both"`(Ĭ��)��`"A"`��`"B"`��`intervalMs` Ϊ�������ں��룻`minStrength`/`maxStrength` ����������䣻`smoothing` ȡֵ 0~0.99 ����ָ��ƽ��ϵ����
- ����˻��ȡ������ CPU �����ʲ��� APP ���� `strength-<channel>+2+<value>` ָ�ͬʱ����ҳ�ش� `type: "cpuAuto", message: "tick"`������������ UI չʾ��ǰ CPU% ��ӳ��ǿ�ȡ�
- ��δ�����Ϸ��󶨻᷵�� `message: "402"`�����߻�����ֹͣʱ���� `message: "stopped"` ���Զ�������ʱ����
