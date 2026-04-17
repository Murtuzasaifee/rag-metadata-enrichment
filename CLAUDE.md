# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Common Commands

### Development
```bash
# Start Qdrant (required for vector storage)
docker-compose up -d

# Install dependencies (use uv for speed)
pip install -r requirements.txt

# Run the FastAPI application
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# Run notebooks
jupyter notebook notebooks/
```

### Environment Setup
```bash
cp .env.example .env
# Edit .env to add OPENAI_API_KEY
```

## Architecture

### Processing Pipeline (Upload Flow)
1. **DocumentProcessor** (`services/document_processor.py`): Uses Docling HybridChunker to convert PDF/MD/TXT/JSON files into semantic chunks. The chunker uses `sentence-transformers/all-MiniLM-L6-v2` tokenizer for intelligent boundaries.

2. **MetadataExtractor** (`services/metadata_extractor.py`): GLiNER2 zero-shot extraction per chunk. Schema defined in `_create_schema()` extracts:
   - Entities: technology, company, product, concept, metric, person, location, date
   - Domain: database, cloud_computing, machine_learning, backend_development, etc.
   - Content Type: tutorial, api_documentation, architecture_guide, etc.
   - Tech Specs: mentioned_products, versions, requirements, capabilities

3. **VectorStore** (`services/vector_store.py`): Qdrant client with hybrid search support:
   - Dense embeddings: OpenAI `text-embedding-3-small` (1536 dim)
   - Sparse embeddings: FastEmbed `Qdrant/bm25` for BM25 keyword matching
   - Collection created with both vector configs on first run

4. **RetrievalService** (`services/retrieval.py`): Orchestrates retrieval and answer generation using GPT-4o-mini.

### Query Flow
`POST /query` → RetrievalService.retrieve() (VectorStore.hybrid_search with metadata filters) → RetrievalService.generate_answer() (OpenAI chat completion)

### Key Data Models (`app/models.py`)
- `ChunkMetadata`: Core schema stored in Qdrant payloads. Contains entities dict, domain, content_type, tech_specs, plus structural fields (source_file, file_type, chunk_index, page_number)
- `QueryRequest`: Accepts `hybrid_alpha` (0-1, 0=bm25-only, 1=dense-only) and optional filters: `domain_filter`, `content_type_filter`, `entity_filter`

### Configuration
All settings via `app/config.py` with pydantic-settings from `.env`:
- GLiNER model: defaults to `fastino/gliner2-large-v1` (~500MB download on first run)
- Chunking: 512 tokens, 50 overlap
- Hybrid alpha: 0.5 (balanced dense/sparse)

### Notebooks
- `metadata_enrichment_tutorial.ipynb`: End-to-end RAG baseline vs enriched comparison. Creates `baseline_rag` and `enriched_rag` collections.
- `gliner2_complete_features.ipynb`: GLiNER2 standalone showcase (no API keys needed)

## Development Notes

- Docling HybridChunker requires the `sentence-transformers/all-MiniLM-L6-v2` model, which downloads automatically.
- GLiNER2 schema is built once at initialization and cached for all extractions.
- Qdrant collection creation is idempotent — will reuse existing collection.
- Sparse BM25 embeddings are computed via FastEmbed `Qdrant/bm25` model.
- When modifying GLiNER schema (labels, domains, content types), update `_create_schema()` in MetadataExtractor.
