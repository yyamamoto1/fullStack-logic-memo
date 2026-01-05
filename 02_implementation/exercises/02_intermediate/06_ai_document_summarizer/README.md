# AI文書要約システム ⭐⭐

**難易度**: 中級
**推奨時間**: 15-20時間
**技術スタック**: Python, FastAPI, OpenAI API, PyPDF2, LangChain

---

## 概要

PDF/文書を解析して要約を生成するシステムを構築します。自然言語処理とAI APIの活用方法を学びます。

---

## 学習目標

- [ ] PDF解析（PyPDF2）
- [ ] OpenAI APIによるテキスト要約
- [ ] LangChainによる大規模文書処理
- [ ] チャンク分割戦略
- [ ] 多言語対応
- [ ] 非同期処理

---

## 機能要件

### エンドポイント

| メソッド | パス | 説明 |
|---------|------|------|
| POST | /api/summarize | 文書要約生成 |
| POST | /api/upload | PDF/文書アップロード |
| GET | /api/documents | 文書一覧取得 |
| GET | /api/documents/:id | 要約結果取得 |
| POST | /api/documents/:id/qa | 文書に対する質問応答 |

---

## 実装例

### FastAPI アプリケーション

```python
from fastapi import FastAPI, UploadFile, HTTPException, BackgroundTasks
from pydantic import BaseModel
from typing import List, Optional
import openai
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.embeddings import OpenAIEmbeddings
from langchain.vectorstores import FAISS
import PyPDF2
import io

app = FastAPI()

class SummarizeRequest(BaseModel):
    text: str
    max_length: int = 500
    language: str = "ja"
    style: str = "concise"  # concise, detailed, bullet_points

class DocumentSummary(BaseModel):
    document_id: str
    title: str
    summary: str
    key_points: List[str]
    word_count: int
    page_count: int

# PDF解析
def extract_text_from_pdf(file_content: bytes) -> tuple[str, int]:
    pdf_reader = PyPDF2.PdfReader(io.BytesIO(file_content))
    text = ""
    for page in pdf_reader.pages:
        text += page.extract_text() + "\n"
    return text, len(pdf_reader.pages)

# チャンク分割
def split_text_into_chunks(text: str) -> List[str]:
    text_splitter = RecursiveCharacterTextSplitter(
        chunk_size=4000,
        chunk_overlap=200,
        length_function=len,
        separators=["\n\n", "\n", "。", ".", " ", ""]
    )
    return text_splitter.split_text(text)

# 要約生成
async def generate_summary(
    text: str,
    max_length: int = 500,
    language: str = "ja",
    style: str = "concise"
) -> dict:
    chunks = split_text_into_chunks(text)

    # 大規模文書の場合は段階的要約
    if len(chunks) > 1:
        return await hierarchical_summarize(chunks, max_length, language, style)

    system_prompt = get_system_prompt(language, style)

    response = await openai.ChatCompletion.acreate(
        model="gpt-4-turbo-preview",
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": f"以下の文書を要約してください:\n\n{text}"}
        ],
        temperature=0.3,
        max_tokens=max_length
    )

    summary = response.choices[0].message.content
    key_points = await extract_key_points(text, language)

    return {
        "summary": summary,
        "key_points": key_points,
        "word_count": len(text)
    }

def get_system_prompt(language: str, style: str) -> str:
    prompts = {
        "ja": {
            "concise": "あなたは文書要約の専門家です。簡潔で分かりやすい要約を作成してください。",
            "detailed": "あなたは文書要約の専門家です。詳細で包括的な要約を作成してください。",
            "bullet_points": "あなたは文書要約の専門家です。箇条書きで要点をまとめてください。"
        },
        "en": {
            "concise": "You are a document summarization expert. Create concise and clear summaries.",
            "detailed": "You are a document summarization expert. Create detailed and comprehensive summaries.",
            "bullet_points": "You are a document summarization expert. Summarize key points in bullet points."
        }
    }
    return prompts.get(language, prompts["en"]).get(style, prompts["en"]["concise"])

# 階層的要約（大規模文書用）
async def hierarchical_summarize(
    chunks: List[str],
    max_length: int,
    language: str,
    style: str
) -> dict:
    # 各チャンクを要約
    chunk_summaries = []
    for chunk in chunks:
        response = await openai.ChatCompletion.acreate(
            model="gpt-4-turbo-preview",
            messages=[
                {"role": "system", "content": "この文書の一部を簡潔に要約してください。"},
                {"role": "user", "content": chunk}
            ],
            temperature=0.3,
            max_tokens=500
        )
        chunk_summaries.append(response.choices[0].message.content)

    # 統合要約
    combined = "\n\n".join(chunk_summaries)
    system_prompt = get_system_prompt(language, style)

    final_response = await openai.ChatCompletion.acreate(
        model="gpt-4-turbo-preview",
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": f"以下の部分要約を統合して、一つの要約を作成してください:\n\n{combined}"}
        ],
        temperature=0.3,
        max_tokens=max_length
    )

    return {
        "summary": final_response.choices[0].message.content,
        "key_points": await extract_key_points(combined, language),
        "chunk_count": len(chunks)
    }

# キーポイント抽出
async def extract_key_points(text: str, language: str = "ja") -> List[str]:
    prompt = "以下の文書から5つの重要なポイントを抽出してください。" if language == "ja" else "Extract 5 key points from the following document."

    response = await openai.ChatCompletion.acreate(
        model="gpt-4-turbo-preview",
        messages=[
            {"role": "system", "content": prompt},
            {"role": "user", "content": text[:8000]}  # トークン制限
        ],
        temperature=0.3
    )

    # 箇条書きを解析
    content = response.choices[0].message.content
    points = [line.strip().lstrip("•-1234567890.）) ")
              for line in content.split("\n")
              if line.strip()]

    return points[:5]

# エンドポイント
@app.post("/api/summarize")
async def summarize_text(request: SummarizeRequest):
    result = await generate_summary(
        request.text,
        request.max_length,
        request.language,
        request.style
    )
    return result

@app.post("/api/upload")
async def upload_document(
    file: UploadFile,
    background_tasks: BackgroundTasks
):
    if not file.filename.endswith('.pdf'):
        raise HTTPException(status_code=400, detail="Only PDF files are supported")

    content = await file.read()
    text, page_count = extract_text_from_pdf(content)

    # ドキュメントをDBに保存
    doc_id = save_document(file.filename, text, page_count)

    # バックグラウンドで要約を生成
    background_tasks.add_task(process_document_summary, doc_id, text)

    return {
        "document_id": doc_id,
        "status": "processing",
        "page_count": page_count
    }
```

