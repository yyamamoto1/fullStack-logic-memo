# ブログプラットフォーム ⭐⭐

**難易度**: 中級
**推奨時間**: 20-25時間
**技術スタック**: Next.js, TypeScript, Prisma, JWT, TipTap, AWS S3

---

## 概要

記事のCRUD操作、認証、画像アップロードを備えた本格的なブログシステムを構築します。

---

## 学習目標

- [ ] Next.js App Routerの活用
- [ ] PrismaによるORM操作
- [ ] JWT認証の実装
- [ ] リッチテキストエディタ（TipTap）の統合
- [ ] AWS S3への画像アップロード
- [ ] SEO対策の実装

---

## 機能要件

### 必須機能

1. **認証機能**
   - ユーザー登録・ログイン
   - JWTトークン管理
   - プロフィール管理

2. **記事管理**
   - 記事作成・編集・削除
   - リッチテキストエディタ
   - 下書き保存
   - 公開/非公開切り替え

3. **画像アップロード**
   - 記事内画像挿入
   - サムネイル設定
   - 画像リサイズ

4. **記事表示**
   - 記事一覧（ページネーション）
   - 記事詳細
   - カテゴリ・タグフィルター
   - 検索機能

---

## データモデル

```prisma
model User {
  id        String   @id @default(cuid())
  email     String   @unique
  password  String
  name      String
  avatar    String?
  bio       String?
  posts     Post[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model Post {
  id          String     @id @default(cuid())
  title       String
  slug        String     @unique
  content     String     @db.Text
  excerpt     String?
  thumbnail   String?
  published   Boolean    @default(false)
  author      User       @relation(fields: [authorId], references: [id])
  authorId    String
  categories  Category[]
  tags        Tag[]
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt
  publishedAt DateTime?
}

model Category {
  id    String @id @default(cuid())
  name  String @unique
  slug  String @unique
  posts Post[]
}

model Tag {
  id    String @id @default(cuid())
  name  String @unique
  slug  String @unique
  posts Post[]
}
```

---

## 実装例

### 記事作成API

```typescript
// app/api/posts/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { prisma } from '@/lib/prisma';
import { verifyToken } from '@/lib/auth';

export async function POST(request: NextRequest) {
  try {
    const user = await verifyToken(request);
    if (!user) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
    }

    const { title, content, excerpt, thumbnail, categories, tags, published } =
      await request.json();

    const slug = generateSlug(title);

    const post = await prisma.post.create({
      data: {
        title,
        slug,
        content,
        excerpt,
        thumbnail,
        published,
        publishedAt: published ? new Date() : null,
        authorId: user.id,
        categories: {
          connect: categories.map((id: string) => ({ id }))
        },
        tags: {
          connectOrCreate: tags.map((name: string) => ({
            where: { name },
            create: { name, slug: generateSlug(name) }
          }))
        }
      },
      include: {
        author: { select: { id: true, name: true, avatar: true } },
        categories: true,
        tags: true
      }
    });

    return NextResponse.json(post, { status: 201 });
  } catch (error) {
    return NextResponse.json(
      { error: 'Failed to create post' },
      { status: 500 }
    );
  }
}
```

### リッチテキストエディタ

```typescript
'use client';

import { useEditor, EditorContent } from '@tiptap/react';
import StarterKit from '@tiptap/starter-kit';
import Image from '@tiptap/extension-image';
import Link from '@tiptap/extension-link';
import Placeholder from '@tiptap/extension-placeholder';

interface EditorProps {
  content: string;
  onChange: (content: string) => void;
}

export function RichTextEditor({ content, onChange }: EditorProps) {
  const editor = useEditor({
    extensions: [
      StarterKit,
      Image.configure({ HTMLAttributes: { class: 'rounded-lg' } }),
      Link.configure({ openOnClick: false }),
      Placeholder.configure({ placeholder: '記事を書き始めましょう...' })
    ],
    content,
    onUpdate: ({ editor }) => {
      onChange(editor.getHTML());
    }
  });

  return (
    <div className="border rounded-lg">
      <EditorToolbar editor={editor} />
      <EditorContent editor={editor} className="prose max-w-none p-4" />
    </div>
  );
}
```

---

## 受け入れ基準

- [ ] ユーザー登録・ログインが動作する
- [ ] 記事のCRUD操作ができる
- [ ] リッチテキストエディタで記事が書ける
- [ ] 画像アップロードが動作する
- [ ] カテゴリ・タグでフィルターできる
- [ ] SEOメタタグが設定されている

---

**最終更新**: 2025-10-22
