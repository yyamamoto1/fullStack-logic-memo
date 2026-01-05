# ECサイトバックエンド ⭐⭐

**難易度**: 中級
**推奨時間**: 20-25時間
**技術スタック**: Node.js, Express, PostgreSQL, Prisma, Stripe

---

## 概要

ECサイトのバックエンドAPIを構築します。データベース設計、商品管理、注文処理、在庫管理を学びます。

---

## 学習目標

- [ ] リレーショナルデータベース設計
- [ ] Prismaによるデータ操作
- [ ] トランザクション処理
- [ ] 在庫管理ロジック
- [ ] Stripe決済連携
- [ ] 注文ステータス管理

---

## 機能要件

### エンドポイント一覧

| メソッド | パス | 説明 | 認証 |
|---------|------|------|------|
| GET | /api/products | 商品一覧取得 | 不要 |
| GET | /api/products/:id | 商品詳細取得 | 不要 |
| POST | /api/products | 商品登録 | 管理者 |
| PUT | /api/products/:id | 商品更新 | 管理者 |
| DELETE | /api/products/:id | 商品削除 | 管理者 |
| POST | /api/orders | 注文作成 | 必要 |
| GET | /api/orders | 注文一覧取得 | 必要 |
| GET | /api/orders/:id | 注文詳細取得 | 必要 |
| POST | /api/checkout | 決済処理 | 必要 |

---

## データモデル

```prisma
model Product {
  id          String       @id @default(cuid())
  name        String
  description String       @db.Text
  price       Int
  comparePrice Int?
  images      String[]
  category    Category     @relation(fields: [categoryId], references: [id])
  categoryId  String
  variants    ProductVariant[]
  orderItems  OrderItem[]
  createdAt   DateTime     @default(now())
  updatedAt   DateTime     @updatedAt
}

model ProductVariant {
  id        String   @id @default(cuid())
  product   Product  @relation(fields: [productId], references: [id])
  productId String
  name      String   // "S / Red"
  sku       String   @unique
  stock     Int      @default(0)
  price     Int?     // バリアント固有の価格
  orderItems OrderItem[]
}

model Order {
  id              String      @id @default(cuid())
  orderNumber     String      @unique
  user            User        @relation(fields: [userId], references: [id])
  userId          String
  status          OrderStatus @default(PENDING)
  items           OrderItem[]
  subtotal        Int
  shippingFee     Int
  tax             Int
  total           Int
  shippingAddress Json
  paymentIntentId String?
  createdAt       DateTime    @default(now())
  updatedAt       DateTime    @updatedAt
}

model OrderItem {
  id        String          @id @default(cuid())
  order     Order           @relation(fields: [orderId], references: [id])
  orderId   String
  product   Product         @relation(fields: [productId], references: [id])
  productId String
  variant   ProductVariant? @relation(fields: [variantId], references: [id])
  variantId String?
  quantity  Int
  price     Int
}

enum OrderStatus {
  PENDING
  PAID
  PROCESSING
  SHIPPED
  DELIVERED
  CANCELLED
}
```

---

## 実装例

### 注文作成（トランザクション処理）

```typescript
import { prisma } from '../lib/prisma';
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

interface CreateOrderInput {
  userId: string;
  items: { productId: string; variantId?: string; quantity: number }[];
  shippingAddress: ShippingAddress;
}

export async function createOrder(input: CreateOrderInput) {
  const { userId, items, shippingAddress } = input;

  // トランザクションで注文と在庫を処理
  return prisma.$transaction(async (tx) => {
    // 1. 在庫確認と価格計算
    let subtotal = 0;
    const orderItems = [];

    for (const item of items) {
      const product = await tx.product.findUnique({
        where: { id: item.productId },
        include: { variants: true }
      });

      if (!product) {
        throw new Error(`Product not found: ${item.productId}`);
      }

      // バリアントがある場合は在庫確認
      if (item.variantId) {
        const variant = product.variants.find(v => v.id === item.variantId);
        if (!variant || variant.stock < item.quantity) {
          throw new Error(`Insufficient stock for ${product.name}`);
        }

        // 在庫を減らす
        await tx.productVariant.update({
          where: { id: item.variantId },
          data: { stock: { decrement: item.quantity } }
        });
      }

      const price = item.variantId
        ? product.variants.find(v => v.id === item.variantId)?.price ?? product.price
        : product.price;

      subtotal += price * item.quantity;
      orderItems.push({
        productId: item.productId,
        variantId: item.variantId,
        quantity: item.quantity,
        price
      });
    }

    // 2. 送料・税金計算
    const shippingFee = calculateShippingFee(subtotal);
    const tax = Math.floor(subtotal * 0.1);
    const total = subtotal + shippingFee + tax;

    // 3. Stripe PaymentIntent作成
    const paymentIntent = await stripe.paymentIntents.create({
      amount: total,
      currency: 'jpy',
      metadata: { userId }
    });

    // 4. 注文レコード作成
    const order = await tx.order.create({
      data: {
        orderNumber: generateOrderNumber(),
        userId,
        subtotal,
        shippingFee,
        tax,
        total,
        shippingAddress,
        paymentIntentId: paymentIntent.id,
        items: {
          create: orderItems
        }
      },
      include: {
        items: {
          include: { product: true, variant: true }
        }
      }
    });

    return {
      order,
      clientSecret: paymentIntent.client_secret
    };
  });
}
```

### Webhook処理

```typescript
import { stripe } from '../lib/stripe';

export async function handleStripeWebhook(payload: Buffer, sig: string) {
  const event = stripe.webhooks.constructEvent(
    payload,
    sig,
    process.env.STRIPE_WEBHOOK_SECRET!
  );

  switch (event.type) {
    case 'payment_intent.succeeded':
      const paymentIntent = event.data.object;
      await prisma.order.update({
        where: { paymentIntentId: paymentIntent.id },
        data: { status: 'PAID' }
      });
      break;

    case 'payment_intent.payment_failed':
      const failedPayment = event.data.object;
      await restoreStock(failedPayment.id);
      await prisma.order.update({
        where: { paymentIntentId: failedPayment.id },
        data: { status: 'CANCELLED' }
      });
      break;
  }
}
```

---

## 受け入れ基準

- [ ] 商品のCRUD操作ができる
- [ ] 在庫管理が正しく動作する
- [ ] 注文処理がトランザクションで実行される
- [ ] Stripe決済が動作する
- [ ] Webhookで決済結果を受け取れる
- [ ] 適切なエラーハンドリング

---

**最終更新**: 2025-10-22
