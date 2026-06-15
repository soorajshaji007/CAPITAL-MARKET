# Capital Markets Advisory Agent - Technical Deep Dive

## 🔬 Detailed Implementation Guide

---

## 1. Frontend Architecture

### App.tsx - Root Component Structure

```typescript
// State Management
const [systemStatus, setSystemStatus] = useState<StatusResponse | null>(null);
const [loading, setLoading] = useState<boolean>(true);
const [error, setError] = useState<string | null>(null);
const [showDocs, setShowDocs] = useState<boolean>(false);

// Lifecycle
useEffect(() => {
  checkSystemStatus();  // Runs once on mount
}, []);

// Conditional Rendering
if (loading) → Loading spinner
if (error) → Error message + retry button
if (!systemStatus?.agent_loaded) → Init button
else → Main UI (Chat + Sidebar)
```

### Chat Component - Message Management

```typescript
interface Conversation {
  id: string;                // UUID
  title: string;             // Auto-generated from first message
  messages: ChatMessage[];    // Chat history
  createdAt: string;          // ISO timestamp
}

interface ChatMessage {
  content: string;
  role: 'user' | 'assistant';
  timestamp?: string;
  isError?: boolean;
}

// Message Flow
User types → onChange handler → state update
Submit button → apiService.queryAgent() → HTTP POST
Response received → Add to messages array → Re-render
```

### Response Parsing Logic

```typescript
const parseStructuredResponse = (content: string) => {
  // Look for patterns like:
  // "Price and Trend Analysis: ..."
  // "Fundamentals Analysis: ..."
  // "Recommendation: ..."
  
  const headerRegex = /^\s*\*{0,2}\s*(Price and Trend Analysis|Fundamentals Analysis|Recommendation)\s*\*{0,2}\s*:?\s*(.*)$/i;
  
  // Split response into sections
  // Return array of {title, content} objects
  // Render each section with formatted styling
};
```

### Stock Search Component

```typescript
// Real-time Search Flow
Input value changes
  ↓
handleSearch() called with new query
  ↓
apiService.searchStocks(query)
  ↓
Debounced API call (avoid hammering backend)
  ↓
Dropdown shows matching results
  ↓
User selects stock
  ↓
onSelect callback fired
  ↓
Selected ticker becomes context for next query

// Keyboard Navigation
ArrowDown/Up → Highlight results
Enter → Select highlighted item
Escape → Close dropdown
```

### API Service (Axios)

```typescript
const api = axios.create({
  baseURL: 'http://localhost:8000/api',
  headers: { 'Content-Type': 'application/json' },
  timeout: 300000,  // 5 minutes for slow LLM inference
});

// Each method returns Promise<T>
apiService.queryAgent() 
  → POST /api/query 
  → Returns QueryResponse

apiService.searchStocks()
  → GET /api/stocks/search
  → Returns StockSearchResponse

apiService.rebuildVectorDB()
  → POST /api/rebuild
  → Returns StatusResponse
```

---

## 2. Backend Architecture

### FastAPI Application Initialization

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

# Create app with metadata
app = FastAPI(
    title="Capital Markets Advisory Agent API",
    version="1.0.0"
)

# Enable CORS for frontend communication
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Allow all origins
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Global state
vector_db: Optional[CapitalMarketsVectorDB] = None
agent: Optional[CapitalMarketsAdvisoryAgent] = None

# Startup hook
@app.on_event("startup")
async def startup_event():
    global vector_db, agent
    vector_db = initialize_vector_db(config.DOCUMENTS_PATH)
    agent = CapitalMarketsAdvisoryAgent(vector_db)
```

### Request/Response Models (Pydantic)

```python
class QueryRequest(BaseModel):
    question: str
    k: Optional[int] = None  # Number of docs to retrieve
    stock_ticker: Optional[str] = None  # Specific stock

class QueryResponse(BaseModel):
    answer: str
    status: str  # "success" or "error"

class StatusResponse(BaseModel):
    status: str
    message: str
    vector_db_loaded: bool
    agent_loaded: bool
    total_chunks: int

# Pydantic handles:
# - Validation
# - Type checking
# - JSON serialization
# - OpenAPI documentation
```

### Route Handler Examples

```python
@app.post("/api/query", response_model=QueryResponse, tags=["Advisory"])
async def query_agent(request: QueryRequest):
    global agent
    
    # Validation
    if agent is None:
        raise HTTPException(status_code=503, detail="Agent not initialized")
    
    try:
        # Route based on request
        if request.stock_ticker:
            answer = agent.analyze_stock(
                request.stock_ticker,
                request.question,
                request.k
            )
        else:
            answer = agent.answer_question(request.question, request.k)
        
        # Return structured response
        return QueryResponse(answer=answer, status="success")
    
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))


