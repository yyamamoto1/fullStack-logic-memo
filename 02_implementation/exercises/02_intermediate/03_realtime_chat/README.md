# リアルタイムチャット ⭐⭐

**難易度**: 中級
**推奨時間**: 15-20時間
**技術スタック**: React, Socket.io, Node.js, Express, Redis

---

## 概要

複数ルーム対応のリアルタイムチャットアプリを構築します。WebSocket通信の基礎と実践的な使用方法を学びます。

---

## 学習目標

- [ ] WebSocket通信の仕組み
- [ ] Socket.ioによるリアルタイム通信
- [ ] Redisによるセッション管理
- [ ] 複数ルーム管理
- [ ] タイピングインジケータ
- [ ] オンラインステータス管理

---

## 機能要件

### 必須機能

1. **認証機能**
   - ニックネーム設定
   - ログイン/ログアウト
   - セッション管理

2. **ルーム機能**
   - ルーム一覧表示
   - ルーム作成
   - ルーム参加/退出
   - ルーム内メンバー表示

3. **メッセージ機能**
   - テキストメッセージ送信
   - リアルタイム受信
   - メッセージ履歴
   - タイピングインジケータ

4. **通知機能**
   - 新着メッセージ通知
   - 入退室通知
   - 未読カウント

---

## アーキテクチャ

```
┌─────────────┐     WebSocket     ┌─────────────┐
│   Client    │◄─────────────────►│   Server    │
│   (React)   │                   │  (Express)  │
└─────────────┘                   └──────┬──────┘
                                         │
                                         ▼
                                  ┌─────────────┐
                                  │    Redis    │
                                  │ (Pub/Sub)   │
                                  └─────────────┘
```

---

## 実装例

### サーバー側（Socket.io）

```typescript
import express from 'express';
import { createServer } from 'http';
import { Server } from 'socket.io';
import { createClient } from 'redis';

const app = express();
const httpServer = createServer(app);
const io = new Server(httpServer, {
  cors: { origin: 'http://localhost:3000' }
});

const redisClient = createClient();

io.on('connection', (socket) => {
  console.log('User connected:', socket.id);

  // ルームに参加
  socket.on('join_room', async ({ roomId, username }) => {
    socket.join(roomId);
    socket.data.username = username;
    socket.data.roomId = roomId;

    // ルームメンバーを更新
    await redisClient.sAdd(`room:${roomId}:members`, username);

    // 入室通知
    io.to(roomId).emit('user_joined', { username, roomId });

    // メンバーリスト送信
    const members = await redisClient.sMembers(`room:${roomId}:members`);
    io.to(roomId).emit('room_members', members);
  });

  // メッセージ送信
  socket.on('send_message', async ({ roomId, message }) => {
    const messageData = {
      id: generateId(),
      username: socket.data.username,
      message,
      timestamp: new Date().toISOString()
    };

    // メッセージを保存
    await redisClient.lPush(
      `room:${roomId}:messages`,
      JSON.stringify(messageData)
    );

    // ルーム内にブロードキャスト
    io.to(roomId).emit('new_message', messageData);
  });

  // タイピング中
  socket.on('typing', ({ roomId }) => {
    socket.to(roomId).emit('user_typing', {
      username: socket.data.username
    });
  });

  // 切断処理
  socket.on('disconnect', async () => {
    const { username, roomId } = socket.data;
    if (roomId) {
      await redisClient.sRem(`room:${roomId}:members`, username);
      io.to(roomId).emit('user_left', { username, roomId });
    }
  });
});

httpServer.listen(8000);
```

### クライアント側（React）

```typescript
import { useEffect, useState, useCallback } from 'react';
import { io, Socket } from 'socket.io-client';

const socket = io('http://localhost:8000');

interface Message {
  id: string;
  username: string;
  message: string;
  timestamp: string;
}

export function useChat(roomId: string, username: string) {
  const [messages, setMessages] = useState<Message[]>([]);
  const [members, setMembers] = useState<string[]>([]);
  const [typingUsers, setTypingUsers] = useState<string[]>([]);

  useEffect(() => {
    socket.emit('join_room', { roomId, username });

    socket.on('new_message', (message: Message) => {
      setMessages(prev => [...prev, message]);
    });

    socket.on('room_members', (memberList: string[]) => {
      setMembers(memberList);
    });

    socket.on('user_typing', ({ username }: { username: string }) => {
      setTypingUsers(prev => [...new Set([...prev, username])]);
      setTimeout(() => {
        setTypingUsers(prev => prev.filter(u => u !== username));
      }, 2000);
    });

    return () => {
      socket.emit('leave_room', { roomId });
      socket.off('new_message');
      socket.off('room_members');
      socket.off('user_typing');
    };
  }, [roomId, username]);

  const sendMessage = useCallback((message: string) => {
    socket.emit('send_message', { roomId, message });
  }, [roomId]);

  const sendTyping = useCallback(() => {
    socket.emit('typing', { roomId });
  }, [roomId]);

  return { messages, members, typingUsers, sendMessage, sendTyping };
}
```

---

## 受け入れ基準

- [ ] リアルタイムでメッセージが送受信できる
- [ ] 複数ルームに対応している
- [ ] タイピングインジケータが動作する
- [ ] オンラインメンバーが表示される
- [ ] 入退室通知が表示される
- [ ] ページリロード後も動作する

---

**最終更新**: 2025-10-22
