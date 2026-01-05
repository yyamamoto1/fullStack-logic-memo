# プロジェクト管理ツール ⭐⭐⭐

**難易度**: 上級
**推奨時間**: 30-40時間
**技術スタック**: React, TypeScript, Next.js, Prisma, WebSocket, PWA

---

## 概要

Jira/Trelloのようなプロジェクト管理ツールを構築します。リアルタイムコラボレーション、複雑な状態管理、PWA対応を学びます。

---

## 学習目標

- [ ] 複雑な状態管理（Zustand + React Query）
- [ ] ドラッグ&ドロップ（dnd-kit）
- [ ] リアルタイムコラボレーション（WebSocket）
- [ ] オプティミスティック更新
- [ ] PWA（Progressive Web App）対応
- [ ] 権限管理システム

---

## 機能要件

### 必須機能

1. **プロジェクト管理**
   - プロジェクト作成・編集・削除
   - メンバー招待・権限設定
   - プロジェクト設定

2. **ボード機能（カンバン）**
   - カラム作成・編集・削除
   - タスクのドラッグ&ドロップ
   - WIP制限設定

3. **タスク管理**
   - タスク作成・編集・削除
   - サブタスク
   - ラベル・優先度・担当者設定
   - 期限設定・リマインダー
   - 添付ファイル・コメント

4. **リアルタイム同期**
   - 複数ユーザーの同時編集
   - リアルタイム更新通知
   - プレゼンス表示（誰がオンラインか）

5. **PWA対応**
   - オフライン対応
   - プッシュ通知
   - ホーム画面追加

---

## アーキテクチャ

```
┌──────────────────────────────────────────────────┐
│                   Client (PWA)                    │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐  │
│  │   Zustand   │  │ React Query │  │  Service │  │
│  │   (State)   │  │  (Server)   │  │  Worker  │  │
│  └─────────────┘  └─────────────┘  └──────────┘  │
└────────────────────────┬─────────────────────────┘
                         │ REST + WebSocket
┌────────────────────────┴─────────────────────────┐
│                   API Server                      │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐  │
│  │   Express   │  │  Socket.io  │  │ Bull Queue │ │
│  └─────────────┘  └─────────────┘  └──────────┘  │
└────────────────────────┬─────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
   ┌──────────┐   ┌──────────┐   ┌──────────┐
   │PostgreSQL│   │  Redis   │   │   S3     │
   └──────────┘   └──────────┘   └──────────┘
```

---

## データモデル

```prisma
model Project {
  id          String       @id @default(cuid())
  name        String
  description String?
  key         String       @unique // PRJ-001形式
  owner       User         @relation("OwnedProjects", fields: [ownerId], references: [id])
  ownerId     String
  members     ProjectMember[]
  boards      Board[]
  createdAt   DateTime     @default(now())
  updatedAt   DateTime     @updatedAt
}

model ProjectMember {
  id        String   @id @default(cuid())
  project   Project  @relation(fields: [projectId], references: [id])
  projectId String
  user      User     @relation(fields: [userId], references: [id])
  userId    String
  role      ProjectRole @default(MEMBER)

  @@unique([projectId, userId])
}

enum ProjectRole {
  ADMIN
  MEMBER
  VIEWER
}

model Board {
  id        String   @id @default(cuid())
  name      String
  project   Project  @relation(fields: [projectId], references: [id])
  projectId String
  columns   Column[]
  createdAt DateTime @default(now())
}

model Column {
  id       String @id @default(cuid())
  name     String
  position Int
  wipLimit Int?
  board    Board  @relation(fields: [boardId], references: [id])
  boardId  String
  tasks    Task[]
}

model Task {
  id          String     @id @default(cuid())
  key         String     @unique // PRJ-001形式
  title       String
  description String?    @db.Text
  position    Int
  priority    Priority   @default(MEDIUM)
  status      TaskStatus @default(TODO)
  column      Column     @relation(fields: [columnId], references: [id])
  columnId    String
  assignee    User?      @relation("AssignedTasks", fields: [assigneeId], references: [id])
  assigneeId  String?
  reporter    User       @relation("ReportedTasks", fields: [reporterId], references: [id])
  reporterId  String
  parentTask  Task?      @relation("Subtasks", fields: [parentId], references: [id])
  parentId    String?
  subtasks    Task[]     @relation("Subtasks")
  labels      Label[]
  comments    Comment[]
  attachments Attachment[]
  dueDate     DateTime?
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt
}

enum Priority {
  CRITICAL
  HIGH
  MEDIUM
  LOW
}

enum TaskStatus {
  TODO
  IN_PROGRESS
  IN_REVIEW
  DONE
}
```