@app.get("/api/stocks/search", response_model=StockSearchResponse, tags=["Stocks"])
async def search_stocks(query: str):
    if not query or not query.strip():
        raise HTTPException(status_code=400, detail="Search query required")
    
    # Try as ticker first
    if is_valid_ticker(query.upper()):
        results = [{'name': query.upper(), 'ticker': query.upper()}]
    else:
        # Search by company name
        results = get_similar_stocks(query.strip(), limit=10)
    
    return StockSearchResponse(
        query=query.strip(),
        results=results,
        status="success"
    )
```

---

## 3. Agent Implementation Details

### Capital Markets Advisory Agent Class

```python
class CapitalMarketsAdvisoryAgent:
    def __init__(self, vector_db: CapitalMarketsVectorDB):
        self.vector_db = vector_db
        self.llm_model = config.LLM_MODEL_PATH
        self.llm = None  # Lazy load
    
    def _initialize_llm(self):
        """Lazy load LLM on first query"""
        llm = Llama(
            model_path=self.llm_model,
            n_ctx=5048,         # Context window
            n_gpu_layers=0,     # CPU only
            n_threads=8,        # Threading
            n_batch=128,        # Batch size
            verbose=False
        )
        return llm
```

### Ticker Detection Regex

```python
def _detect_ticker_in_question(self, question: str) -> Optional[str]:
    """Detect stock ticker in natural language question"""
    
    # Pattern matches: SYMBOL.NS or SYMBOL.BO
    # Examples: INFY.NS, RELIANCE.BO, TCS.NS
    ticker_pattern = r'\b([A-Z][A-Z0-9]{0,6})\.(?:NS|BO)\b'
    
    matches = re.finditer(ticker_pattern, question, re.IGNORECASE)
    
    for match in matches:
        full_ticker = match.group(0).upper()
        # Validate ticker exists in database
        if is_valid_ticker(full_ticker):
            return full_ticker
    
    return None
```

### Query Routing Logic

```python
def answer_question(self, question: str, k: Optional[int] = None) -> str:
    """Main entry point for queries"""
    
    # Step 1: Detect ticker
    ticker = self._detect_ticker_in_question(question)
    
    # Step 2: Route based on ticker
    if ticker:
        # BRANCH A: Stock-specific analysis
        return self.analyze_stock(ticker, question, k)
    else:
        # BRANCH B: General capital markets question
        prompt = self._build_prompt(question)
        return self._generate_response(prompt)
```

### Stock Analysis Pipeline

```python
def analyze_stock(self, ticker: str, question: str, k: Optional[int] = None) -> str:
    """Detailed stock analysis with price data + document context"""
    
    # Step 1: Normalize ticker
    ticker_candidates = self._normalize_ticker_candidates(ticker)
    # Returns: ["INFY.NS", "INFY.BO"] from "INFY"
    
    # Step 2: Fetch current market data
    price_data = self._fetch_stock_price_data(ticker)
    # Returns: {
    #   "ticker": "INFY.NS",
    #   "current_price": 1850.50,
    #   "previous_close": 1820.00,
    #   "volume": 15000000,
    #   "history": [historical data...]
    # }
    
    # Step 3: Build price trend summary
    trend_summary = self._build_price_trend_summary(price_data)
    # Returns: "Trading at ₹1850, up 1.6% from previous close..."
    
    # Step 4: Retrieve relevant documents
    context = self._retrieve_context(question, k)
    # Returns: Concatenated relevant document chunks
    
    # Step 5: Build comprehensive prompt
    prompt = self._build_stock_analysis_prompt(
        ticker, price_data, trend_summary, context, question
    )
    
    # Step 6: Generate LLM response
    raw_response = self._generate_response(prompt)
    
    # Step 7: Format response
    return self._format_stock_analysis_response(raw_response)
```

### Prompt Building

```python
def _build_prompt(self, question: str) -> str:
    """Build prompt for LLM inference"""
    
    system_msg = (
        "You are a professional capital markets advisory analyst "
        "with deep expertise in financial markets.\n"
        "Provide clear, accurate, and professional answers in plain text.\n"
        "If the question is NOT about capital markets, reply: "
        "'I can only answer capital markets advisory questions.'\n"
        "Do not include source citations, metadata, or references."
    )
    
    # Llama 3 instruction format
    return (
        "<|start_header_id|>system<|end_header_id|>\n"
        f"{system_msg}\n"
        "<|eot_id|><|start_header_id|>user<|end_header_id|>\n"
        f"Question: {question}\n\n"
        f"{context_section if available}"
        "<|eot_id|><|start_header_id|>assistant<|end_header_id|>\n"
    )
