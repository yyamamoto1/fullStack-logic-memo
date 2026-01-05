# ファイルアップロードAPI ⭐⭐

**難易度**: 中級
**推奨時間**: 10-15時間
**技術スタック**: Node.js, Express, Multer, Sharp, AWS S3

---

## 概要

画像・ドキュメントのアップロードAPIを構築します。マルチパートアップロード、画像リサイズ、クラウドストレージ連携を学びます。

---

## 学習目標

- [ ] Multerによるファイル受信
- [ ] Sharpによる画像処理
- [ ] AWS S3へのアップロード
- [ ] プリサインドURLの生成
- [ ] ファイルメタデータ管理
- [ ] セキュリティ対策

---

## 機能要件

### エンドポイント

| メソッド | パス | 説明 |
|---------|------|------|
| POST | /api/upload/image | 画像アップロード |
| POST | /api/upload/document | ドキュメントアップロード |
| GET | /api/files | ファイル一覧取得 |
| GET | /api/files/:id | ファイル詳細取得 |
| DELETE | /api/files/:id | ファイル削除 |
| GET | /api/files/:id/download | ダウンロードURL取得 |

---

## 実装例

### ファイルアップロード処理

```typescript
import express from 'express';
import multer from 'multer';
import sharp from 'sharp';
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { v4 as uuidv4 } from 'uuid';

const s3Client = new S3Client({
  region: process.env.AWS_REGION,
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID!,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY!
  }
});

const upload = multer({
  storage: multer.memoryStorage(),
  limits: {
    fileSize: 10 * 1024 * 1024, // 10MB
  },
  fileFilter: (req, file, cb) => {
    const allowedMimeTypes = [
      'image/jpeg',
      'image/png',
      'image/gif',
      'image/webp',
      'application/pdf',
      'application/msword',
      'application/vnd.openxmlformats-officedocument.wordprocessingml.document'
    ];

    if (allowedMimeTypes.includes(file.mimetype)) {
      cb(null, true);
    } else {
      cb(new Error('Invalid file type'));
    }
  }
});

// 画像アップロード（リサイズ付き）
app.post('/api/upload/image', upload.single('file'), async (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({ error: 'No file provided' });
    }

    const fileId = uuidv4();
    const originalBuffer = req.file.buffer;
    const filename = `${fileId}-original.webp`;
    const thumbnailFilename = `${fileId}-thumbnail.webp`;

    // オリジナル画像をWebPに変換
    const originalWebp = await sharp(originalBuffer)
      .webp({ quality: 85 })
      .toBuffer();

    // サムネイル作成
    const thumbnail = await sharp(originalBuffer)
      .resize(300, 300, { fit: 'cover' })
      .webp({ quality: 80 })
      .toBuffer();

    // S3にアップロード
    await Promise.all([
      s3Client.send(new PutObjectCommand({
        Bucket: process.env.S3_BUCKET,
        Key: `images/${filename}`,
        Body: originalWebp,
        ContentType: 'image/webp'
      })),
      s3Client.send(new PutObjectCommand({
        Bucket: process.env.S3_BUCKET,
        Key: `thumbnails/${thumbnailFilename}`,
        Body: thumbnail,
        ContentType: 'image/webp'
      }))
    ]);

    // メタデータを取得
    const metadata = await sharp(originalBuffer).metadata();

    // データベースに保存
    const fileRecord = await prisma.file.create({
      data: {
        id: fileId,
        originalName: req.file.originalname,
        filename,
        mimeType: 'image/webp',
        size: originalWebp.length,
        width: metadata.width,
        height: metadata.height,
        s3Key: `images/${filename}`,
        thumbnailKey: `thumbnails/${thumbnailFilename}`,
        userId: req.user.id
      }
    });

    res.status(201).json({
      id: fileRecord.id,
      url: await getSignedUrl(s3Client, new GetObjectCommand({
        Bucket: process.env.S3_BUCKET,
        Key: `images/${filename}`
      }), { expiresIn: 3600 }),
      thumbnailUrl: await getSignedUrl(s3Client, new GetObjectCommand({
        Bucket: process.env.S3_BUCKET,
        Key: `thumbnails/${thumbnailFilename}`
      }), { expiresIn: 3600 }),
      metadata: {
        originalName: req.file.originalname,
        size: originalWebp.length,
        width: metadata.width,
        height: metadata.height
      }
    });
  } catch (error) {
    console.error('Upload failed:', error);
    res.status(500).json({ error: 'Upload failed' });
  }
});

// 複数ファイルアップロード
app.post('/api/upload/images', upload.array('files', 10), async (req, res) => {
  const files = req.files as Express.Multer.File[];

  const results = await Promise.all(
    files.map(file => processAndUploadImage(file, req.user.id))
  );

  res.status(201).json(results);
});

// ダウンロードURL生成
app.get('/api/files/:id/download', async (req, res) => {
  const file = await prisma.file.findUnique({
    where: { id: req.params.id }
  });

  if (!file) {
    return res.status(404).json({ error: 'File not found' });
  }

  const url = await getSignedUrl(s3Client, new GetObjectCommand({
    Bucket: process.env.S3_BUCKET,
    Key: file.s3Key,
    ResponseContentDisposition: `attachment; filename="${file.originalName}"`
  }), { expiresIn: 300 }); // 5分間有効

  res.json({ downloadUrl: url });
});
```

### データモデル

```prisma
model File {
  id           String   @id
  originalName String
  filename     String
  mimeType     String
  size         Int
  width        Int?
  height       Int?
  s3Key        String
  thumbnailKey String?
  user         User     @relation(fields: [userId], references: [id])
  userId       String
  createdAt    DateTime @default(now())
}
```

---

## セキュリティ対策

1. **ファイルタイプ検証**: MIMEタイプとマジックナンバーを確認
2. **ファイルサイズ制限**: 適切な上限を設定
3. **ファイル名サニタイズ**: 元のファイル名を直接使用しない
4. **署名付きURL**: S3の直接アクセスを防止
5. **アクセス制御**: 所有者のみがファイルにアクセス

---

## 受け入れ基準

- [ ] 画像アップロードが動作する
- [ ] 画像リサイズが正しく行われる
- [ ] S3にファイルが保存される
- [ ] 署名付きURLが生成される
- [ ] ファイル削除が動作する
- [ ] 不正なファイルタイプを拒否する

---

**最終更新**: 2025-10-22