---

## 実装例

### ドラッグ&ドロップ

```typescript
import { DndContext, DragEndEvent, closestCorners } from '@dnd-kit/core';
import { SortableContext, arrayMove } from '@dnd-kit/sortable';

function KanbanBoard({ boardId }: { boardId: string }) {
  const { data: board, mutate } = useBoardData(boardId);
  const socket = useSocket();

  const handleDragEnd = async (event: DragEndEvent) => {
    const { active, over } = event;

    if (!over) return;

    const activeTask = findTask(active.id);
    const overColumn = findColumn(over.id);

    if (!activeTask || !overColumn) return;

    // オプティミスティック更新
    const optimisticUpdate = {
      ...board,
      columns: board.columns.map(col => ({
        ...col,
        tasks: col.id === overColumn.id
          ? [...col.tasks, activeTask]
          : col.tasks.filter(t => t.id !== activeTask.id)
      }))
    };

    mutate(optimisticUpdate, false);

    // サーバーに送信
    try {
      await moveTask({
        taskId: activeTask.id,
        columnId: overColumn.id,
        position: calculatePosition(overColumn.tasks, over.id)
      });

      // リアルタイム通知
      socket.emit('task:moved', {
        taskId: activeTask.id,
        columnId: overColumn.id,
        boardId
      });
    } catch (error) {
      // ロールバック
      mutate();
    }
  };

  return (
    <DndContext collisionDetection={closestCorners} onDragEnd={handleDragEnd}>
      <div className="flex gap-4 overflow-x-auto p-4">
        {board?.columns.map(column => (
          <KanbanColumn key={column.id} column={column} />
        ))}
      </div>
    </DndContext>
  );
}
```

### リアルタイム同期

```typescript
// サーバー側
io.on('connection', (socket) => {
  socket.on('board:join', (boardId) => {
    socket.join(`board:${boardId}`);
    socket.to(`board:${boardId}`).emit('user:joined', {
      userId: socket.data.userId,
      username: socket.data.username
    });
  });

  socket.on('task:moved', async (data) => {
    // DBを更新
    await updateTaskPosition(data);

    // 他のユーザーに通知
    socket.to(`board:${data.boardId}`).emit('task:updated', data);
  });

  socket.on('task:editing', (data) => {
    socket.to(`board:${data.boardId}`).emit('task:being_edited', {
      taskId: data.taskId,
      userId: socket.data.userId
    });
  });
});
```

### Service Worker（PWA）

```typescript
// service-worker.ts
import { precacheAndRoute } from 'workbox-precaching';
import { registerRoute } from 'workbox-routing';
import { NetworkFirst, CacheFirst } from 'workbox-strategies';

precacheAndRoute(self.__WB_MANIFEST);

// API呼び出しはネットワーク優先
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/'),
  new NetworkFirst({
    cacheName: 'api-cache',
    networkTimeoutSeconds: 10
  })
);

// 静的アセットはキャッシュ優先
registerRoute(
  ({ request }) => request.destination === 'image',
  new CacheFirst({
    cacheName: 'image-cache'
  })
);

// オフライン時のフォールバック
self.addEventListener('fetch', (event) => {
  if (event.request.mode === 'navigate') {
    event.respondWith(
      fetch(event.request).catch(() => caches.match('/offline.html'))
    );
  }
});
```

---

## 受け入れ基準

- [ ] プロジェクト・ボード・タスクのCRUDが動作する
- [ ] ドラッグ&ドロップでタスクを移動できる
- [ ] 複数ユーザーがリアルタイムで同期される
- [ ] オフラインでも基本操作ができる
- [ ] プッシュ通知が動作する
- [ ] 権限管理が正しく機能する

---

**最終更新**: 2025-10-22
