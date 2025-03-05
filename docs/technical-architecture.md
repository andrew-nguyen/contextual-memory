# Contextual Memory System: Technical Architecture

## System Components

### 1. Memory Formation System
- **Technology**: BAML + LangChain
- **Purpose**: Creates structured memory notes from conversations

#### BAML Prompt Definitions
```yaml
# memory_reflection.baml
prompt MemoryReflection<MemoryNote> {
  input {
    conversation: String
    previous_notes: Array<MemoryNote>?
  }

  schema MemoryNote {
    facts: Array<Fact>
    inferences: Array<Inference>
    preferences: Array<Preference>
    emotional_context: String
    potential_topics: Array<String>
    connections: Array<Connection>
    knowledge_gaps: Array<String>
  }

  schema Fact {
    statement: String
    source_turn: Int
  }

  schema Inference {
    statement: String
    confidence: Int
    evidence: Array<String>
    alternatives: Array<String>
    invalidation_conditions: String
  }

  schema Preference {
    description: String
    evidence: Array<String>
    confidence: Int
  }

  schema Connection {
    topic: String
    related_note_ids: Array<String>
    relationship_type: String
  }

  # Prompt template defined here...
}
```

#### LangChain Implementation
```python
from langchain.chains import LLMChain
from langchain.prompts import PromptTemplate
from langchain_community.llms import Anthropic
from baml.llm import BamlLLM

# Initialize LLM
llm = Anthropic(model="claude-3.7-sonnet-20250219")

# Create BAML integration
baml_llm = BamlLLM(llm=llm)

# Memory reflection chain
memory_reflection = baml_llm.get_prompt("MemoryReflection")
```

### 2. Memory Storage System
- **Technology**: LangChain + Vector Database (Pinecone/Chroma)
- **Purpose**: Indexes and stores memory notes for efficient retrieval

```python
from langchain.vectorstores import Chroma
from langchain.embeddings import OpenAIEmbeddings

# Initialize embedding model
embeddings = OpenAIEmbeddings()

# Initialize vector store
vector_store = Chroma(
    embedding_function=embeddings,
    collection_name="memory_notes",
    persist_directory="./memory_storage"
)

# Store memory note
def store_memory(memory_note):
    # Convert structured note to text for embedding
    text_representation = json.dumps(memory_note)
    
    # Store with metadata for filtering
    metadata = {
        "confidence_avg": calculate_avg_confidence(memory_note),
        "creation_time": time.time(),
        "last_accessed": time.time(),
        "access_count": 0,
        "topics": memory_note["potential_topics"],
        # Additional metadata for filtering
    }
    
    vector_store.add_texts(
        texts=[text_representation],
        metadatas=[metadata]
    )
```

### 3. Memory Retrieval System
- **Technology**: LangChain
- **Purpose**: Retrieves relevant memories based on conversation context

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import LLMChainExtractor

# Create base retriever
retriever = vector_store.as_retriever(
    search_type="similarity_score_threshold",
    search_kwargs={
        "k": 10,
        "score_threshold": 0.7,
        "filter": lambda x: x["last_accessed"] > (time.time() - 30*24*60*60)  # Active in last 30 days
    }
)

# Create compressor for contextual relevance
compressor = LLMChainExtractor.from_llm(llm)

# Create contextual compression retriever
contextual_retriever = ContextualCompressionRetriever(
    base_retriever=retriever,
    base_compressor=compressor
)

# Retrieve relevant memories
def retrieve_memories(query):
    return contextual_retriever.get_relevant_documents(query)
```

### 4. Consolidation System
- **Technology**: BAML + LangChain + Scheduling
- **Purpose**: Implements sleep-like processing of memories

#### BAML Prompt Definitions
```yaml
# memory_consolidation.baml
prompt MemoryConsolidation<ConsolidationResult> {
  input {
    recent_memories: Array<MemoryNote>
    random_older_memories: Array<MemoryNote>
  }

  schema ConsolidationResult {
    meta_connections: Array<MetaConnection>
    contradictions: Array<Contradiction>
    pruning_recommendations: Array<PruningRecommendation>
    new_memory_notes: Array<MemoryNote>
    confidence_updates: Array<ConfidenceUpdate>
  }

  schema MetaConnection {
    description: String
    memory_ids: Array<String>
    confidence: Int
  }

  # Additional schemas defined here...
}
```

#### Scheduling Implementation
```python
import schedule
import time
from datetime import datetime

