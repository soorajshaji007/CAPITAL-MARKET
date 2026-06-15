# Capital Markets Advisory Agent - Complete System Working

## 📌 Executive Summary

The **Capital Markets Advisory Agent** is a full-stack AI application that provides professional financial advisory insights. It combines:
- **FastAPI Backend** - REST API for processing queries
- **React Frontend** - Interactive user interface
- **Local LLM (Llama 3 8B)** - AI model for generating responses
- **ChromaDB** - Vector database for semantic search
- **yfinance** - Real-time stock market data

---

## 🏗️ System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    USER INTERFACE (React)                       │
│  • Chat Interface for advisory questions                        │
│  • Stock Search component with autocomplete                     │
│  • Documentation display                                        │
│  • System status monitoring                                     │
└─────────────────────────────────────────────────────────────────┘
                         ↕ (HTTP REST API)
┌─────────────────────────────────────────────────────────────────┐
│                 FASTAPI BACKEND (Python)                        │
│  • /api/query - Process queries                                 │
│  • /api/initialize - Initialize system                          │
│  • /api/rebuild - Rebuild vector DB                             │
│  • /api/stocks/search - Stock lookup                            │
│  • /api/status - System health check                            │
└─────────────────────────────────────────────────────────────────┘
                         ↕
┌─────────────────────────────────────────────────────────────────┐
│             CAPITAL MARKETS ADVISORY AGENT (RAG)                │
│  • Query orchestration                                          │
│  • Document retrieval via ChromaDB                              │
│  • Stock ticker detection                                       │
│  • LLM prompt building                                          │
│  • Response generation and formatting                           │
└─────────────────────────────────────────────────────────────────┘
                         ↕
┌─────────────────────────────────────────────────────────────────┐
│                   AI & DATA LAYER                               │
│  ┌─────────────────────────┐  ┌──────────────────────────┐     │
│  │ LOCAL MODELS (GGUF)     │  │ EXTERNAL DATA SOURCES    │     │
│  │ • LLM Generation Model  │  │ • yfinance (stocks)      │     │
│  │ • Embedding Model       │  │ • NSE/BSE ticker DB      │     │
│  └─────────────────────────┘  └──────────────────────────┘     │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ CHROMADB VECTOR DATABASE                              │    │
│  │ • Embedded documents (capital markets docs)            │    │
│  │ • Semantic similarity search capability                │    │
│  │ • Persistent storage on disk                           │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🎯 Key Components Explanation

### 1. **Frontend (React TypeScript)**

#### Components:

**App.tsx** - Main Entry Point
```
- Manages global state (systemStatus, loading, error, showDocs)
- On startup: checks system status via /api/status
- If not initialized: shows initialization button
- Routes between Chat, StockSearch, and Documentation views
- Handles system initialization flow
```

**Chat.tsx** - Advisory Query Interface
```
- Manages conversation history with multiple chat sessions
- User inputs queries → API calls /api/query
- Displays assistant responses with structured formatting
- Parses responses into sections (Price Analysis, Fundamentals, Recommendation)
- Real-time message updates with timestamps
- Error handling and reconnection logic
```

**StockSearch.tsx** - Stock Lookup Component
```
- Real-time stock search with autocomplete
- Calls /api/stocks/search endpoint
- Validates stock tickers (NSE/BSE format)
- Shows dropdown with matching results
- Keyboard navigation (arrow keys, enter)
- Selected ticker is passed to chat queries
```

**Sidebar.tsx** - Navigation & Controls
```
- Navigation between features
- System status display
- Initialize/Rebuild button triggers
```

**RebuildModal.tsx** - Confirmation Dialog
```
- Confirmation before rebuilding vector DB
- Progress indication during rebuild
```

#### Frontend Data Flow:
```
User Input 
  ↓
Chat/StockSearch Component 
  ↓
apiService (HTTP POST/GET) 
  ↓
Backend API Endpoint 
  ↓
Response processed and displayed
```

---

### 2. **Backend API (FastAPI)**

#### Startup Process (`main.py`):

```python
@app.on_event("startup")
async def startup_event():
    # 1. Initialize Vector Database
    vector_db = initialize_vector_db(doc_path)
    
    # 2. Initialize Capital Markets Agent
    agent = CapitalMarketsAdvisoryAgent(vector_db)
    
    # System ready for queries
```

