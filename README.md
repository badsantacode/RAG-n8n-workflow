# RAG-n8n: Hybrid Document Retrieval & RAG n8n Workflow with Qdrant, Azure DevOps, and Reranking

[![n8n](https://img.shields.io/badge/n8n-0.237.0-%23EA4B2A)](https://n8n.io/)
[![Qdrant](https://img.shields.io/badge/Qdrant-1.9.0-%234B32C4)](https://qdrant.tech/)
[![LangChain](https://img.shields.io/badge/LangChain-0.2.0-%231C3C3C)](https://www.langchain.com/)
[![Ollama](https://img.shields.io/badge/Ollama-Qwen3-%23000000)](https://ollama.ai/)

## Overview

This repository contains a production-ready **Retrieval-Augmented Generation (RAG)** workflow built on **n8n**. It implements a hybrid search strategy combining dense and sparse vectors in **Qdrant**, with a **cross-encoder reranker** to improve retrieval precision. The system is designed for enterprise knowledge bases and is fully integrated with **Azure DevOps (Wiki)** as the document source.

**Key Use Cases:**
- Enterprise knowledge base chatbots
- Technical support automation
- Internal documentation Q&A
- Hybrid search over large document collections

---

## Architecture & Workflow Process

The workflow is divided into two main pipelines that run in parallel:

### 1. Document Indexing Pipeline (Scheduled)
Triggered by a **Schedule Trigger**, this pipeline keeps the vector database in sync with your Azure DevOps Wiki.

1. **Fetch Document Lists** – Two branches run in parallel:
   - **Qdrant**: Retrieves all existing document metadata (paths and modification dates) using the Qdrant node.
   - **Azure DevOps**: Fetches the current file list from an Azure DevOps Wiki repository via HTTP Request.
2. **Prepare & Compare Metadata** – The two lists are normalized and compared using the **Compare Datasets** node. This identifies:
   - **New/Updated files** (to be (re)indexed)
   - **Deleted files** (to be removed from Qdrant)
3. **Delete Old Vectors** – Using an HTTP Request node, the workflow deletes points in Qdrant that match the `metadata.path` of removed files.
4. **Loop Through Changed Files** – For each new or updated file:
   - Fetches the raw content from Azure DevOps.
   - Uses **Data preparation** to clean the content (e.g., removes Markdown image tags).
   - Splits the text into chunks using a **Recursive Character Text Splitter** (chunk size: 2000, overlap: 200).
   - Generates embeddings using a local **Ollama** instance with the `qwen3-embedding:0.6b` model.
   - Stores both dense and sparse vectors in Qdrant (collection: `RagDocuments`).

### 2. Query Pipeline (User-Facing Chat)
This pipeline is triggered by user input through a **Chat Trigger** (webhook).

1. **Hybrid Search** – The user query is sent to the **Qdrant Hybrid Vector Store** node, which:
   - Uses the same `qwen3-embedding` model to generate query vectors.
   - Performs both dense (semantic) and sparse (keyword) retrieval in parallel.
   - Returns the top 20 candidate documents.
2. **Reranking with Cross-Encoder** – The **Use Reranker** node sends the top candidates to an external reranker service (running `BAAI/bge-reranker-v2-m3` at `http://10.1.222.23:8003/rerank`). This step:
   - Re-scores documents based on relevance to the query.
   - Applies a **relevance threshold (0.5)** – documents below this score are flagged as `NO_RELEVANT_INFO`.
   - Returns only the single most relevant document.
3. **Context Assembly** – The **Get n results** node formats the retrieved document and its metadata into a clean text block.
4. **Prompt Engineering** – The **Prompt** node constructs a system message that:
   - Instructs the LLM to only use the provided context.
   - Explicitly forbids adding external URLs or hallucinated paths.
   - Mandates a final "Sources" block with the exact file path from the retrieved document.
5. **LLM Generation** – The prompt and context are sent to the **AI Agent** node, which uses a local **Ollama** model (`qwen3.8:27b`) for answer generation. The agent also has access to **Postgres Chat Memory** to maintain conversation context.

---

## Key Features

- **Hybrid Search**: Combines dense embeddings (semantic) and sparse vectors (keyword) for balanced retrieval.
- **Cross-Encoder Reranking**: Improves precision by re-scoring the top candidates with a BGE reranker.
- **Relevance Thresholding**: Filters out irrelevant documents to prevent hallucination.
- **Automated Document Sync**: Keeps the vector database up-to-date with your Azure DevOps Wiki.
- **Conversation Memory**: Maintains chat history using PostgreSQL (via `Postgres Chat Memory` node).
- **Local LLM & Embeddings**: Uses Ollama for both embedding and generation, ensuring data privacy.
- **Enterprise Ready**: Designed for secure internal knowledge bases with Azure DevOps authentication.

---

## Technology Stack

| Component | Technology | Purpose |
|-----------|------------|---------|
| Workflow Engine | **n8n** | Orchestrate the pipelines |
| Vector Database | **Qdrant** | Store dense + sparse vectors |
| Embeddings | **Ollama** (Qwen3-embedding) | Generate vector representations |
| Reranker | **BGE-reranker-v2-m3** | Cross-encoder for relevance scoring |
| LLM | **Ollama** (Qwen3:27b) | Generate final answers |
| Document Source | **Azure DevOps Wiki** | Store raw documentation files |
| Chat Memory | **PostgreSQL** | Store conversation history |

---

## Getting Started

### Prerequisites
- n8n (self-hosted or cloud)
- Qdrant (running locally or cloud)
- Ollama with `qwen3-embedding:0.6b` and `qwen3.8:27b` models
- Reranker service (or you can adapt the HTTP endpoint)
- PostgreSQL database

### Installation

1. **Clone the repository:**
   ```bash
   git clone https://github.com/badsantacode/RAG-n8n-workflow.git
   cd RAG-n8n-workflow
   ```

2. **Import the workflow**  
   Import the file `RAG_qdrant_azuredevops_hybrid.json` into your n8n instance using the n8n UI (Workflows → Import from File).

3. **Configure credentials**  
   The workflow uses several credential types. You'll need to set up the following in n8n's credential manager:

   | Credential Name | Type | Purpose |
   |-----------------|------|---------|
   | `DocumentVector` | PostgreSQL | Chat memory storage |
   | `Qdrant account prod-mod` | Qdrant REST API | Read access to vector DB |
   | `Unnamed credential` (HTTP Basic Auth) | HTTP Basic Auth | Azure DevOps access (username + personal access token) |
   | `Qdrant account prod` | Qdrant REST API | Write access to vector DB |
   | `Ollama account prod` | Ollama API | Local LLM and embeddings |
   | `Qdrant account prod` (second instance) | Qdrant REST API | Hybrid search operations |

4. **Update environment-specific URLs**  
   Edit the following nodes to match your infrastructure:
   - `Reranker service URL`: In the "Use Reranker" node (currently `http://10.1.1.1:8003/rerank`)
   - `Azure DevOps URL`: In both "Azure get list" and "Azure get file" nodes (currently `https://azure.company.com/...`)
   - `Qdrant URLs`: In all Qdrant nodes (currently `http://10.1.1.1:6333`)

5. **Activate the workflow**  
   Toggle the workflow to **Active** in the n8n UI. The scheduled indexer will run according to the **Schedule Trigger** settings, and the chat endpoint will be available immediately.

### Usage

- **Access the chat interface**  
  The Chat Trigger node provides a webhook URL. Open it in a browser to start interacting with the RAG system.

- **Monitor indexing**  
  The "Schedule Trigger" node runs the indexing pipeline. Check the execution history to verify documents are being processed.

- **Test the system**  
  Ask questions related to your indexed Azure DevOps Wiki content. The system will return answers with source attribution.

---

## Customization

- **Relevance Threshold**: Adjust `RELEVANCE_THRESHOLD = 0.5` in the "Use Reranker" Code node to make results more (lower) or less (higher) conservative.
- **Chunk Size**: Modify the `chunkSize` parameter in the "Recursive Character Text Splitter" node.
- **LLM Model**: Change the model name in the "LLM model" node to use any Ollama-supported model.
- **Reranker Endpoint**: Replace the URL in the "Use Reranker" node if you host your own reranker service.

---

## Keywords

`RAG`, `n8n`, `Qdrant`, `hybrid search`, `dense vector`, `sparse vector`, `cross-encoder`, `reranker`, `Azure DevOps`, `wiki`, `document retrieval`, `chatbot`, `LLM`, `Ollama`, `Qwen`, `embedding`, `knowledge base`, `enterprise search`, `chat memory`, `PostgreSQL`, `automation`, `LangChain`.

---

## Contributing

Contributions are welcome! Please open an issue or pull request for any improvements.

---

## License

This project is licensed under the MIT License – see the `LICENSE` file for details.

---

## Contact & Support

For questions or support, please open an issue on GitHub or contact the repository owner.