---

## ベクトル検索による質問応答

```python
from langchain.chains import RetrievalQA

class DocumentQA:
    def __init__(self, document_text: str):
        self.text_splitter = RecursiveCharacterTextSplitter(
            chunk_size=1000,
            chunk_overlap=100
        )
        chunks = self.text_splitter.split_text(document_text)

        embeddings = OpenAIEmbeddings()
        self.vectorstore = FAISS.from_texts(chunks, embeddings)

    async def ask(self, question: str) -> str:
        retriever = self.vectorstore.as_retriever(
            search_type="similarity",
            search_kwargs={"k": 3}
        )

        relevant_docs = retriever.get_relevant_documents(question)
        context = "\n".join([doc.page_content for doc in relevant_docs])

        response = await openai.ChatCompletion.acreate(
            model="gpt-4-turbo-preview",
            messages=[
                {"role": "system", "content": "与えられたコンテキストに基づいて質問に答えてください。"},
                {"role": "user", "content": f"コンテキスト:\n{context}\n\n質問: {question}"}
            ],
            temperature=0.3
        )

        return response.choices[0].message.content
```

---

## 受け入れ基準

- [ ] PDFアップロードが動作する
- [ ] テキスト抽出が正確に行われる
- [ ] 要約生成が動作する
- [ ] 大規模文書も処理できる
- [ ] 質問応答機能が動作する
- [ ] 多言語に対応している

---

**最終更新**: 2025-10-22
