# マイクロサービス構築 ⭐⭐⭐

**難易度**: 上級
**推奨時間**: 35-45時間
**技術スタック**: Node.js, Docker, Kubernetes, gRPC, RabbitMQ, Redis

---

## 概要

モノリシックアプリをマイクロサービスに分割し、サービス間通信、API Gateway、分散トランザクションを実装します。

---

## 学習目標

- [ ] マイクロサービス設計原則
- [ ] Dockerコンテナ化
- [ ] Kubernetes基礎
- [ ] API Gateway（Kong/nginx）
- [ ] サービス間通信（gRPC/REST）
- [ ] メッセージキュー（RabbitMQ）
- [ ] 分散トレーシング
- [ ] サーキットブレーカー

---

## アーキテクチャ

```
                    ┌─────────────────┐
                    │   API Gateway   │
                    │     (Kong)      │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  User Service │   │ Product Svc   │   │  Order Svc    │
│   (Express)   │   │  (Express)    │   │  (Express)    │
└───────┬───────┘   └───────┬───────┘   └───────┬───────┘
        │                   │                   │
        ▼                   ▼                   ▼
  ┌──────────┐        ┌──────────┐        ┌──────────┐
  │ MongoDB  │        │PostgreSQL│        │PostgreSQL│
  └──────────┘        └──────────┘        └──────────┘

              ┌─────────────────────────┐
              │       RabbitMQ          │
              │    (Event Bus)          │
              └─────────────────────────┘
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
    ┌──────────┐   ┌──────────┐   ┌──────────┐
    │Notification│ │ Inventory│   │ Payment  │
    │  Service  │  │  Service │   │ Service  │
    └──────────┘   └──────────┘   └──────────┘
```

---

## サービス構成

### 1. User Service
- ユーザー登録・認証
- プロフィール管理
- JWT発行

### 2. Product Service
- 商品CRUD
- カテゴリ管理
- 検索機能

### 3. Order Service
- 注文作成・管理
- 注文ステータス更新
- 注文履歴

### 4. Inventory Service
- 在庫管理
- 在庫予約
- 在庫更新イベント

### 5. Payment Service
- 決済処理
- 返金処理
- 決済履歴

### 6. Notification Service
- メール送信
- プッシュ通知
- SMS通知

---

## 実装例

### API Gateway設定（Kong）

```yaml
# kong.yml
_format_version: "2.1"

services:
  - name: user-service
    url: http://user-service:3001
    routes:
      - name: user-routes
        paths:
          - /api/users
          - /api/auth

  - name: product-service
    url: http://product-service:3002
    routes:
      - name: product-routes
        paths:
          - /api/products
          - /api/categories

  - name: order-service
    url: http://order-service:3003
    routes:
      - name: order-routes
        paths:
          - /api/orders

plugins:
  - name: rate-limiting
    config:
      minute: 100
      policy: local

  - name: jwt
    config:
      secret_is_base64: false
      claims_to_verify:
        - exp

  - name: cors
    config:
      origins:
        - http://localhost:3000
      methods:
        - GET
        - POST
        - PUT
        - DELETE
```

### サービス間通信（gRPC）

```protobuf
// inventory.proto
syntax = "proto3";

package inventory;

service InventoryService {
  rpc CheckStock(CheckStockRequest) returns (CheckStockResponse);
  rpc ReserveStock(ReserveStockRequest) returns (ReserveStockResponse);
  rpc ReleaseStock(ReleaseStockRequest) returns (ReleaseStockResponse);
}

message CheckStockRequest {
  string product_id = 1;
  int32 quantity = 2;
}

message CheckStockResponse {
  bool available = 1;
  int32 current_stock = 2;
}

message ReserveStockRequest {
  string product_id = 1;
  int32 quantity = 2;
  string order_id = 3;
}

message ReserveStockResponse {
  bool success = 1;
  string reservation_id = 2;
  string error = 3;
}
```

```typescript
// gRPC クライアント
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';

const packageDefinition = protoLoader.loadSync('inventory.proto');
const inventoryProto = grpc.loadPackageDefinition(packageDefinition).inventory;

const inventoryClient = new inventoryProto.InventoryService(
  'inventory-service:50051',
  grpc.credentials.createInsecure()
);

async function checkStock(productId: string, quantity: number): Promise<boolean> {
  return new Promise((resolve, reject) => {
    inventoryClient.CheckStock({ product_id: productId, quantity }, (error, response) => {
      if (error) reject(error);
      else resolve(response.available);
    });
  });
}
```

### イベント駆動通信（RabbitMQ）

