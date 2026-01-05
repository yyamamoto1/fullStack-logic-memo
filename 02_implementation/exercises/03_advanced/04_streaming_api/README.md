# リアルタイム配信API ⭐⭐⭐

**難易度**: 上級
**推奨時間**: 25-30時間
**技術スタック**: Node.js, WebSocket, Server-Sent Events, Redis Pub/Sub, nginx

---

## 概要

WebSocketとServer-Sent Eventsを使ったリアルタイム配信APIを構築します。負荷分散とスケーリングの実装方法を学びます。

---

## 学習目標

- [ ] WebSocket実装（ws/Socket.io）
- [ ] Server-Sent Events（SSE）実装
- [ ] Redis Pub/Subによるスケールアウト
- [ ] 負荷分散（nginx/HAProxy）
- [ ] 接続管理とハートビート
- [ ] バックプレッシャー対策

---

## アーキテクチャ

```
                    ┌─────────────────┐
                    │     nginx       │
                    │ (Load Balancer) │
                    └────────┬────────┘
                             │
           ┌─────────────────┼─────────────────┐
           ▼                 ▼                 ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │ Streaming    │ │ Streaming    │ │ Streaming    │
    │ Server #1    │ │ Server #2    │ │ Server #3    │
    └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
           │                │                │
           └────────────────┼────────────────┘
                            ▼
                    ┌──────────────┐
                    │    Redis     │
                    │   Pub/Sub    │
                    └──────────────┘
```

---

## 機能要件

### 必須機能

1. **WebSocket API**
   - リアルタイム双方向通信
   - ルーム/チャンネル管理
   - 認証・認可
   - 再接続処理

2. **SSE API**
   - サーバープッシュ
   - イベントストリーム
   - 自動再接続

3. **スケーリング**
   - 水平スケール対応
   - Redis Pub/Sub連携
   - Sticky Session

4. **監視・管理**
   - 接続数モニタリング
   - メッセージ統計
   - エラーログ

---

## 実装例

### WebSocket サーバー

```typescript
import { WebSocketServer, WebSocket } from 'ws';
import { createClient } from 'redis';
import jwt from 'jsonwebtoken';

const wss = new WebSocketServer({ port: 8080, path: '/ws' });
const redisSubscriber = createClient();
const redisPublisher = createClient();

interface Client {
  ws: WebSocket;
  userId: string;
  channels: Set<string>;
  lastPing: number;
}

const clients = new Map<string, Client>();
const channels = new Map<string, Set<string>>();

// 認証ミドルウェア
wss.on('connection', async (ws, req) => {
  const token = new URL(req.url!, 'http://localhost').searchParams.get('token');

  try {
    const decoded = jwt.verify(token!, process.env.JWT_SECRET!) as { userId: string };
    const clientId = generateId();

    const client: Client = {
      ws,
      userId: decoded.userId,
      channels: new Set(),
      lastPing: Date.now()
    };

    clients.set(clientId, client);

    // 接続成功通知
    ws.send(JSON.stringify({
      type: 'connected',
      clientId,
      userId: decoded.userId
    }));

    // メッセージハンドリング
    ws.on('message', (data) => handleMessage(clientId, data.toString()));
    ws.on('close', () => handleDisconnect(clientId));
    ws.on('pong', () => { client.lastPing = Date.now(); });

  } catch (error) {
    ws.close(4001, 'Unauthorized');
  }
});

function handleMessage(clientId: string, data: string) {
  const client = clients.get(clientId);
  if (!client) return;

  try {
    const message = JSON.parse(data);

    switch (message.type) {
      case 'subscribe':
        subscribeToChannel(clientId, message.channel);
        break;

      case 'unsubscribe':
        unsubscribeFromChannel(clientId, message.channel);
        break;

      case 'publish':
        publishToChannel(message.channel, {
          type: 'message',
          channel: message.channel,
          data: message.data,
          sender: client.userId,
          timestamp: Date.now()
        });
        break;
    }
  } catch (error) {
    client.ws.send(JSON.stringify({ type: 'error', message: 'Invalid message format' }));
  }
}

function subscribeToChannel(clientId: string, channel: string) {
  const client = clients.get(clientId);
  if (!client) return;

  client.channels.add(channel);

  if (!channels.has(channel)) {
    channels.set(channel, new Set());
    // Redis購読開始
    redisSubscriber.subscribe(channel, (message) => {
      broadcastToChannel(channel, message);
    });
  }

  channels.get(channel)!.add(clientId);

  client.ws.send(JSON.stringify({
    type: 'subscribed',
    channel
  }));
}

// マルチサーバー対応のパブリッシュ
async function publishToChannel(channel: string, message: any) {
  // Redisに発行（他サーバーにも配信される）
  await redisPublisher.publish(channel, JSON.stringify(message));
}

function broadcastToChannel(channel: string, message: string) {
  const subscribers = channels.get(channel);
  if (!subscribers) return;

  const data = JSON.parse(message);

  for (const clientId of subscribers) {
    const client = clients.get(clientId);
    if (client && client.ws.readyState === WebSocket.OPEN) {
      client.ws.send(message);
    }
  }
}

// ハートビート
setInterval(() => {
  const now = Date.now();
  for (const [clientId, client] of clients) {
    if (now - client.lastPing > 60000) {
      client.ws.terminate();
      handleDisconnect(clientId);
    } else {
      client.ws.ping();
    }
  }
}, 30000);
```