#### Key Endpoints:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/` | GET | Health check (returns running status) |
| `/api/health` | GET | Detailed system health |
| `/api/status` | GET | System operational status |
| `/api/initialize` | POST | Initialize vector DB & agent |
| `/api/rebuild` | POST | Force rebuild vector database |
| `/api/query` | POST | Process advisory query |
| `/api/stocks/search` | GET | Search stocks by name/ticker |

#### Query Processing Flow:

```
POST /api/query
  ↓
Check if agent is initialized
  ↓
Extract question from request
  ↓
Call agent.answer_question(question)
  ↓
Generate response
  ↓
Return QueryResponse (answer + status)
```

---

### 3. **Capital Markets Advisory Agent** (`agent.py`)

#### Core Responsibilities:

**Initialization:**
```python
agent = CapitalMarketsAdvisoryAgent(vector_db)
# Stores reference to vector_db for context retrieval
# Lazily loads LLM model on first query
```

**Query Processing (`answer_question`):**

```
Input: question string
  ↓
Step 1: Detect if stock ticker mentioned in question
  Pattern: SYMBOL.NS or SYMBOL.BO (e.g., INFY.NS, RELIANCE.BO)
  ↓
Step 2: Branch logic
  If ticker found:
    → Call analyze_stock(ticker, question)
  Else:
    → General capital markets query
  ↓
Step 3: LLM Response Generation
  Build prompt with system instructions
  → Generate response with Llama 3 8B
  → Clean and format response
  ↓
Output: Professional advisory answer
```

#### Stock Analysis Flow:

```
analyze_stock(ticker, question) called
  ↓
Step 1: Normalize ticker (handle .NS/.BO suffixes)
  ↓
Step 2: Fetch stock price data via yfinance
  - Current price, previous close, open, high, low, volume
  - Historical data (2 months)
  ↓
Step 3: Build price trend summary
  - Calculate % change from previous close
  - Trend analysis from historical data
  ↓
Step 4: Retrieve relevant documents from ChromaDB
  - Search for context about company/sector
  ↓
Step 5: Build comprehensive prompt with:
  - Ticker info
  - Price trends
  - Document context
  - User question
  ↓
Step 6: Generate LLM response with structured output:
  - Price and Trend Analysis
  - Fundamentals Analysis
  - Recommendation
  ↓
Output: Detailed stock analysis
```

#### Prompt Structure:

```
System Message:
  "You are a professional capital markets advisory analyst..."
  "Provide clear, accurate professional answers..."
  
User Message:
  "Question: [USER_QUESTION]"
  "[CONTEXT: Retrieved documents]"
  "[STOCK DATA: If analyzing stock]"

Model: Meta-Llama-3-8B-Instruct
Temperature: 0.1 (deterministic)
Max Tokens: 556
```

---

### 4. **Vector Database** (`vector_db.py`)

#### Initialization:

```python
class CapitalMarketsVectorDB:
    def __init__(self):
        # 1. Load embedding model
        self.embeddings = LlamaCppEmbeddings(
            model_path="models/nomic-embed-text-v1.5.Q4_K_M.gguf",
            n_ctx=5048
        )
        
        # 2. Initialize ChromaDB client
        self.chroma_client = chromadb.PersistentClient(
            path="chroma_db/"
        )
        
        # 3. Create Langchain Chroma vector store
        self.vector_store = Chroma(
            client=self.chroma_client,
            collection_name="capital_markets_docs",
            embedding_function=self.embeddings
        )
```

#### Document Loading Process:

```
load_documents(doc_path)
  ↓
Step 1: Read all PDFs/text files from doc_path
  Example: data/capital_markets_docs/
  ↓
Step 2: Parse document content
  - PDF: Extract text using pypdf
  - Text: Direct read
  ↓
Step 3: Split into chunks
  - Chunk size: 1400 characters
  - Overlap: 400 characters
  - Recursive splitter for context preservation
  ↓
Step 4: Generate embeddings
  - Each chunk embedded using nomic-embed-text
  - Vector dimension: 768
  ↓
Step 5: Store in ChromaDB
  - Persistent storage in chroma_db/
  - Metadata includes source, chunk index
  ↓
Document collection created and indexed
```

