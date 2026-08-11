# RAG System Architecture

## System Overview

This RAG (Retrieval-Augmented Generation) system is designed specifically for analyzing earnings reports and financial documents. It combines document processing, vector search, and large language models to provide intelligent question-answering capabilities.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    USER INTERFACE                            │
│  (CLI, API, Streamlit Dashboard, Jupyter Notebook)          │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│               EARNINGS RAG PIPELINE                          │
│                                                              │
│  ┌────────────────────────────────────────────────────┐     │
│  │  Query Engine                                      │     │
│  │  • Question preprocessing                          │     │
│  │  • Query rewriting                                 │     │
│  │  • Metadata filtering                              │     │
│  └────────────┬───────────────────────────────────────┘     │
│               │                                              │
│               ▼                                              │
│  ┌────────────────────────────────────────────────────┐     │
│  │  Retriever                                         │     │
│  │  • Similarity search                               │     │
│  │  • Metadata filtering (company, quarter)           │     │
│  │  • Re-ranking                                      │     │
│  └────────────┬───────────────────────────────────────┘     │
│               │                                              │
│               ▼                                              │
│  ┌────────────────────────────────────────────────────┐     │
│  │  LLM Chain                                         │     │
│  │  • Context assembly                                │     │
│  │  • Prompt engineering                              │     │
│  │  • Answer generation                               │     │
│  └────────────────────────────────────────────────────┘     │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                 STORAGE LAYER                                │
│                                                              │
│  ┌──────────────────┐         ┌──────────────────┐          │
│  │  Vector Store    │         │  Metadata Store  │          │
│  │  • FAISS/Chroma  │         │  • Companies     │          │
│  │  • Embeddings    │         │  • Earnings      │          │
│  │  • Similarity    │         │  • Metrics       │          │
│  └──────────────────┘         └──────────────────┘          │
└─────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              DOCUMENT PROCESSING                             │
│                                                              │
│  ┌────────┐    ┌─────────┐    ┌──────────┐    ┌─────────┐  │
│  │ Loader │ →  │ Chunker │ →  │ Embedder │ →  │ Storage │  │
│  └────────┘    └─────────┘    └──────────┘    └─────────┘  │
│      │             │               │                │        │
│   PDF,TXT      Semantic      OpenAI/HF          Vector      │
│   HTML,DOCX    Chunks        Embeddings          Store      │
└─────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                 DATA SOURCES                                 │
│  • SEC EDGAR (10-K, 10-Q, 8-K)                              │
│  • Company websites                                          │
│  • Uploaded files                                            │
│  • Web scraping                                              │
└─────────────────────────────────────────────────────────────┘
```

## Component Details

### 1. Document Loader (`core/document_loader.py`)

**Purpose**: Load documents from various sources

**Features**:
- Multi-format support (PDF, TXT, HTML, DOCX)
- URL scraping
- SEC EDGAR integration (placeholder)
- Metadata preservation

**Flow**:
```
Source → Load → Parse → Document + Metadata
```

**Key Classes**:
- `EarningsDocumentLoader`: Main loader interface

### 2. Text Chunker (`core/text_chunker.py`)

**Purpose**: Split documents into optimal chunks for retrieval

**Strategies**:
- Recursive character splitting
- Token-based splitting
- Section-aware splitting

**Parameters**:
- `chunk_size`: 500-1500 characters
- `chunk_overlap`: 10-20% of chunk_size
- `separators`: `["\n\n", "\n", ". ", " "]`

**Flow**:
```
Document → Split → Chunks with Metadata
```

**Key Classes**:
- `SmartChunker`: Generic chunker
- `EarningsChunker`: Earnings-specific logic

### 3. Embeddings (`core/embeddings.py`)

**Purpose**: Convert text to vector representations

**Providers**:
1. **OpenAI** (text-embedding-3-small/large)
   - Dimensions: 1536/3072
   - Quality: Excellent
   - Cost: Pay per token
   
2. **HuggingFace** (sentence-transformers)
   - Dimensions: 384-768
   - Quality: Good
   - Cost: Free

**Flow**:
```
Text → Tokenize → Embed → Vector[1536]
```

**Key Classes**:
- `EmbeddingManager`: Multi-provider support
- `CachedEmbeddings`: Caching layer

### 4. Vector Store (`core/vector_store.py`)

**Purpose**: Store and search document vectors

**Backends**:
1. **FAISS** (Facebook AI Similarity Search)
   - Type: Local, in-memory
   - Speed: Very fast
   - Scalability: Millions of vectors
   
2. **Chroma**
   - Type: Local, persistent
   - Speed: Fast
   - Features: Built-in metadata filtering

**Operations**:
- `add_documents()`: Ingest new documents
- `similarity_search()`: Find similar chunks
- `similarity_search_with_score()`: With relevance scores
- Metadata filtering: `filter={'company': 'AAPL'}`

**Flow**:
```
Query → Embed → Similarity Search → Top-K Results
```

**Key Classes**:
- `VectorStoreManager`: Generic interface
- `EarningsVectorStore`: Earnings-specific methods

### 5. RAG Pipeline (`rag/pipeline.py`)

**Purpose**: End-to-end RAG workflow

**Components**:
1. Document ingestion
2. Chunking and embedding
3. Vector storage
4. Query processing
5. Answer generation

**Chain Structure**:
```python
chain = (
    {"context": retriever, "question": input}
    | prompt
    | llm
    | output_parser
)
```

**Flow**:
```
Question → Retrieve Context → Format Prompt → LLM → Answer
```

**Key Classes**:
- `EarningsRAG`: Main pipeline orchestrator

### 6. Earnings Analyzer (`earnings/analyzer.py`)

**Purpose**: Company-specific analysis and comparison

**Features**:
- Company tracking
- Earnings history
- Trend analysis
- Multi-company comparison

**Key Classes**:
- `CompanyTracker`: Maintain company database
- `EarningsAnalyzer`: Analysis methods
- `EarningsMetrics`: Structured data model

## Data Flow

### Ingestion Pipeline

```
1. Source Document
   ↓
