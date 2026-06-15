# Capital Markets Advisory Agent - Quick Visual Reference

## 🎯 System Data Flow Diagram

```
                          USER INTERFACE (React)
                    ┌─────────────────────────────┐
                    │   Chat Interface            │
                    │   • Input question          │
                    │   • View responses          │
                    │   • Message history         │
                    └────────┬────────────────────┘
                             │
                             ↓ HTTP POST /api/query
                    ┌─────────────────────────────┐
                    │   FastAPI Backend           │
                    │   • Route Handler           │
                    │   • Validate request        │
                    │   • Call Agent              │
                    └────────┬────────────────────┘
                             │
                             ↓
                    ┌─────────────────────────────┐
                    │   Agent: answer_question()  │
                    │   • Detect ticker           │
                    │   • Decide query type       │
                    └────────┬────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ↓                   ↓                   ↓
    General Q&A        Stock Analysis      Document Context
    (All queries)      (SYMBOL.NS/BO)      (All queries)
         │                   │                   │
         └───┬───────────────┼───────────────┬───┘
             │               │               │
             ↓               ↓               ↓
    ┌────────────────┐ ┌────────────┐ ┌──────────────┐
    │  Build Prompt  │ │  yfinance  │ │   ChromaDB   │
    │  • System msg  │ │  Fetch     │ │  • Search    │
    │  • Question    │ │  • Price   │ │  • Retrieve  │
    │  • Context     │ │  • Trend   │ │  • Context   │
    └────────┬───────┘ └──────┬─────┘ └──────┬───────┘
             │                │              │
             └────────────────┼──────────────┘
                              │
                              ↓
                    ┌─────────────────────────────┐
                    │   Llama 3 8B LLM            │
                    │   • Generate response       │
                    │   • 556 max tokens          │
                    │   • 180s timeout            │
                    └────────┬────────────────────┘
                             │
                             ↓
                    ┌─────────────────────────────┐
                    │   Response Processing       │
                    │   • Clean text              │
                    │   • Extract structure       │
                    │   • Format sections         │
                    └────────┬────────────────────┘
                             │
                             ↓ QueryResponse
                    ┌─────────────────────────────┐
                    │   Backend Returns           │
                    │   JSON with answer          │
                    └────────┬────────────────────┘
                             │
                             ↓ HTTP Response
                    ┌─────────────────────────────┐
                    │   Frontend Display          │
                    │   • Parse response          │
                    │   • Show sections           │
                    │   • Format nicely           │
                    └─────────────────────────────┘
```

---

## 🔄 Stock Analysis Detailed Flow

```
User Query: "What is the outlook for INFY.NS?"
                    │
                    ↓
            ┌──────────────────┐
            │ Ticker Detection │
            │ Regex: (\w+)\.(NS|BO)
            │ Finds: INFY.NS   │
            └────────┬─────────┘
                     │
                     ↓
         ┌───────────────────────┐
         │ Fetch Stock Data      │
         │ (yfinance)            │
         │ • Current: ₹1850      │
         │ • Previous: ₹1820     │
         │ • Volume: 15M         │
         │ • 2-month history     │
         └────────┬──────────────┘
                  │
                  ↓
        ┌─────────────────────┐
        │ Vector DB Search    │
        │ (ChromaDB)          │
        │ Query: "Infosys"    │
        │ Return: Top 10 docs │
        └────────┬────────────┘
                 │
                 ↓
    ┌─────────────────────────────┐
    │ Build Stock Analysis Prompt │
    │                             │
    │ System:                     │
    │  "You are advisor..."       │
    │                             │
    │ Stock Data:                 │
    │  INFY.NS: ₹1850 (+1.6%)    │
    │  Trend: Bullish             │
    │                             │
    │ Context:                    │
    │  "Infosys is IT firm..."   │
    │                             │
    │ Question:                   │
    │  "What is outlook?"         │
    └────────┬────────────────────┘
             │
             ↓
    ┌──────────────────────────┐
    │ Llama 3 Inference        │
    │ Temperature: 0.1         │
    │ Max Tokens: 556          │
    │ Timeout: 180s            │
    └────────┬─────────────────┘
             │
             ↓
    Raw Output:
    "Price and Trend Analysis:
     INFY trading at ₹1850...
     
     Fundamentals Analysis:
     Infosys is a leading...
     
     Recommendation:
     Bullish outlook..."
             │
             ↓
    ┌──────────────────┐
    │ Clean Response   │
    │ Extract Sections │
    │ Format Output    │
    └────────┬─────────┘
             │
             ↓
    Structured Response:
    {
      "Price and Trend Analysis": "...",
      "Fundamentals Analysis": "...",
      "Recommendation": "..."
    }
             │
             ↓
    Display to User
```