#### Semantic Search:

```
search(query, k=10)
  ↓
Step 1: Embed user query
  query_embedding = embedding_model(query)
  ↓
Step 2: Find similar documents
  similar_docs = vector_store.similarity_search(
    query=query,
    k=k  # Return top k results
  )
  ↓
Step 3: Return context to agent
  [doc1_content, doc2_content, ...]
  
Relevance based on cosine similarity of embeddings
```

---

### 5. **Stock Data Management** (`stock_lookup.py`)

#### Stock Database:
- Maintains NSE (National Stock Exchange) tickers
- Maintains BSE (Bombay Stock Exchange) tickers
- Format: SYMBOL.NS or SYMBOL.BO

#### Key Functions:

```python
is_valid_ticker(ticker: str) → bool
  # Check if ticker exists in database
  # Example: "INFY.NS" → True

find_stock_by_name(query: str) → list[Stock]
  # Search by company name
  # Example: "Infosys" → [{"name": "Infosys", "ticker": "INFY.NS"}]

get_similar_stocks(query: str, limit: int) → list[Stock]
  # Find similar company names (fuzzy matching)
  # Returns top `limit` matches
```

---

### 6. **Configuration** (`config.py`)

```python
class Config:
    # Model Paths
    LLM_MODEL_PATH = "models/Meta-Llama-3-8B-Instruct.Q4_K_M.gguf"
    EMBEDDING_MODEL = "models/nomic-embed-text-v1.5.Q4_K_M.gguf"
    
    # Database
    CHROMA_DB_PATH = "chroma_db"
    COLLECTION_NAME = "capital_markets_docs"
    
    # Document Processing
    DOCUMENTS_PATH = "data"
    CHUNK_SIZE = 1400
    CHUNK_OVERLAP = 400
    TOP_K_RESULTS = 10
    
    # Loaded from environment variables (.env file)
```

---

## 🔄 Complete User Journey

### Scenario: User asks about Infosys stock

```
1. FRONTEND - User Input
   └─ User types: "What is the outlook for INFY.NS?"
   
2. FRONTEND - Chat Component
   └─ Calls: apiService.queryAgent("What is the outlook for INFY.NS?")
   
3. FRONTEND - API Service
   └─ Makes HTTP POST to http://localhost:8000/api/query
   └─ Payload: {"question": "What is the outlook for INFY.NS?"}
   
4. BACKEND - FastAPI Route Handler
   └─ Receives POST /api/query
   └─ Extracts question from request
   └─ Calls: agent.answer_question(question)
   
5. BACKEND - Agent: answer_question()
   └─ Detects ticker: "INFY.NS" using regex
   └─ Calls: analyze_stock("INFY.NS", question)
   
6. BACKEND - Agent: analyze_stock()
   └─ Step A: Normalize ticker → "INFY.NS"
   └─ Step B: Fetch stock price via yfinance
      ├─ Current price: ₹1850
      ├─ Previous close: ₹1820
      ├─ Volume: 15M shares
      └─ 2-month history for trend
   
   └─ Step C: Retrieve context from ChromaDB
      ├─ Build query embedding
      ├─ Search for similar documents
      ├─ Retrieve top 10 relevant chunks
      └─ Context: "Infosys is a global IT consulting..."
   
   └─ Step D: Build comprehensive prompt
      ├─ System: Agent role + instructions
      ├─ Stock data: Prices, trends
      ├─ Document context: Company info
      └─ User question: "What is the outlook..."
   
   └─ Step E: LLM Generation
      ├─ Load Llama 3 8B model (if not loaded)
      ├─ Run inference with 180s timeout
      ├─ Model generates response:
      │  "Price and Trend Analysis:
      │   INFY is trading at ₹1850, up 1.6% from previous close.
      │   Shows strong uptrend over 2 months...
      │   
      │   Fundamentals Analysis:
      │   Infosys is a leading IT consulting firm...
      │   
      │   Recommendation:
      │   Bullish outlook for IT sector players..."
      │
      ├─ Clean response (remove artifacts)
      └─ Extract structured output
   
7. BACKEND - Response Formatting
   └─ Clean up response text
   └─ Remove metadata and citations
   └─ Format for display
   
8. BACKEND - Return to Frontend
   └─ Returns QueryResponse:
      {
        "answer": "[formatted stock analysis]",
        "status": "success"
      }
   
9. FRONTEND - Display Response
   └─ Chat component receives response
   └─ Parses response into sections:
      ├─ "Price and Trend Analysis"
      ├─ "Fundamentals Analysis"
      └─ "Recommendation"
   └─ Displays with formatted styling
   
10. USER - Views Result
    └─ Professional stock analysis displayed
    └─ Can continue chatting or search new stock
```

