# AI画像生成ツール ⭐

**難易度**: 初級
**推奨時間**: 8-10時間
**技術スタック**: React, TypeScript, OpenAI DALL-E API

---

## 📋 概要

OpenAIのDALL-E APIを使って、テキストから画像を生成するアプリを作成します。

---

## 🎯 学習目標

- [ ] DALL-E APIの使い方
- [ ] プロンプト最適化
- [ ] 画像表示と保存
- [ ] API利用制限の考慮
- [ ] ローディング/エラー状態の管理

---

## 📝 機能要件

### 必須機能

1. **プロンプト入力**
   - テキストで画像の説明を入力
   - 生成ボタンで画像生成

2. **画像表示**
   - 生成された画像を表示
   - 画像サイズ選択（256x256, 512x512, 1024x1024）

3. **画像保存**
   - ダウンロードボタン
   - 履歴表示

---

## 💻 実装例

```typescript
import OpenAI from 'openai';

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});

async function generateImage(prompt: string, size: string = '512x512') {
  const response = await openai.images.generate({
    model: 'dall-e-3',
    prompt: prompt,
    n: 1,
    size: size as '256x256' | '512x512' | '1024x1024',
  });

  return response.data[0].url;
}
```

---

## ⚠️ 注意事項

- APIキーは絶対にフロントエンドに露出させない
- バックエンドを経由してAPIを呼び出す
- 利用コストに注意（1画像あたり約$0.02-0.04）

---

**最終更新**: 2025-10-22