---

## 📊 Component Interaction Matrix

```
                 Backend API  Agent      Vector DB  Stock Lookup  LLM
Frontend           ✓          -           -          -            -
Backend API        -          ✓           ✓          ✓            ✓
Agent              -          -           ✓          ✓            ✓
Vector DB          -          -           -          -            -
Stock Lookup       -          -           -          -            -
LLM                -          -           -          -            -

Legend:
✓ = Component A calls/uses Component B
- = No direct interaction
```

---

## 🔄 State Transitions

```
System Startup:
┌──────────┐   Load Embedding Model   ┌─────────────┐
│          │  ─────────────────────→  │             │
│ Starting │                          │ Initializing│
│          │  ←─────────────────────  │             │
└──────────┘   Initialize ChromaDB    └─────────────┘
                    │
                    ↓
            Load Documents from Disk
                    │
                    ↓
            Create Vector Embeddings
                    │
                    ↓
            Store in ChromaDB
                    │
                    ↓
        ┌───────────────────┐
        │   Ready to Query  │
        └─────────┬─────────┘
                  │
    ┌─────────────┼─────────────┐
    │             │             │
    ↓             ↓             ↓
Processing   Rebuilding    Healthy
  Query      Vector DB     (Idle)
    │             │             │
    └─────────────┼─────────────┘
                  ↓
            Ready Again
```

---

## 📝 API Endpoint Summary

```
┌──────────────────────────────────────────────────────────────┐
│ HEALTH & STATUS                                              │
├──────────────────────────────────────────────────────────────┤
│ GET  /                  → Service running status             │
│ GET  /api/health        → System health (db, agent, chunks)  │
│ GET  /api/status        → Operational status                 │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ SYSTEM MANAGEMENT                                            │
├──────────────────────────────────────────────────────────────┤
│ POST /api/initialize    → Initialize vector DB & agent       │
│ POST /api/rebuild       → Rebuild vector database            │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ ADVISORY QUERIES                                             │
├──────────────────────────────────────────────────────────────┤
│ POST /api/query         → Process capital markets query      │
│                           (auto-detects stock if mentioned)  │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ STOCK OPERATIONS                                             │
├──────────────────────────────────────────────────────────────┤
│ GET  /api/stocks/search → Search stocks by name/ticker       │
└──────────────────────────────────────────────────────────────┘
```

---

## 🎯 Query Type Classification

```
User Input Analysis:
│
├─ Contains "SYMBOL.NS" or "SYMBOL.BO"?
│  │
│  ├─ YES → Stock Analysis Branch
│  │        • Fetch current price
│  │        • Analyze trend
│  │        • Retrieve context
│  │        • Generate stock report
│  │
│  └─ NO → General Q&A Branch
│          • Retrieve relevant documents
│          • Generate advisory answer
│          • Validate capital markets topic
│
└─ Invalid capital markets question?
   │
   └─ YES → Respond: "I can only answer capital
            markets advisory questions"
```

---

## 💾 Data Processing Pipeline

```
RAW DATA INGESTION
│
├─ PDFs from data/capital_markets_docs/
└─ Text files from data/

    PROCESSING STAGE 1: LOADING
    │
    ├─ PDF Parsing (pypdf)
    └─ Text File Reading

    PROCESSING STAGE 2: CHUNKING
    │
    ├─ Split into 1400-char chunks
    └─ 400-char overlap between chunks

    PROCESSING STAGE 3: EMBEDDING
    │
    ├─ Process each chunk
    ├─ nomic-embed-text model
    └─ Generate 768-dim vectors

    PROCESSING STAGE 4: STORAGE
    │
    ├─ Store embeddings
    ├─ ChromaDB persistent database
    └─ chroma_db/ directory on disk

READY FOR SEMANTIC SEARCH
│
└─ Query embedding → Cosine similarity → Top-k results
```

---

## ⚡ Performance Timeline (First Run)

```
0:00    System started
0:15    Embedding model loaded
0:45    Documents loaded and chunked
1:30    Embeddings generated for all chunks
2:00    ChromaDB initialized and indexed
2:00    System ready for queries
    ↓
    [User starts querying]
    ↓
2:30    LLM model loaded (first query only)
2:45    Query processed and response generated
2:50    Response displayed

Subsequent queries:
│
├─ Vector search: <1s
├─ LLM inference: 10-30s
└─ Response display: <1s
```

