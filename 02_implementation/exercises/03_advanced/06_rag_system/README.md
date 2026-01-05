# RAGシステム構築 ⭐⭐⭐

**難易度**: 上級
**推奨時間**: 30-40時間
**技術スタック**: Python, FastAPI, LangChain, Pinecone/Chroma, OpenAI

---

## 概要

Retrieval-Augmented Generation（RAG）システムを構築します。ベクトルデータベース、文書埋め込み、検索拡張生成を学びます。

---

## 学習目標

- [ ] ベクトルデータベース（Pinecone/Chroma）
- [ ] 文書埋め込み（OpenAI Embeddings）
- [ ] テキストチャンキング戦略
- [ ] セマンティック検索
- [ ] コンテキスト管理
- [ ] ハイブリッド検索

---

## アーキテクチャ

```
┌─────────────────────────────────────────────────┐
│                  RAG Pipeline                    │
└─────────────────────────────────────────────────┘
                        │
    ┌───────────────────┼───────────────────┐
    ▼                   ▼                   ▼
┌─────────┐      ┌──────────┐      ┌──────────┐
│Document │      │ Embedding│      │  Query   │
│ Loader  │      │ Generator│      │ Processor│
└────┬────┘      └────┬─────┘      └────┬─────┘
     │                │                 │
     ▼                ▼                 ▼
┌─────────┐      ┌──────────┐      ┌──────────┐
│  Text   │      │  Vector  │      │  Hybrid  │
│ Chunker │      │   Store  │      │  Search  │
└────┬────┘      └────┬─────┘      └────┬─────┘
     │                │                 │
     └────────────────┼─────────────────┘
                      ▼
              ┌──────────────┐
              │   Retriever  │
              └──────┬───────┘
                     ▼
              ┌──────────────┐
              │   Generator  │
              │   (LLM)      │
              └──────────────┘
```

---

## 機能要件

### 必須機能

1. **文書取り込み**
   - PDF/Word/テキストファイル対応
   - Webページスクレイピング
   - 自動チャンキング
   - メタデータ抽出

2. **ベクトル検索**
   - セマンティック検索
   - キーワード検索（BM25）
   - ハイブリッド検索

3. **質問応答**
   - コンテキスト付き回答生成
   - 引用元表示
   - 会話履歴対応

4. **管理機能**
   - 文書管理
   - インデックス管理
   - 使用量モニタリング

---

## 実装例

### RAG パイプライン

```python
from langchain.document_loaders import PyPDFLoader, DirectoryLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.embeddings import OpenAIEmbeddings
from langchain.vectorstores import Chroma, Pinecone
from langchain.chat_models import ChatOpenAI
from langchain.chains import ConversationalRetrievalChain
from langchain.memory import ConversationBufferWindowMemory
import pinecone

class RAGPipeline:
    def __init__(self):
        self.embeddings = OpenAIEmbeddings(
            model="text-embedding-3-small"
        )
        self.llm = ChatOpenAI(
            model="gpt-4-turbo-preview",
            temperature=0.3
        )
        self.text_splitter = RecursiveCharacterTextSplitter(
            chunk_size=1000,
            chunk_overlap=200,
            length_function=len,
            separators=["\n\n", "\n", "。", ".", " ", ""]
        )

        # Pinecone初期化
        pinecone.init(
            api_key=os.environ["PINECONE_API_KEY"],
            environment=os.environ["PINECONE_ENV"]
        )
        self.index_name = "rag-knowledge-base"

    async def ingest_documents(self, file_paths: list[str], namespace: str = "default"):
        """文書を取り込んでベクトル化"""
        documents = []

        for path in file_paths:
            if path.endswith('.pdf'):
                loader = PyPDFLoader(path)
            else:
                loader = TextLoader(path)

            docs = loader.load()

            # メタデータ追加
            for doc in docs:
                doc.metadata.update({
                    'source': path,
                    'namespace': namespace,
                    'ingested_at': datetime.now().isoformat()
                })

            documents.extend(docs)

        # チャンキング
        chunks = self.text_splitter.split_documents(documents)

        # ベクトルストアに追加
        vectorstore = Pinecone.from_documents(
            chunks,
            self.embeddings,
            index_name=self.index_name,
            namespace=namespace
        )

        return {
            'total_documents': len(documents),
            'total_chunks': len(chunks),
            'namespace': namespace
        }

    async def query(
        self,
        question: str,
        namespace: str = "default",
        top_k: int = 5,
        chat_history: list = None
    ) -> dict:
        """質問に回答"""

        # ベクトルストアから取得
        vectorstore = Pinecone.from_existing_index(
            index_name=self.index_name,
            embedding=self.embeddings,
            namespace=namespace
        )

        # ハイブリッド検索（セマンティック + キーワード）
        retriever = vectorstore.as_retriever(
            search_type="mmr",  # Maximum Marginal Relevance
            search_kwargs={
                "k": top_k,
                "fetch_k": top_k * 3,
                "lambda_mult": 0.7  # 多様性と関連性のバランス
            }
        )

        # 関連文書を取得
        relevant_docs = retriever.get_relevant_documents(question)

        # コンテキスト構築
        context = self._build_context(relevant_docs)

        # 回答生成
        response = await self._generate_answer(
            question,
            context,
            chat_history or []
        )

        return {
            'answer': response['answer'],
            'sources': [
                {
                    'content': doc.page_content[:200] + '...',
                    'source': doc.metadata.get('source', 'unknown'),
                    'page': doc.metadata.get('page', None)
                }
                for doc in relevant_docs
            ],
            'confidence': self._calculate_confidence(relevant_docs)
        }

    def _build_context(self, documents: list) -> str:
        """コンテキストを構築"""
        context_parts = []

        for i, doc in enumerate(documents, 1):
            source = doc.metadata.get('source', 'unknown')
            context_parts.append(
                f"[Document {i} - {source}]\n{doc.page_content}"
            )

        return "\n\n---\n\n".join(context_parts)

    async def _generate_answer(
        self,
        question: str,
        context: str,
        chat_history: list
    ) -> dict:
        """回答を生成"""

        system_prompt = """あなたは知識ベースに基づいて質問に回答するアシスタントです。

回答ルール:
1. 提供されたコンテキストに基づいて回答してください
2. コンテキストに情報がない場合は「情報が見つかりませんでした」と回答してください
3. 推測や憶測は避けてください
4. 回答には関連する文書を引用してください
5. 簡潔かつ正確に回答してください"""

        messages = [
            {"role": "system", "content": system_prompt},
        ]

        # 会話履歴を追加
        for h in chat_history[-4:]:  # 直近4往復
            messages.append({"role": "user", "content": h['question']})
            messages.append({"role": "assistant", "content": h['answer']})

        # 現在の質問とコンテキスト
        messages.append({
            "role": "user",
            "content": f"""コンテキスト:
{context}

質問: {question}

上記のコンテキストに基づいて回答してください。"""
        })

        response = await self.llm.agenerate([messages])

        return {
            'answer': response.generations[0][0].text
        }

    def _calculate_confidence(self, documents: list) -> float:
        """回答の信頼度を計算"""
        if not documents:
            return 0.0

        # 類似度スコアの平均（メタデータにスコアがある場合）
        scores = [doc.metadata.get('score', 0.5) for doc in documents]
        return sum(scores) / len(scores)
```

