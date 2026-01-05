# ユーザー認証API ⭐

**難易度**: 初級
**推奨時間**: 10-12時間
**技術スタック**: Node.js, Express, JWT, bcrypt

---

## 📋 概要

JWT（JSON Web Token）を使った認証システムを構築します。セキュアな認証の基礎を学びます。

---

## 🎯 学習目標

- [ ] JWTによる認証の仕組み
- [ ] パスワードのハッシュ化（bcrypt）
- [ ] 認証ミドルウェアの実装
- [ ] トークンの発行と検証
- [ ] セキュリティのベストプラクティス

---

## 📝 機能要件

### エンドポイント

| メソッド | パス | 説明 | 認証 |
|---------|------|------|------|
| POST | /api/auth/register | ユーザー登録 | 不要 |
| POST | /api/auth/login | ログイン | 不要 |
| GET | /api/auth/me | 自分の情報取得 | 必要 |
| POST | /api/auth/logout | ログアウト | 必要 |

---

## 💻 実装例

```typescript
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';

// 登録
app.post('/api/auth/register', async (req, res) => {
  const { email, password, name } = req.body;

  // パスワードハッシュ化
  const hashedPassword = await bcrypt.hash(password, 10);

  // ユーザー作成
  const user = { id: nextId++, email, name, password: hashedPassword };
  users.push(user);

  // トークン発行
  const token = jwt.sign({ userId: user.id }, process.env.JWT_SECRET!, {
    expiresIn: '7d',
  });

  res.status(201).json({ user: { id: user.id, email, name }, token });
});

// ログイン
app.post('/api/auth/login', async (req, res) => {
  const { email, password } = req.body;

  const user = users.find(u => u.email === email);
  if (!user) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  const isValid = await bcrypt.compare(password, user.password);
  if (!isValid) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  const token = jwt.sign({ userId: user.id }, process.env.JWT_SECRET!, {
    expiresIn: '7d',
  });

  res.json({ token });
});

// 認証ミドルウェア
const authMiddleware = (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET!);
    req.userId = decoded.userId;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};
```

---

**最終更新**: 2025-10-22