---

## 🚀 System Initialization Flow

### First Launch:

```
User visits http://localhost:3000
  ↓
Frontend: checkSystemStatus()
  ↓
API: GET /api/status
  ↓
Backend Startup Event Triggered:
  1. Load embedding model (nomic-embed-text)
  2. Initialize ChromaDB client
  3. Load documents from data/ directory
  4. Create vector embeddings for all chunks
  5. Store in persistent ChromaDB database
  6. Initialize LLM reference (lazy load)
  7. Create CapitalMarketsAdvisoryAgent
  └─ System ready (1-2 minutes on first run)
  
Frontend receives: StatusResponse
  ├─ agent_loaded: true
  ├─ vector_db_loaded: true
  ├─ total_chunks: 45 (e.g.)
  
UI displays:
  ├─ Chat interface enabled
  ├─ Stock search enabled
  └─ Ready for queries
```

### Rebuild Vector Database:

```
User clicks "Rebuild" button
  ↓
Frontend: apiService.rebuildVectorDB()
  ↓
API: POST /api/rebuild with force=true
  ↓
Backend:
  1. Delete existing ChromaDB directory
  2. Reload all documents from scratch
  3. Re-generate all embeddings
  4. Rebuild indexes
  5. Return success status
  
Takes 1-2 minutes
```

---

## 💾 Data Storage

### Directory Structure:

```
Capital Market Agent/
├── backend/
│   ├── models/
│   │   ├── Meta-Llama-3-8B-Instruct.Q4_K_M.gguf    (4.7 GB)
│   │   └── nomic-embed-text-v1.5.Q4_K_M.gguf       (80 MB)
│   ├── data/
│   │   └── capital_markets_docs/
│   │       └── [capital_markets_guide.txt]
│   ├── chroma_db/
│   │   ├── chroma.sqlite3                            (Vector DB)
│   │   └── [embedded collections]
│   └── [Python code files]
│
├── frontend/
│   └── [React TypeScript code]
│
└── [Configuration & Documentation]
```

### ChromaDB Storage:

```
chroma_db/
├── chroma.sqlite3
│   └─ Collection: "capital_markets_docs"
│      └─ Documents (embeddings + metadata)
│         Example entry:
│         {
│           id: "uuid-1234",
│           embedding: [0.123, -0.456, 0.789, ...],  (768 dims)
│           document: "Chunk of capital markets text",
│           metadata: {
│             source: "capital_markets_guide.txt",
│             chunk_index: 0
│           }
│         }
```

---

## 🔌 API Contracts

### Query Request/Response

```json
REQUEST POST /api/query
{
  "question": "What are the top IPO opportunities in 2024?",
  "k": 10,                    // optional: number of docs to retrieve
  "stock_ticker": "INFY.NS"  // optional: specific stock to analyze
}

RESPONSE 200 OK
{
  "answer": "Based on capital markets analysis...",
  "status": "success"
}
```

### Stock Search Request/Response

```json
REQUEST GET /api/stocks/search?query=Infosys

RESPONSE 200 OK
{
  "query": "Infosys",
  "results": [
    {
      "name": "Infosys Limited",
      "ticker": "INFY.NS"
    },
    {
      "name": "Infosys Limited",
      "ticker": "INFY.BO"
    }
  ],
  "status": "success"
}
```

### Status Request/Response

```json
REQUEST GET /api/status

RESPONSE 200 OK
{
  "status": "operational",
  "message": "System operational",
  "vector_db_loaded": true,
  "agent_loaded": true,
  "total_chunks": 45
}
```

---

## ⚙️ Technical Stack

### Frontend
- **React 18** - UI framework
- **TypeScript** - Type safety
- **Vite** - Build tool
- **Tailwind CSS** - Styling
- **Lucide Icons** - Icon library
- **Axios** - HTTP client