```

### LLM Generation with Timeout

```python
def _generate_response(self, prompt: str) -> str:
    """Generate response with timeout protection"""
    
    # Use ThreadPoolExecutor to add timeout
    with ThreadPoolExecutor(max_workers=1) as executor:
        future = executor.submit(
            self._generate_with_llm,
            prompt
        )
        try:
            # 180 second timeout for LLM generation
            result = future.result(timeout=180)
            return result
        except FuturesTimeoutError:
            print("[ERROR] LLM generation timed out (180s limit exceeded)")
            return "LLM generation timed out. Please try again or use a simpler query."
        except Exception as e:
            print(f"[ERROR] Unexpected error: {e}")
            return f"Unexpected error: {e}"


def _generate_with_llm(self, prompt: str) -> str:
    """Internal LLM generation (called with timeout wrapper)"""
    
    if self.llm is None:
        self.llm = self._initialize_llm()
    
    # Llama inference
    generation_result = self.llm(
        prompt,
        max_tokens=556,
        temperature=0.1,        # Deterministic
        top_p=0.9,              # Nucleus sampling
        repeat_penalty=1.1,     # Discourage repetition
        stop=[
            "System:", "User:", 
            "Stock:", "Context:", 
            "Question:"
        ]
    )
    
    # Extract text from result
    response_text = generation_result['choices'][0]['text']
    
    # Clean and format
    response_text = extract_structured_response(response_text)
    response_text = clean_response(response_text)
    
    return response_text
```

---

## 4. Vector Database Implementation

### ChromaDB Initialization

```python
class CapitalMarketsVectorDB:
    def __init__(self):
        # Initialize embedding model
        self.embeddings = LlamaCppEmbeddings(
            model_path=config.EMBEDDING_MODEL,  # nomic-embed-text
            n_ctx=5048,
            n_gpu_layers=0,
            n_batch=8,
            verbose=False
        )
        print("[OK] GGUF embedding model loaded")
        
        # Initialize ChromaDB
        self.chroma_client = chromadb.PersistentClient(
            path=config.CHROMA_DB_PATH,
            settings=Settings(anonymized_telemetry=False)
        )
        
        # Create Langchain vector store wrapper
        self.vector_store = Chroma(
            client=self.chroma_client,
            collection_name=config.COLLECTION_NAME,
            embedding_function=self.embeddings
        )
```

### Document Loading & Chunking

```python
def load_documents(self, doc_path: str) -> List[str]:
    """Load and index documents"""
    
    # Step 1: Discover documents
    documents = []
    for filename in os.listdir(doc_path):
        if filename.endswith(('.txt', '.pdf')):
            filepath = os.path.join(doc_path, filename)
            
            if filename.endswith('.pdf'):
                # PDF parsing
                reader = PdfReader(filepath)
                text = ""
                for page in reader.pages:
                    text += page.extract_text()
            else:
                # Text file reading
                with open(filepath, 'r') as f:
                    text = f.read()
            
            documents.append({
                'content': text,
                'source': filename
            })
    
    # Step 2: Split into chunks
    text_splitter = RecursiveCharacterTextSplitter(
        chunk_size=config.CHUNK_SIZE,        # 1400 chars
        chunk_overlap=config.CHUNK_OVERLAP,  # 400 chars
        separators=["\n\n", "\n", " ", ""]
    )
    
    chunks = []
    for doc in documents:
        split_chunks = text_splitter.split_text(doc['content'])
        for i, chunk in enumerate(split_chunks):
            chunks.append({
                'content': chunk,
                'source': doc['source'],
                'chunk_index': i
            })
    
    # Step 3: Add to vector store (automatic embedding)
    texts = [chunk['content'] for chunk in chunks]
    metadatas = [
        {
            'source': chunk['source'],
            'chunk_index': chunk['chunk_index']
        }
        for chunk in chunks
    ]
    
    ids = self.vector_store.add_texts(
        texts=texts,
        metadatas=metadatas
    )
    
    return ids