def run_consolidation():
    print(f"Running memory consolidation at {datetime.now()}")
    
    # Retrieve recent memories (last 24 hours)
    recent_memories = retrieve_recent_memories(24)
    
    # Retrieve random sample of older memories
    older_memories = retrieve_random_older_memories(20)
    
    # Run consolidation
    consolidation_result = run_baml_consolidation(recent_memories, older_memories)
    
    # Process results
    process_consolidation_results(consolidation_result)
    
    # Generate report
    generate_consolidation_report(consolidation_result)

# Schedule daily consolidation
schedule.every().day.at("03:00").do(run_consolidation)  # Run at 3 AM

while True:
    schedule.run_pending()
    time.sleep(60)
```

### 5. Chat Interaction System
- **Technology**: LangChain + BAML
- **Purpose**: Integrates retrieved memories into conversations

```python
from langchain.chat_models import ChatAnthropic
from langchain.schema import HumanMessage, AIMessage, SystemMessage

# Initialize chat model
chat_model = ChatAnthropic(model="claude-3.7-sonnet-20250219")

# Create chat memory handler
def handle_chat_interaction(user_input, chat_history):
    # Step 1: Retrieve relevant memories
    relevant_memories = retrieve_memories(user_input + "\n" + "\n".join(chat_history[-5:]))
    
    # Step 2: Format memories as context
    memory_context = format_memories_as_context(relevant_memories)
    
    # Step 3: Create enhanced system prompt
    system_prompt = f"""You are Claude, an AI assistant with contextual memory.
    
    RELEVANT MEMORIES FROM PRIOR CONVERSATIONS:
    {memory_context}
    
    Use these memories when relevant but don't explicitly mention having memories unless directly asked.
    Respond naturally to the human's input."""
    
    # Step 4: Generate response
    messages = [
        SystemMessage(content=system_prompt),
        *[msg for msg in chat_history],
        HumanMessage(content=user_input)
    ]
    
    response = chat_model.generate(messages)
    
    # Step 5: After response, create new memory asynchronously
    create_memory_async(user_input, response.content, chat_history)
    
    return response.content
```

## Data Flow Diagram

```
┌───────────────────┐      ┌───────────────────┐
│                   │      │                   │
│  User Interaction │◄────►│  Chat System      │
│                   │      │  (LangChain)      │
└───────────────────┘      └─────────┬─────────┘
                                    │
                                    ▼
┌───────────────────┐      ┌───────────────────┐
│                   │      │                   │
│  Memory Retrieval │◄────►│  Memory Formation │
│  (LangChain)      │      │  (BAML+LangChain) │
│                   │      │                   │
└─────────┬─────────┘      └─────────┬─────────┘
          │                          │
          ▼                          ▼
┌───────────────────────────────────────────────┐
│                                               │
│         Vector Database Storage               │
│         (Pinecone/Chroma via LangChain)       │
│                                               │
└─────────────────────┬─────────────────────────┘
                      │
                      ▼
┌───────────────────────────────────────────────┐
│                                               │
│         Consolidation System                  │
│         (BAML+LangChain+Scheduler)            │
│                                               │
└───────────────────────────────────────────────┘
```

## Implementation Phases

### Phase 1: Core Memory System
- Implement basic memory formation with BAML
- Set up vector storage and simple retrieval
- Basic chat integration

### Phase 2: Enhanced Retrieval & Memory Quality
- Implement contextual compression retrieval
- Add confidence scoring and evidence tracking
- Implement memory metadata for better filtering

### Phase 3: Consolidation System
- Implement scheduled consolidation
- Pattern detection across memories
- Contradiction resolution
- Pruning mechanisms

### Phase 4: Reports & Monitoring
- Implement consolidation reports
- Create dashboards for memory system health
- Continuous improvement mechanisms

## Technical Challenges & Considerations

### 1. Embedding Selection
The choice of embedding model significantly impacts memory retrieval quality. OpenAI's text-embedding-3-large currently offers the best semantic representation for this use case.

### 2. Prompt Engineering
BAML will help with prompt versioning, but careful iteration is needed for both memory formation and consolidation prompts.

### 3. Rate Limiting
LLM API rate limits may impact the throughput of the system, especially during consolidation. A queuing system might be necessary.

### 4. Scalability
As the memory store grows, retrieval performance may degrade. Consider:
- Time-based partitioning
- Hierarchical memory organization
- Periodic reembedding as embedding technologies improve

### 5. Error Handling
Implement robust error handling for LLM failures, especially during consolidation, to prevent memory corruption.

### 6. Memory Persistence
Ensure vector database backups and state management to prevent memory loss.

### 7. Privacy & Data Governance
Implement strict access controls and retention policies for stored memories.