### Server-Sent Events

```typescript
import express from 'express';
import { createClient } from 'redis';

const app = express();
const redisSubscriber = createClient();

interface SSEClient {
  res: express.Response;
  userId: string;
  channels: Set<string>;
}

const sseClients = new Map<string, SSEClient>();

app.get('/events', async (req, res) => {
  // SSEヘッダー設定
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  res.setHeader('X-Accel-Buffering', 'no'); // nginx buffering無効

  // 認証
  const token = req.headers.authorization?.split(' ')[1];
  const user = await verifyToken(token);

  const clientId = generateId();
  const client: SSEClient = {
    res,
    userId: user.id,
    channels: new Set()
  };

  sseClients.set(clientId, client);

  // 接続確認イベント
  res.write(`event: connected\ndata: ${JSON.stringify({ clientId })}\n\n`);

  // ハートビート
  const heartbeat = setInterval(() => {
    res.write(': heartbeat\n\n');
  }, 30000);

  // 切断処理
  req.on('close', () => {
    clearInterval(heartbeat);
    sseClients.delete(clientId);
  });
});

// チャンネル購読エンドポイント
app.post('/events/subscribe', async (req, res) => {
  const { clientId, channel } = req.body;
  const client = sseClients.get(clientId);

  if (!client) {
    return res.status(404).json({ error: 'Client not found' });
  }

  if (!client.channels.has(channel)) {
    client.channels.add(channel);

    // Redis購読
    await redisSubscriber.subscribe(channel, (message) => {
      broadcastSSE(channel, message);
    });
  }

  res.json({ subscribed: channel });
});

function broadcastSSE(channel: string, message: string) {
  for (const [, client] of sseClients) {
    if (client.channels.has(channel)) {
      client.res.write(`event: ${channel}\ndata: ${message}\n\n`);
    }
  }
}

// イベント発行エンドポイント
app.post('/events/publish', async (req, res) => {
  const { channel, data } = req.body;

  await redisSubscriber.publish(channel, JSON.stringify({
    data,
    timestamp: Date.now()
  }));

  res.json({ published: true });
});
```

### nginx設定（負荷分散）

```nginx
upstream streaming_servers {
    ip_hash;  # Sticky session
    server streaming1:8080;
    server streaming2:8080;
    server streaming3:8080;
}

map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 80;

    # WebSocket
    location /ws {
        proxy_pass http://streaming_servers;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 86400;  # 24時間
        proxy_send_timeout 86400;
    }

    # SSE
    location /events {
        proxy_pass http://streaming_servers;
        proxy_http_version 1.1;
        proxy_set_header Connection '';
        proxy_set_header Cache-Control no-cache;
        proxy_buffering off;
        proxy_read_timeout 86400;
        chunked_transfer_encoding off;
    }
}
```

### クライアント実装

```typescript
// WebSocket クライアント
class WebSocketClient {
  private ws: WebSocket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;

  connect(token: string) {
    this.ws = new WebSocket(`wss://api.example.com/ws?token=${token}`);

    this.ws.onopen = () => {
      this.reconnectAttempts = 0;
      console.log('Connected');
    };

    this.ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      this.handleMessage(data);
    };

    this.ws.onclose = () => {
      this.reconnect(token);
    };

    this.ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };
  }

  private reconnect(token: string) {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      const delay = Math.min(1000 * Math.pow(2, this.reconnectAttempts), 30000);
      this.reconnectAttempts++;
      setTimeout(() => this.connect(token), delay);
    }
  }

  subscribe(channel: string) {
    this.ws?.send(JSON.stringify({ type: 'subscribe', channel }));
  }

  publish(channel: string, data: any) {
    this.ws?.send(JSON.stringify({ type: 'publish', channel, data }));
  }
}

// SSE クライアント
class SSEClient {
  private eventSource: EventSource | null = null;

  connect(token: string) {
    this.eventSource = new EventSource(`/events`, {
      // @ts-ignore - カスタムヘッダー対応にはfetch SSEライブラリが必要
    });

    this.eventSource.onopen = () => {
      console.log('SSE connected');
    };

    this.eventSource.addEventListener('connected', (event) => {
      console.log('Client ID:', JSON.parse(event.data).clientId);
    });

    this.eventSource.onerror = () => {
      // EventSourceは自動再接続する
      console.log('SSE reconnecting...');
    };
  }

  subscribe(channel: string, handler: (data: any) => void) {
    this.eventSource?.addEventListener(channel, (event) => {
      handler(JSON.parse(event.data));
    });
  }
}
```

---

## 受け入れ基準

- [ ] WebSocket接続が動作する
- [ ] SSE配信が動作する
- [ ] チャンネル購読/解除ができる
- [ ] 複数サーバーでスケールする
- [ ] 自動再接続が機能する
- [ ] ハートビートで切断検知できる

---

**最終更新**: 2025-10-22