```

### Semantic Search

```python
def search(self, query: str, k: int = 10) -> List[Dict]:
    """Find relevant documents"""
    
    # Similarity search returns:
    # - Document content
    # - Metadata (source, chunk_index)
    # - Similarity score
    
    results = self.vector_store.similarity_search_with_score(
        query,
        k=k
    )
    
    # Format results
    formatted = []
    for doc, score in results:
        formatted.append({
            'content': doc.page_content,
            'source': doc.metadata.get('source'),
            'chunk_index': doc.metadata.get('chunk_index'),
            'similarity_score': score
        })
    
    return formatted


def get_collection_count(self) -> int:
    """Get total indexed documents"""
    collection = self.chroma_client.get_collection(
        name=config.COLLECTION_NAME
    )
    return collection.count()
```

---

## 5. Stock Data Management

### Stock Database Structure

```python
# In-memory database of Indian stocks
STOCK_DATABASE = {
    "INFY.NS": {
        "name": "Infosys Limited",
        "sector": "Information Technology",
        "market": "NSE"
    },
    "INFY.BO": {
        "name": "Infosys Limited",
        "sector": "Information Technology",
        "market": "BSE"
    },
    "RELIANCE.NS": {
        "name": "Reliance Industries",
        "sector": "Energy",
        "market": "NSE"
    },
    # ... 1000+ more stocks
}

class StockTicker:
    symbol: str      # INFY
    market: str      # NS or BO
    name: str        # Infosys Limited
    sector: str      # Information Technology
```

### Stock Search Logic

```python
def find_stock_by_name(name: str, limit: int = 5) -> List[Stock]:
    """Fuzzy match company names"""
    from difflib import get_close_matches
    
    names = [stock['name'] for stock in STOCK_DATABASE.values()]
    matches = get_close_matches(name, names, n=limit, cutoff=0.6)
    
    results = []
    for match in matches:
        for ticker, data in STOCK_DATABASE.items():
            if data['name'] == match:
                results.append({
                    'name': data['name'],
                    'ticker': ticker
                })
    
    return results


def is_valid_ticker(ticker: str) -> bool:
    """Check if ticker exists in database"""
    return ticker.upper() in STOCK_DATABASE


def get_similar_stocks(query: str, limit: int = 10) -> List[Stock]:
    """Comprehensive stock search"""
    
    results = []
    query_upper = query.upper()
    
    # Direct ticker match
    if is_valid_ticker(query_upper):
        results.append({
            'name': STOCK_DATABASE[query_upper]['name'],
            'ticker': query_upper
        })
    
    # Company name search
    if len(results) < limit:
        name_matches = find_stock_by_name(query, limit=limit-len(results))
        results.extend(name_matches)
    
    return results[:limit]
```

---

## 6. Data Flow Examples

### Complete Query Lifecycle

```
1. FRONTEND
   User: "What is the outlook for INFY.NS?"
   Interaction: Chat input → Send button
   
2. FRONTEND - API Call
   apiService.queryAgent(question)
   axios.post('http://localhost:8000/api/query', {
     question: "What is the outlook for INFY.NS?"
   })
   
3. BACKEND - Route Handler
   @app.post("/api/query")
   async def query_agent(request: QueryRequest):
       answer = agent.answer_question(request.question)
   
4. BACKEND - Agent Processing
   agent.answer_question("What is the outlook for INFY.NS?")
   
   a) Ticker Detection
      regex.findall(r'([A-Z]+\.(NS|BO))', question)
      detected_ticker = "INFY.NS"
   
   b) Stock Analysis Branch
      agent.analyze_stock("INFY.NS", question)
      
      i) Fetch Price Data
         yfinance.Ticker("INFY.NS").history(period="2mo")
         Returns: {
           current_price: 1850.50,
           previous_close: 1820.00,
           volume: 15000000,
           history: [...]
         }
      
      ii) Build Trend Summary
          "Trading at ₹1850, up 1.6% from previous close.
           Recent trend: Bullish with support at ₹1800"
      
      iii) Retrieve Context
          vector_db.search("Infosys outlook", k=10)
          ChromaDB returns top 10 relevant documents
      
      iv) Build Prompt
          System message + Stock data + Context + Question
      
      v) Generate Response
          llm.generate(prompt, max_tokens=556, temperature=0.1)
          Model generates structured response:
          "Price and Trend Analysis: ...
           Fundamentals Analysis: ...
           Recommendation: ..."
   
   c) Clean Response
      clean_response(raw_output)
      extract_structured_response(raw_output)
   
   d) Return Answer
      return formatted_answer
   
5. BACKEND - HTTP Response
   return QueryResponse(
     answer="[formatted analysis]",
     status="success"
   )
   HTTP 200 with JSON payload
   