### Backend
- **FastAPI** - Web framework
- **Uvicorn** - ASGI server
- **Pydantic** - Data validation
- **Python 3.10+** - Language

### AI/ML
- **Meta-Llama-3-8B-Instruct** - LLM model (GGUF format)
- **nomic-embed-text-v1.5** - Embedding model (GGUF format)
- **llama-cpp-python** - LLM inference
- **LangChain** - AI framework
- **ChromaDB** - Vector database
- **Sentence Transformers** - Embeddings (fallback)

### Data
- **yfinance** - Stock price data
- **pypdf** - PDF parsing
- **RecursiveCharacterTextSplitter** - Text chunking

---

## 🔐 Error Handling

### Backend Error Scenarios:

```python
# Model not found
if not os.path.exists(model_path):
    return "Local LLM model not available"

# LLM generation timeout (180s)
try:
    result = executor.submit(generate, prompt).result(timeout=180)
except TimeoutError:
    return "LLM generation timed out"

# ChromaDB corruption
except BaseException:
    self._reset_chroma_db()  # Rebuild database

# Query processing error
except Exception as e:
    raise HTTPException(status_code=500, detail=str(e))
```

### Frontend Error Handling:

```typescript
// Connection error
if connection fails:
  → Display error message
  → Show "Try Again" button
  → Suggest checking backend server

// Query timeout
if response takes > 5 minutes:
  → Display timeout message
  → Allow retry

// Stock search error
if search fails:
  → Display error notification
  → Clear search results
```

---

## 📊 Performance Characteristics

### Timing

| Operation | Time |
|-----------|------|
| System startup (first time) | 1-2 minutes |
| Embedding model load | 30-60 seconds |
| Document chunking & embedding | 30-90 seconds |
| LLM model load (first query) | 30-60 seconds |
| Normal query response | 10-30 seconds |
| Stock price fetch | 2-5 seconds |
| Vector search | <1 second |

### Resource Usage

| Component | Memory | GPU |
|-----------|--------|-----|
| LLM (Llama 3 8B) | 4-6 GB | CPU only |
| Embedding model | 100-200 MB | CPU only |
| ChromaDB + Indexes | 200-500 MB | N/A |
| Application & OS | 500 MB - 1 GB | N/A |
| **Total** | **6-8 GB** | Not required |

---

## 🎓 Key Insights

1. **RAG Architecture**: System uses Retrieval-Augmented Generation to combine vector search with LLM generation for accurate, context-aware responses

2. **Local Models**: All AI models run locally (GGUF format) - no cloud dependency, privacy-preserving

3. **Lazy Loading**: LLM model loaded only on first query to reduce startup time

4. **Timeout Protection**: 180s timeout on LLM generation prevents indefinite hangs

5. **Dual Mode**: Can handle general capital markets questions and detailed stock analysis

6. **Structured Output**: Responses formatted into sections for better readability

7. **Scalability**: ChromaDB allows easy addition of more documents

8. **Error Recovery**: Automatic ChromaDB rebuild on corruption

---

## 🔧 Configuration Options

Customize via `.env` file:

```env
# Model Paths
LLM_MODEL_PATH=models/Meta-Llama-3-8B-Instruct.Q4_K_M.gguf
EMBEDDING_MODEL_PATH=models/nomic-embed-text-v1.5.Q4_K_M.gguf

# Database
CHROMA_DB_PATH=chroma_db
COLLECTION_NAME=capital_markets_docs

# Document Processing
DOCUMENTS_PATH=data
CHUNK_SIZE=1400
CHUNK_OVERLAP=400
TOP_K_RESULTS=10

# LLM Parameters (in code)
N_THREADS=8
N_GPU_LAYERS=0 (set to >0 if using GPU)
TEMPERATURE=0.1
MAX_TOKENS=556
```

---

## 📚 Summary

The Capital Markets Advisory Agent is a sophisticated AI system that:
- **Processes** natural language queries about capital markets
- **Retrieves** relevant context from embedded documents
- **Analyzes** real-time stock data
- **Generates** professional advisory responses
- **Displays** structured, easy-to-understand results

All through a seamless full-stack application combining modern frontend (React) with powerful backend (FastAPI + local LLMs).