```typescript
// event-bus.ts
import amqp from 'amqplib';

class EventBus {
  private connection: amqp.Connection;
  private channel: amqp.Channel;

  async connect() {
    this.connection = await amqp.connect(process.env.RABBITMQ_URL);
    this.channel = await this.connection.createChannel();
  }

  async publish(exchange: string, routingKey: string, message: any) {
    await this.channel.assertExchange(exchange, 'topic', { durable: true });
    this.channel.publish(
      exchange,
      routingKey,
      Buffer.from(JSON.stringify(message)),
      { persistent: true }
    );
  }

  async subscribe(exchange: string, routingKey: string, handler: (msg: any) => Promise<void>) {
    await this.channel.assertExchange(exchange, 'topic', { durable: true });
    const { queue } = await this.channel.assertQueue('', { exclusive: true });
    await this.channel.bindQueue(queue, exchange, routingKey);

    this.channel.consume(queue, async (msg) => {
      if (msg) {
        try {
          const content = JSON.parse(msg.content.toString());
          await handler(content);
          this.channel.ack(msg);
        } catch (error) {
          this.channel.nack(msg, false, true);
        }
      }
    });
  }
}

// Order Service: 注文作成イベント発行
orderEventBus.publish('orders', 'order.created', {
  orderId: order.id,
  userId: order.userId,
  items: order.items,
  total: order.total,
  createdAt: new Date().toISOString()
});

// Notification Service: 注文作成イベント購読
notificationEventBus.subscribe('orders', 'order.created', async (event) => {
  await sendOrderConfirmationEmail(event.userId, event.orderId);
});
```

### サーキットブレーカー

```typescript
import CircuitBreaker from 'opossum';

const circuitBreakerOptions = {
  timeout: 3000,
  errorThresholdPercentage: 50,
  resetTimeout: 30000
};

const inventoryBreaker = new CircuitBreaker(
  async (productId: string, quantity: number) => {
    return await inventoryClient.checkStock(productId, quantity);
  },
  circuitBreakerOptions
);

inventoryBreaker.on('open', () => {
  console.log('Circuit breaker opened - inventory service unavailable');
});

inventoryBreaker.on('halfOpen', () => {
  console.log('Circuit breaker half-open - testing inventory service');
});

inventoryBreaker.fallback(() => ({
  available: false,
  message: 'Inventory service temporarily unavailable'
}));
```

### Saga パターン（分散トランザクション）

```typescript
// order-saga.ts
class OrderSaga {
  private steps: SagaStep[] = [];
  private completedSteps: SagaStep[] = [];

  async execute(orderData: CreateOrderData) {
    try {
      // Step 1: 在庫予約
      const reservation = await this.reserveInventory(orderData.items);
      this.completedSteps.push({ type: 'RESERVE_INVENTORY', data: reservation });

      // Step 2: 決済処理
      const payment = await this.processPayment(orderData.userId, orderData.total);
      this.completedSteps.push({ type: 'PROCESS_PAYMENT', data: payment });

      // Step 3: 注文作成
      const order = await this.createOrder(orderData);
      this.completedSteps.push({ type: 'CREATE_ORDER', data: order });

      // Step 4: 在庫確定
      await this.confirmInventory(reservation.reservationId);

      return order;
    } catch (error) {
      // 補償トランザクション実行
      await this.compensate();
      throw error;
    }
  }

  private async compensate() {
    for (const step of this.completedSteps.reverse()) {
      switch (step.type) {
        case 'RESERVE_INVENTORY':
          await this.releaseInventory(step.data.reservationId);
          break;
        case 'PROCESS_PAYMENT':
          await this.refundPayment(step.data.paymentId);
          break;
        case 'CREATE_ORDER':
          await this.cancelOrder(step.data.orderId);
          break;
      }
    }
  }
}
```

---

## Docker Compose

```yaml
version: '3.8'

services:
  api-gateway:
    image: kong:latest
    ports:
      - "8000:8000"
      - "8001:8001"
    volumes:
      - ./kong.yml:/usr/local/kong/declarative/kong.yml
    environment:
      KONG_DATABASE: "off"
      KONG_DECLARATIVE_CONFIG: /usr/local/kong/declarative/kong.yml

  user-service:
    build: ./services/user
    environment:
      - MONGODB_URI=mongodb://mongodb:27017/users
    depends_on:
      - mongodb

  product-service:
    build: ./services/product
    environment:
      - DATABASE_URL=postgresql://postgres:password@postgres:5432/products
    depends_on:
      - postgres

  order-service:
    build: ./services/order
    environment:
      - DATABASE_URL=postgresql://postgres:password@postgres:5432/orders
      - RABBITMQ_URL=amqp://rabbitmq:5672
    depends_on:
      - postgres
      - rabbitmq

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"

  mongodb:
    image: mongo:6
    volumes:
      - mongodb_data:/data/db

  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  mongodb_data:
  postgres_data:
```

---

## 受け入れ基準

- [ ] 各サービスが独立してデプロイできる
- [ ] API Gatewayでルーティングされる
- [ ] サービス間通信が動作する
- [ ] イベント駆動通信が機能する
- [ ] サーキットブレーカーが動作する
- [ ] 分散トランザクションが正しく処理される

---

**最終更新**: 2025-10-22
