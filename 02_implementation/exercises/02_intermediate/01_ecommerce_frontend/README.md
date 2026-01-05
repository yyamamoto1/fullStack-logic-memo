# ECサイトフロントエンド ⭐⭐

**難易度**: 中級
**推奨時間**: 15-20時間
**技術スタック**: React, TypeScript, Zustand, React Router, Tailwind CSS, Stripe

---

## 概要

商品一覧・詳細・カート機能を持つECサイトのフロントエンドを構築します。状態管理とルーティングの実践的な使用方法を学びます。

---

## 学習目標

- [ ] React Routerによるルーティング
- [ ] Zustandによる状態管理
- [ ] Stripeによる決済フロー実装
- [ ] レスポンシブデザインの実践
- [ ] 商品フィルタリング・ソート機能
- [ ] カート機能の実装

---

## 機能要件

### 必須機能

1. **商品一覧ページ**
   - 商品カード表示（画像、名前、価格）
   - カテゴリフィルター
   - 価格帯フィルター
   - ソート機能（価格順、新着順）
   - ページネーション

2. **商品詳細ページ**
   - 商品画像ギャラリー
   - 商品説明
   - サイズ・カラー選択
   - 数量選択
   - カートに追加ボタン
   - 関連商品表示

3. **カート機能**
   - カート内商品一覧
   - 数量変更・削除
   - 小計・合計表示
   - チェックアウトボタン

4. **決済フロー**
   - Stripe Checkoutへの連携
   - 注文確認画面
   - 注文完了画面

---

## 技術仕様

### ディレクトリ構成

```
src/
├── components/
│   ├── common/
│   │   ├── Header.tsx
│   │   ├── Footer.tsx
│   │   └── Loading.tsx
│   ├── product/
│   │   ├── ProductCard.tsx
│   │   ├── ProductGrid.tsx
│   │   └── ProductGallery.tsx
│   └── cart/
│       ├── CartItem.tsx
│       ├── CartSummary.tsx
│       └── CartDrawer.tsx
├── pages/
│   ├── HomePage.tsx
│   ├── ProductListPage.tsx
│   ├── ProductDetailPage.tsx
│   ├── CartPage.tsx
│   └── CheckoutPage.tsx
├── stores/
│   ├── cartStore.ts
│   └── productStore.ts
├── hooks/
│   ├── useProducts.ts
│   └── useCart.ts
├── types/
│   └── index.ts
└── utils/
    ├── formatPrice.ts
    └── api.ts
```

### 状態管理（Zustand）

```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface CartItem {
  productId: string;
  name: string;
  price: number;
  quantity: number;
  size?: string;
  color?: string;
  image: string;
}

interface CartStore {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  removeItem: (productId: string) => void;
  updateQuantity: (productId: string, quantity: number) => void;
  clearCart: () => void;
  getTotalPrice: () => number;
  getTotalItems: () => number;
}

export const useCartStore = create<CartStore>()(
  persist(
    (set, get) => ({
      items: [],
      addItem: (item) =>
        set((state) => {
          const existing = state.items.find(
            (i) => i.productId === item.productId
          );
          if (existing) {
            return {
              items: state.items.map((i) =>
                i.productId === item.productId
                  ? { ...i, quantity: i.quantity + item.quantity }
                  : i
              ),
            };
          }
          return { items: [...state.items, item] };
        }),
      removeItem: (productId) =>
        set((state) => ({
          items: state.items.filter((i) => i.productId !== productId),
        })),
      updateQuantity: (productId, quantity) =>
        set((state) => ({
          items: state.items.map((i) =>
            i.productId === productId ? { ...i, quantity } : i
          ),
        })),
      clearCart: () => set({ items: [] }),
      getTotalPrice: () =>
        get().items.reduce((sum, item) => sum + item.price * item.quantity, 0),
      getTotalItems: () =>
        get().items.reduce((sum, item) => sum + item.quantity, 0),
    }),
    { name: 'cart-storage' }
  )
);
```

### ルーティング設定

```typescript
import { BrowserRouter, Routes, Route } from 'react-router-dom';

function App() {
  return (
    <BrowserRouter>
      <Layout>
        <Routes>
          <Route path="/" element={<HomePage />} />
          <Route path="/products" element={<ProductListPage />} />
          <Route path="/products/:id" element={<ProductDetailPage />} />
          <Route path="/cart" element={<CartPage />} />
          <Route path="/checkout" element={<CheckoutPage />} />
          <Route path="/checkout/success" element={<CheckoutSuccessPage />} />
        </Routes>
      </Layout>
    </BrowserRouter>
  );
}
```

---

## 受け入れ基準

- [ ] 全ページが正しくルーティングされる
- [ ] 商品の追加・削除・数量変更が動作する
- [ ] カートデータがページリロード後も保持される
- [ ] レスポンシブデザインが適用されている
- [ ] Stripe決済フローが動作する
- [ ] TypeScriptの型エラーがない

---

## テスト項目

```typescript
describe('ECサイトフロントエンド', () => {
  describe('商品一覧', () => {
    it('商品一覧が表示される');
    it('カテゴリフィルターが動作する');
    it('価格ソートが動作する');
  });

  describe('カート機能', () => {
    it('商品をカートに追加できる');
    it('カート内の商品数量を変更できる');
    it('カートから商品を削除できる');
    it('合計金額が正しく計算される');
  });

  describe('決済フロー', () => {
    it('チェックアウトに進める');
    it('Stripe決済が開始される');
  });
});
```

---

**最終更新**: 2025-10-22