6. FRONTEND - Display
   response received via axios promise
   Chat component adds assistant message
   Parses response into sections
   Renders with formatting
   
7. USER - Sees Result
   Beautiful, professional stock analysis displayed
   Can ask follow-up questions
```

---

## 7. Performance Optimization Techniques

### Lazy Loading

```python
# LLM loaded on first query, not on startup
def _initialize_llm(self):
    if self.llm is not None:
        return  # Already loaded
    
    self.llm = Llama(model_path=self.llm_model)

# Reduces startup time from 2m30s to 2m
```

### Concurrent Processing

```python
# Use ThreadPoolExecutor for timeout protection
from concurrent.futures import ThreadPoolExecutor, TimeoutError

def _generate_response(self, prompt: str) -> str:
    with ThreadPoolExecutor(max_workers=1) as executor:
        future = executor.submit(self._generate_with_llm, prompt)
        try:
            return future.result(timeout=180)
        except TimeoutError:
            return "Timeout error"
```

### Vector Search Limits

```python
# Only retrieve top-k documents
top_k = min(k, config.TOP_K_RESULTS)  # Default: 10
docs = vector_store.similarity_search(query, k=top_k)

# Reduces:
# - Token count in prompt
# - LLM processing time
# - Memory usage
```

### Batch Processing for Embeddings

```python
# Embedding model uses small batch size
embeddings = LlamaCppEmbeddings(
    n_batch=8  # Process 8 chunks at a time
)

# Balances:
# - Memory usage (small batch)
# - Processing speed (not too small)
```

---

## 8. Error Handling Patterns

### Backend Error Handling

```python
try:
    vector_db = initialize_vector_db(doc_path)
except Exception as e:
    # Automatic ChromaDB recovery
    self._reset_chroma_db()
    vector_db = initialize_vector_db(doc_path)

# For queries
try:
    answer = agent.answer_question(question)
except ValueError as e:
    raise HTTPException(status_code=400, detail=str(e))
except TimeoutError as e:
    raise HTTPException(status_code=504, detail="Request timeout")
except Exception as e:
    raise HTTPException(status_code=500, detail=str(e))
```

### Frontend Error Handling

```typescript
try {
  const response = await apiService.queryAgent(question);
  setMessages([...messages, {
    role: 'assistant',
    content: response.answer
  }]);
} catch (error) {
  setMessages([...messages, {
    role: 'assistant',
    content: 'Error processing query',
    isError: true
  }]);
}
```

---

## 9. Configuration Management

### Environment Variables

```env
# .env file
LLM_MODEL_PATH=./models/Meta-Llama-3-8B-Instruct.Q4_K_M.gguf
EMBEDDING_MODEL_PATH=./models/nomic-embed-text-v1.5.Q4_K_M.gguf
CHROMA_DB_PATH=./chroma_db
DOCUMENTS_PATH=./data
CHUNK_SIZE=1400
CHUNK_OVERLAP=400
TOP_K_RESULTS=10
```

### Dynamic Configuration Loading

```python
from dotenv import load_dotenv
import os

load_dotenv()

class Config:
    LLM_MODEL_PATH = os.getenv(
        "LLM_MODEL_PATH",
        "models/Meta-Llama-3-8B-Instruct.Q4_K_M.gguf"
    )
    # ... more variables

config = Config()
```

---

## 10. Testing Scenarios

### Query Types to Test

```python
# Test Case 1: General capital markets question
query: "What is the difference between IPO and FPO?"

# Test Case 2: Stock analysis query
query: "Analyze RELIANCE.NS stock"

# Test Case 3: Multi-ticker query
query: "Compare INFY.NS and TCS.NS"

# Test Case 4: Out-of-domain query (should reject)
query: "What is the weather today?"

# Test Case 5: Invalid ticker
query: "Tell me about INVALID.NS"

# Test Case 6: Complex analysis
query: "Should I invest in ICICIBANK.NS for long-term wealth creation?"
```

### API Endpoint Testing

```bash
# Health check
curl http://localhost:8000/api/status

# Query agent
curl -X POST http://localhost:8000/api/query \
  -H "Content-Type: application/json" \
  -d '{"question": "What is an IPO?"}'

# Search stock
curl "http://localhost:8000/api/stocks/search?query=Infosys"

# Rebuild vector DB
curl -X POST http://localhost:8000/api/rebuild \
  -H "Content-Type: application/json" \
  -d '{"force": true}'
```

---

This technical deep dive provides comprehensive implementation details for understanding and extending the system!