### FastAPI エンドポイント

```python
from fastapi import FastAPI, UploadFile, HTTPException
from pydantic import BaseModel
from typing import List, Optional

app = FastAPI()
rag = RAGPipeline()

class QueryRequest(BaseModel):
    question: str
    namespace: str = "default"
    top_k: int = 5
    chat_history: Optional[List[dict]] = None

class IngestRequest(BaseModel):
    namespace: str = "default"

@app.post("/api/ingest")
async def ingest_documents(
    files: List[UploadFile],
    namespace: str = "default"
):
    """文書をアップロードしてインデックス化"""
    file_paths = []

    for file in files:
        # 一時ファイルに保存
        temp_path = f"/tmp/{file.filename}"
        with open(temp_path, "wb") as f:
            content = await file.read()
            f.write(content)
        file_paths.append(temp_path)

    result = await rag.ingest_documents(file_paths, namespace)

    # 一時ファイル削除
    for path in file_paths:
        os.remove(path)

    return result

@app.post("/api/query")
async def query(request: QueryRequest):
    """質問に回答"""
    result = await rag.query(
        question=request.question,
        namespace=request.namespace,
        top_k=request.top_k,
        chat_history=request.chat_history
    )
    return result

@app.get("/api/namespaces")
async def list_namespaces():
    """名前空間一覧を取得"""
    index = pinecone.Index(rag.index_name)
    stats = index.describe_index_stats()
    return {
        'namespaces': list(stats['namespaces'].keys()),
        'total_vectors': stats['total_vector_count']
    }

@app.delete("/api/namespaces/{namespace}")
async def delete_namespace(namespace: str):
    """名前空間を削除"""
    index = pinecone.Index(rag.index_name)
    index.delete(delete_all=True, namespace=namespace)
    return {'deleted': namespace}
```

### 高度なチャンキング戦略

```python
from langchain.text_splitter import (
    RecursiveCharacterTextSplitter,
    MarkdownHeaderTextSplitter,
    TokenTextSplitter
)

class SmartChunker:
    """文書タイプに応じた最適なチャンキング"""

    def __init__(self):
        self.default_splitter = RecursiveCharacterTextSplitter(
            chunk_size=1000,
            chunk_overlap=200
        )

        self.markdown_splitter = MarkdownHeaderTextSplitter(
            headers_to_split_on=[
                ("#", "Header 1"),
                ("##", "Header 2"),
                ("###", "Header 3"),
            ]
        )

        self.code_splitter = RecursiveCharacterTextSplitter.from_language(
            language=Language.PYTHON,
            chunk_size=2000,
            chunk_overlap=200
        )

    def chunk(self, document: Document) -> List[Document]:
        """文書タイプに応じてチャンキング"""
        source = document.metadata.get('source', '')

        if source.endswith('.md'):
            return self._chunk_markdown(document)
        elif source.endswith(('.py', '.js', '.ts')):
            return self._chunk_code(document)
        else:
            return self._chunk_default(document)

    def _chunk_markdown(self, document: Document) -> List[Document]:
        """Markdownをヘッダーで分割"""
        md_splits = self.markdown_splitter.split_text(document.page_content)

        # さらに長いセクションを分割
        final_chunks = []
        for split in md_splits:
            if len(split.page_content) > 1500:
                sub_chunks = self.default_splitter.split_text(split.page_content)
                for chunk in sub_chunks:
                    final_chunks.append(Document(
                        page_content=chunk,
                        metadata={**document.metadata, **split.metadata}
                    ))
            else:
                final_chunks.append(Document(
                    page_content=split.page_content,
                    metadata={**document.metadata, **split.metadata}
                ))

        return final_chunks
```

---

## 受け入れ基準

- [ ] PDF/テキスト文書を取り込める
- [ ] セマンティック検索が動作する
- [ ] 質問に対して適切な回答が生成される
- [ ] 引用元が表示される
- [ ] 会話履歴が考慮される
- [ ] 名前空間で文書を分類できる

---

**最終更新**: 2025-10-22
