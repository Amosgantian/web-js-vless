const os = require('os');
const http = require('http');
const { Buffer } = require('buffer');
const fs = require('fs');
const axios = require('axios');
const path = require('path');
const net = require('net');
const { WebSocket, createWebSocketStream } = require('ws');

const logcb = (...args) => console.log.bind(this, ...args);
const errcb = (...args) => console.error.bind(this, ...args);

// 从环境变量中获取配置，设置默认值
const UUID = process.env.UUID || '43836a55-3f1c-4054-bd63-7e70387fa9ce'; // 默认 UUID，建议修改
const uuid = UUID.replace(/-/g, "");
const DOMAIN = process.env.DOMAIN || 'example.com'; // 默认域名，建议修改
const NAME = process.env.NAME || 'My-Websocket-Proxy'; // 默认名称，建议修改
const PORT = process.env.PORT || 5000; // 默认端口，建议修改

// 验证 UUID 格式
function isValidUUID(uuid) {
    const uuidRegex = /^[0-9a-f]{8}[0-9a-f]{4}[0-9a-f]{4}[0-9a-f]{4}[0-9a-f]{12}$/i;
    return uuidRegex.test(uuid);
}

if (!isValidUUID(uuid)) {
    console.error("Invalid UUID format. Please set a valid UUID in the environment variables.");
    process.exit(1);
}

// 创建HTTP路由
const httpServer = http.createServer((req, res) => {
    if (req.url === '/') {
        res.writeHead(200, { 'Content-Type': 'text/plain' });
        res.end('Hello, World!\n');
    } else if (req.url === '/sub') {
        // 生成 VLESS URL
        const vlessURL = `vless://${UUID}@${DOMAIN}:443?encryption=none&security=tls&sni=${DOMAIN}&type=ws&host=${DOMAIN}&path=%2F#${NAME}`;

        // Base64 编码
        const base64Content = Buffer.from(vlessURL).toString('base64');

        res.writeHead(200, { 'Content-Type': 'text/plain' });
        res.end(base64Content + '\n');
    } else {
        res.writeHead(404, { 'Content-Type': 'text/plain' });
        res.end('Not Found\n');
    }
});

httpServer.on('error', (err) => {
    console.error('HTTP server error:', err);
});

httpServer.listen(PORT, () => {
    console.log(`HTTP Server is running on port ${PORT}`);
});

// WebSocket 服务器
const wss = new WebSocket.Server({ server: httpServer });

wss.on('listening', () => {
    console.log('WebSocket server is listening');
});

wss.on('connection', ws => {
    console.log("WebSocket 连接成功");

    ws.on('message', msg => {
        if (msg.length < 18) {
            console.error("数据长度无效");
            ws.close(1002, "Invalid data length"); // Close the connection with an error code
            return;
        }
        try {
            const [VERSION] = msg;
            const id = msg.slice(1, 17);

            // UUID 验证
            if (!id.every((v, i) => v == parseInt(uuid.substr(i * 2, 2), 16))) {
                console.error("UUID 验证失败");
                ws.close(1002, "UUID verification failed"); // Close the connection with an error code
                return;
            }

            let i = msg.slice(17, 18).readUInt8() + 19;
            const port = msg.slice(i, i += 2).readUInt16BE(0);
            const ATYP = msg.slice(i, i += 1).readUInt8();

            let host;
            if (ATYP === 1) {
                host = msg.slice(i, i += 4).join('.');
            } else if (ATYP === 3) {
                host = new TextDecoder().decode(msg.slice(i + 1, i += 1 + msg.slice(i, i + 1).readUInt8()));
            } else if (ATYP === 4) {
                host = msg.slice(i, i += 16).reduce((s, b, i, a) => (i % 2 ? s.concat(a.slice(i - 1, i + 1)) : s), []).map(b => b.readUInt16BE(0).toString(16)).join(':');
            } else {
                console.error("Unsupported ATYP:", ATYP);
                ws.close(1002, "Unsupported ATYP");
                return;
            }

            console.log('Connecting to:', host, port);
            ws.send(new Uint8Array([VERSION, 0]));

            const duplex = createWebSocketStream(ws);
            const tcpSocket = net.connect({ host, port }, () => {
                tcpSocket.write(msg.slice(i));
                duplex.on('error', err => {
                    console.error("WebSocket stream error:", err.message);
                    tcpSocket.destroy(err);  // Destroy the TCP socket on WebSocket error
                }).pipe(tcpSocket).on('error', err => {
                    console.error("TCP socket error:", err.message);
                    duplex.destroy(err); // Destroy the WebSocket stream on TCP error
                }).pipe(duplex);
            });

            tcpSocket.on('error', err => {
                console.error("TCP connection error:", err.message);
                ws.close(1011, `TCP connection error: ${err.message}`); // Close the WebSocket with an error code
                duplex.destroy(err); // Clean up WebSocket stream
            });

            tcpSocket.on('close', () => {
                duplex.destroy(); // Clean up WebSocket stream when TCP connection closes
            });

            duplex.on('close', () => {
                tcpSocket.destroy(); // Clean up TCP socket when WebSocket closes
            });

        } catch (err) {
            console.error("Error processing message:", err.message);
            ws.close(1011, `Error processing message: ${err.message}`); // Close the connection with an error code
        }
    });

    ws.on('error', err => {
        console.error("WebSocket error:", err.message);
    });

    ws.on('close', (code, reason) => {
        console.log(`WebSocket closed: code=${code}, reason=${reason}`);
    });
});

wss.on('error', (err) => {
    console.error('WebSocket server error:', err);
});

console.log('Application started.');
