# Backend Architecture: Contextual Memory System

## Overview

This document outlines the backend-focused architecture for our contextual memory system, with special emphasis on where LangChain and BAML components integrate into the system.

## Architecture Decision

After evaluating architectural options, we've chosen a backend-heavy approach for implementing the contextual memory system. This approach provides several advantages for our memory-intensive application:

- **Security**: Protects LLM API keys and vector database credentials
- **Performance**: Offloads computationally intensive operations from the client
- **Reliability**: Ensures robust database operations and consistent state management
- **Scalability**: Allows independent scaling of memory processing components
- **Background Processing**: Facilitates the "sleep-like" consolidation process

## System Architecture

```
┌─────────────────┐                  ┌────────────────────────────────────────────┐
│                 │                  │                                            │
│   Frontend      │                  │   Backend                                  │
│   (React)       │                  │                                            │
│                 │                  │   ┌─────────────┐                          │
│  ┌───────────┐  │   REST/GraphQL   │   │ API Layer   │                          │
│  │ LLMChat   │◄─┼──────────────────┼──►│             │                          │
│  │ UI        │  │                  │   └─────┬───────┘                          │
│  └───────────┘  │                  │         │                                  │
│                 │                  │   ┌─────▼───────┐                          │
│  ┌───────────┐  │                  │   │ Memory      │                          │
│  │ Memory    │  │                  │   │ Service     │      ┌─────────────┐     │
│  │ UI        │◄─┼──────────────────┼──►│ (LangChain) │◄────►│  LanceDB    │     │
│  └───────────┘  │                  │   └─────┬───────┘      └─────────────┘     │
│                 │                  │         │                                  │
└─────────────────┘                  │   ┌─────▼───────┐                          │
                                     │   │ LLM Service │                          │
                                     │   │ (BAML)      │                          │
                                     │   └─────────────┘                          │
                                     │                                            │
                                     │   ┌─────────────────────┐                  │
                                     │   │ Consolidation       │                  │
                                     │   │ Service             │                  │
                                     │   │ (LangChain + BAML)  │                  │
                                     │   └─────────────────────┘                  │
                                     │                                            │
                                     └────────────────────────────────────────────┘
```

## Backend Components

### 1. API Layer
**Purpose**: Provides interface for frontend communication and routes requests to appropriate services

**Implementation**:
- REST or GraphQL API endpoints for all memory and chat functions
- Authentication and authorization middleware
- Request validation and error handling

**Technologies**:
- Express.js or FastAPI (Python)
- GraphQL (optional for complex data requirements)

### 2. Memory Service (LangChain Integration)
**Purpose**: Manages memory storage, retrieval, and formation

**LangChain Components Used**:
- `VectorStoreRetriever` for memory retrieval
- `ContextualCompressionRetriever` for relevance filtering
- `Embeddings` for converting memory notes to vector representations

**Key Functionalities**:
```python
# Memory formation with LangChain
from langchain.chains import LLMChain
from langchain.prompts import PromptTemplate
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import LLMChainExtractor

class MemoryService:
    def __init__(self, vector_store, llm_service):
        self.vector_store = vector_store
        self.llm_service = llm_service
        
        # Create base retriever with LangChain
        self.base_retriever = vector_store.as_retriever(
            search_type="similarity_score_threshold",
            search_kwargs={
                "k": 10,
                "score_threshold": 0.7,
                "filter": self._active_memory_filter
            }
        )
        
        # Create contextual compression retriever
        compressor = LLMChainExtractor.from_llm(llm_service.get_llm())
        self.retriever = ContextualCompressionRetriever(
            base_retriever=self.base_retriever,
            base_compressor=compressor
        )
    
    def retrieve_memories(self, context):
        """Retrieve relevant memories using LangChain's retrieval components"""
        return self.retriever.get_relevant_documents(context)
    
    def store_memory(self, memory_note):
        """Store formed memory in vector database with metadata"""
        # Implementation details...
```

### 3. LLM Service (BAML Integration)
**Purpose**: Manages LLM interactions with structured prompts

**BAML Components Used**:
- Prompt schemas for memory formation
- Type validation for memory structure
- Versioned prompts for consistent memory generation

**Key Functionalities**:
```python
# LLM Service with BAML integration
from baml.llm import BamlLLM
from langchain_community.llms import Anthropic

class LLMService:
    def __init__(self):
        # Initialize LLM
        self.llm = Anthropic(model="claude-3.7-sonnet-20250219")
        
        # Create BAML integration
        self.baml_llm = BamlLLM(llm=self.llm)
        
        # Load BAML prompts
        self.memory_reflection = self.baml_llm.get_prompt("MemoryReflection")
        self.memory_consolidation = self.baml_llm.get_prompt("MemoryConsolidation")
    
    def form_memory(self, conversation, previous_notes=None):
        """Use BAML-defined prompt to form structured memory note"""
        input_data = {
            "conversation": conversation,
            "previous_notes": previous_notes or []
        }
        
        # BAML handles the prompt structure and output validation
        return self.memory_reflection(input_data)
    
    def get_llm(self):
        """Return LLM instance for other services to use"""
        return self.llm
```

### 4. Consolidation Service (LangChain + BAML)
**Purpose**: Implements "sleep-like" processing of memories

**LangChain + BAML Components Used**:
- BAML for structured consolidation prompts and output schemas
- LangChain for memory batch processing and vector operations
- LangChain schedulers for periodic consolidation