---

## 🔧 Configuration Hierarchy

```
DEFAULT VALUES (config.py)
    │
    ↓ OVERRIDE BY
    
ENVIRONMENT VARIABLES (.env file)
    │
    ├─ LLM_MODEL_PATH
    ├─ EMBEDDING_MODEL_PATH
    ├─ CHROMA_DB_PATH
    ├─ DOCUMENTS_PATH
    ├─ CHUNK_SIZE
    ├─ CHUNK_OVERLAP
    └─ TOP_K_RESULTS
    │
    ↓ USED BY
    
RUNTIME SYSTEM
    │
    ├─ Backend initialization
    ├─ Model loading
    ├─ Database setup
    └─ Query processing
```

---

## 🚨 Error Recovery Paths

```
LLM Model Not Found
├─ Detection: File path check
├─ Response: "Local LLM model not available"
└─ Resolution: Download model file

ChromaDB Corruption
├─ Detection: Connection failure
├─ Action: Reset and rebuild database
├─ Impact: 1-2 minute rebuild
└─ Recovery: Automatic

Query Timeout (180s)
├─ Detection: LLM generation exceeds limit
├─ Response: "LLM generation timed out"
└─ Resolution: Try again / Use simpler query

Stock Data Unavailable
├─ Detection: yfinance fetch fails
├─ Impact: Use query without price data
└─ Response: Continue with analysis

Vector Search Empty
├─ Detection: No relevant docs found
├─ Response: Use LLM general knowledge
└─ Result: Still provide answer
```

---

## 📱 Frontend Component Tree

```
App.tsx (Main)
├─ Sidebar.tsx
│  ├─ Navigation buttons
│  ├─ Status display
│  └─ Rebuild button
├─ Chat.tsx
│  ├─ MessageBubble (User)
│  ├─ MessageBubble (Assistant)
│  ├─ StockSearch.tsx
│  │  ├─ Input field
│  │  ├─ Dropdown results
│  │  └─ Stock suggestions
│  ├─ Input form
│  └─ Send button
└─ Documentation.tsx
   └─ Markdown display
```

---

## 🎓 Key Concepts

### RAG (Retrieval-Augmented Generation)
```
Query → Vector Search → Retrieved Docs → LLM Prompt → Response
```

### Vector Embeddings
```
Text: "Infosys is an IT company"
  ↓ (nomic-embed-text model)
Vector: [0.123, -0.456, 0.789, ..., 0.234] (768 dimensions)
```

### ChromaDB Collections
```
Collection: "capital_markets_docs"
├─ Document: Chunk 1
├─ Document: Chunk 2
├─ Document: Chunk 3
└─ Document: Chunk N
```

### GGUF Format
```
Quantized Model Files:
├─ LLM: 4.7 GB (8B parameters, quantized to 4-bit)
└─ Embeddings: 80 MB (compressed)

Benefits:
├─ Runs on CPU (no GPU needed)
├─ Low memory footprint
└─ Fast inference
```

---

## 📊 System Resource Usage

```
Memory Allocation (approx):
├─ LLM Model: 4-6 GB
├─ Embedding Model: 100-200 MB
├─ ChromaDB Indexes: 200-500 MB
├─ Application Runtime: 300-500 MB
└─ OS & Buffer: 500-1000 MB
   ═══════════════════════════
   Total: ~6-8 GB RAM

Processing Power:
├─ Model Inference: CPU (8+ threads recommended)
├─ Embedding Generation: CPU
├─ Vector Search: CPU
└─ No GPU required
```

---

## ✅ System Health Indicators

```
Healthy System:
├─ ✓ Backend running (Port 8000)
├─ ✓ Frontend running (Port 3000)
├─ ✓ Vector DB loaded
├─ ✓ Agent initialized
├─ ✓ LLM available
└─ ✓ All endpoints responding

Warning Signs:
├─ ⚠ Queries taking > 60s
├─ ⚠ Stock search returning no results
├─ ⚠ High memory usage (>10 GB)
└─ ⚠ API response errors

Critical Issues:
├─ ✗ Backend not running
├─ ✗ ChromaDB corrupted
├─ ✗ Models missing
└─ ✗ Frontend cannot connect
```

---

This quick reference provides visual understanding of how all components work together!

