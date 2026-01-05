# REST API基礎 ⭐

**難易度**: 初級
**推奨時間**: 10-12時間
**技術スタック**: Node.js, Express, TypeScript

---

## 📋 概要

Express.jsを使って基本的なREST APIを構築します。CRUD操作、ミドルウェア、エラーハンドリングの基礎を学びます。

---

## 🎯 学習目標

- [ ] Express.jsの基本
- [ ] REST APIの設計原則
- [ ] CRUD操作の実装
- [ ] ミドルウェアの理解
- [ ] エラーハンドリング
- [ ] リクエストバリデーション

---

## 📝 機能要件

### エンドポイント

| メソッド | パス | 説明 |
|---------|------|------|
| GET | /api/users | ユーザー一覧取得 |
| GET | /api/users/:id | ユーザー詳細取得 |
| POST | /api/users | ユーザー作成 |
| PUT | /api/users/:id | ユーザー更新 |
| DELETE | /api/users/:id | ユーザー削除 |

---

## 💻 実装例

```typescript
import express from 'express';

const app = express();
app.use(express.json());

interface User {
  id: number;
  name: string;
  email: string;
}

let users: User[] = [];
let nextId = 1;

// 一覧取得
app.get('/api/users', (req, res) => {
  res.json(users);
});

// 詳細取得
app.get('/api/users/:id', (req, res) => {
  const user = users.find(u => u.id === parseInt(req.params.id));
  if (!user) {
    return res.status(404).json({ error: 'User not found' });
  }
  res.json(user);
});

// 作成
app.post('/api/users', (req, res) => {
  const { name, email } = req.body;
  const user: User = { id: nextId++, name, email };
  users.push(user);
  res.status(201).json(user);
});

app.listen(8000, () => {
  console.log('Server running on port 8000');
});
```

---

## ✅ 受け入れ基準

- [ ] 全CRUDエンドポイントが動作する
- [ ] 適切なHTTPステータスコードを返す
- [ ] エラー時は適切なエラーレスポンスを返す
- [ ] バリデーションが機能する
- [ ] TypeScriptの型が正しく定義されている

---

**最終更新**: 2025-10-22