**Key Functionalities**:
```python
# Consolidation Service using both LangChain and BAML
import schedule
import time
from datetime import datetime

class ConsolidationService:
    def __init__(self, memory_service, llm_service):
        self.memory_service = memory_service
        self.llm_service = llm_service
    
    def schedule_consolidation(self, time="03:00"):
        """Schedule daily consolidation using LangChain's scheduler"""
        schedule.every().day.at(time).do(self.run_consolidation)
        
        # Start scheduler in background thread
        # Implementation details...
    
    def run_consolidation(self):
        """Run memory consolidation process"""
        print(f"Running memory consolidation at {datetime.now()}")
        
        # Retrieve memories using LangChain
        recent_memories = self.memory_service.retrieve_recent_memories(24)
        older_memories = self.memory_service.retrieve_random_older_memories(20)
        
        # Process using BAML prompt
        input_data = {
            "recent_memories": recent_memories,
            "random_older_memories": older_memories
        }
        
        # BAML ensures properly structured output
        consolidation_result = self.llm_service.memory_consolidation(input_data)
        
        # Process results and update vector database
        self._process_consolidation_results(consolidation_result)
        
        # Generate report
        self._generate_consolidation_report(consolidation_result)
```

### 5. LanceDB Integration
**Purpose**: Vector database for storing and querying memory embeddings

**Implementation**:
- Custom adapter for LanceDB to work with LangChain's vector store interface
- Optimized storage and retrieval of memory embeddings
- Metadata filtering for memory queries

```python
# LanceDB integration
import lancedb
import numpy as np
from langchain.vectorstores.base import VectorStore

class LanceDBStore(VectorStore):
    """Custom LangChain vectorstore for LanceDB"""
    
    def __init__(self, connection_string, table_name, embedding_function):
        self.db = lancedb.connect(connection_string)
        self.table_name = table_name
        self.embedding_function = embedding_function
        
        # Create or access table
        if table_name in self.db.table_names():
            self.table = self.db.open_table(table_name)
        else:
            # Create schema and table
            # Implementation details...
    
    def add_texts(self, texts, metadatas=None, **kwargs):
        """Add text and embeddings to LanceDB"""
        # Implementation details...
    
    def similarity_search(self, query, k=4, **kwargs):
        """Search for similar vectors in LanceDB"""
        # Implementation details...
```

## API Design

The backend exposes a RESTful API for the frontend to interact with:

```typescript
// Core API endpoints
interface MemorySystemAPI {
  // Chat endpoints
  POST /api/chat/message
    Request: { message: string, conversation_id: string }
    Response: { 
      reply: string, 
      memories_used: Memory[],
      memory_formed: Memory | null
    }
  
  // Memory endpoints
  GET /api/memories
    Query: { query?: string, filters?: MemoryFilter, limit?: number }
    Response: { memories: Memory[] }
  
  GET /api/memories/:id
    Response: { memory: Memory }
  
  PATCH /api/memories/:id
    Request: { confidence?: number, is_correct?: boolean }
    Response: { updated: boolean }
  
  // Consolidation endpoints
  GET /api/consolidation/reports
    Query: { limit?: number, offset?: number }
    Response: { reports: ConsolidationReport[] }
  
  GET /api/consolidation/network
    Response: { nodes: MemoryNode[], edges: MemoryEdge[] }
  
  POST /api/consolidation/trigger
    Response: { scheduled: boolean, estimated_completion: string }
}
```

## LangChain and BAML Integration Points

### LangChain Integration Points

1. **Vector Operations**:
   - Vector storage and retrieval using LangChain's vectorstore abstractions
   - Adapting LanceDB to work with LangChain's interfaces

2. **Retrieval Mechanisms**:
   - Using LangChain's retrievers for semantic and metadata-based filtering
   - Contextual compression for more relevant memory retrieval

3. **Chat Chains**:
   - Integrating retrieved memories into conversation context
   - Managing conversation state and history

4. **Embedding Generation**:
   - Using LangChain's embedding interfaces for consistent vector creation
   - Batched processing of embeddings for efficiency

### BAML Integration Points

1. **Memory Formation**:
   - Structured prompts for consistent memory note creation
   - Schema validation of memory outputs

2. **Consolidation Prompts**:
   - Complex prompt engineering for memory consolidation
   - Type-safe handling of consolidation results

3. **Versioning and Iteration**:
   - Tracking changes to prompt design over time
   - A/B testing different memory formation approaches

## Implementation Phases

### Phase 1: Core Backend Infrastructure
- Set up API layer with basic endpoints
- Implement LanceDB integration with LangChain
- Create basic memory formation with BAML

### Phase 2: Memory Services
- Implement full memory retrieval with contextual compression
- Add confidence scoring and evidence tracking
- Develop memory formation pipeline

### Phase 3: Consolidation System
- Implement scheduled consolidation
- Develop BAML prompts for consolidation operations
- Create consolidation reporting system

### Phase 4: Advanced Features
- Implement memory correction and feedback
- Add memory network visualization API
- Develop cross-conversation memory insights

## Technical Considerations

### 1. Performance Optimization
- Efficient vector operations for large memory sets
- Caching strategies for frequently accessed memories
- Asynchronous processing for memory formation

### 2. Scalability
- Horizontal scaling of API and processing components
- Database partitioning for growing memory collections
- Queue-based architecture for high-volume operations

### 3. Security
- Secure storage of API credentials and database connections
- User isolation for multi-tenant deployments
- Data encryption for sensitive memory content

### 4. Testing Strategy
- Unit tests for individual services
- Integration tests for LangChain and BAML components
- End-to-end tests for complete memory workflows

### 5. Monitoring and Observability
- Logging of memory operations and LLM interactions
- Performance metrics for vector operations
- Quality assessment of formed memories

## Conclusion

This backend-heavy architecture leverages LangChain and BAML to create a robust contextual memory system. LangChain provides the retrieval and vector operations components, while BAML ensures consistent, structured memory formation and consolidation. The resulting system should deliver reliable memory capabilities while maintaining security, performance, and scalability. 