2. Load & Parse
   ↓ (metadata: company, quarter, date)
3. Chunk into Segments
   ↓ (chunks: 500-1500 chars with overlap)
4. Generate Embeddings
   ↓ (vectors: 384-1536 dimensions)
5. Store in Vector DB
   ↓ (indexed for similarity search)
6. Save Metadata
   (company tracker, metrics)
```

### Query Pipeline

```
1. User Question
   ↓
2. Preprocess & Filter
   ↓ (optional: company, quarter filters)
3. Generate Query Embedding
   ↓
4. Similarity Search
   ↓ (retrieve top-k relevant chunks)
5. Assemble Context
   ↓ (format retrieved chunks)
6. Generate Prompt
   ↓ (question + context + instructions)
7. LLM Generation
   ↓
8. Return Answer + Sources
```

## Embedding Strategy

### Why Embeddings?

Traditional keyword search fails for:
- "What was Apple's revenue?" vs "How much did AAPL make?"
- "Q3 performance" vs "Third quarter results"

Embeddings capture semantic meaning:
```
"revenue growth" ≈ "sales increase" ≈ "top-line expansion"
```

### Similarity Calculation

Cosine similarity between query and document vectors:
```
similarity = dot(query_vec, doc_vec) / (||query_vec|| * ||doc_vec||)
```

Range: -1 to 1 (higher = more similar)

## Chunking Strategy

### Why Chunk?

1. **Context window limits**: LLMs have token limits
2. **Retrieval precision**: Smaller chunks = more focused results
3. **Relevance**: Avoid mixing unrelated content

### Optimal Chunk Size

- **Too small** (< 200 chars): Lacks context
- **Too large** (> 2000 chars): Too broad, less precise
- **Sweet spot**: 500-1500 characters

### Overlap Strategy

```
Chunk 1: [-------- 1000 chars --------]
                            [200 overlap]
Chunk 2:              [-------- 1000 chars --------]
```

Benefits:
- Preserves context across boundaries
- Improves retrieval of split information

## Prompt Engineering

### System Prompt Template

```
You are a financial analyst assistant specializing in earnings reports.

Use ONLY the following context to answer the question.
If you cannot find the answer in the context, say so clearly.

Context:
{retrieved_chunks}

Question: {user_question}

Guidelines:
1. Be specific with numbers
2. Cite company and quarter
3. Don't make up information
4. If comparing, use exact figures

Answer:
```

### Context Formatting

```
[Apple - Q3 2024]
Revenue: $85.8 billion, up 5% YoY...

[Microsoft - Q3 2024]  
Azure revenue grew 31%...
```

## Retrieval Strategies

### 1. Basic Similarity Search
```python
results = vector_store.similarity_search(query, k=4)
```

### 2. Metadata Filtering
```python
results = vector_store.similarity_search(
    query,
    k=4,
    filter={'company': 'AAPL', 'quarter': 'Q3 2024'}
)
```

### 3. Hybrid Search (Future)
Combine:
- Semantic search (embeddings)
- Keyword search (BM25)
- Metadata filters

## Scalability Considerations

### Current System
- **Documents**: ~1000s
- **Chunks**: ~10,000s  
- **Storage**: Local disk
- **Latency**: < 1 second

### Production Scale
- **Documents**: ~100,000s
- **Chunks**: ~1,000,000s
- **Storage**: Cloud (Pinecone, Weaviate)
- **Latency**: < 500ms

### Optimization Strategies

1. **Batch Processing**: Ingest in batches
2. **Caching**: Cache embeddings and queries
3. **Indexing**: Use optimized index structures (HNSW)
4. **Sharding**: Distribute across multiple stores
5. **Compression**: Compress embeddings (PQ, LSH)

## Security & Privacy

### Data Protection
- Encryption at rest
- Access control lists
- Audit logging
- Data retention policies

### API Security
- API key authentication
- Rate limiting
- Input validation
- Output filtering

## Monitoring & Debugging

### Key Metrics
1. **Retrieval Quality**
   - Relevance scores
   - Top-k accuracy
   
2. **System Performance**
   - Query latency
   - Throughput (QPS)
   - Cache hit rate
   
3. **Answer Quality**
   - User feedback
   - Accuracy rate

### Debug Tools
```python
# Check retrieved context
result = rag.query(question, return_sources=True)
for source in result['sources']:
    print(source['content'])
    print(source['metadata'])

# Inspect embeddings
embedding = embedder.embed_query("test")
print(f"Dimension: {len(embedding)}")
print(f"First 5 values: {embedding[:5]}")
```

## Future Enhancements

### Short Term
- [ ] Add re-ranking with cross-encoder
- [ ] Implement query rewriting
- [ ] Add streaming responses
- [ ] Build Streamlit UI

### Medium Term
- [ ] Multi-modal RAG (text + charts)
- [ ] Automated SEC EDGAR fetching
- [ ] Advanced analytics dashboard
- [ ] API deployment

### Long Term
- [ ] Real-time monitoring system
- [ ] Comparative sector analysis
- [ ] Predictive insights
- [ ] Multi-language support
