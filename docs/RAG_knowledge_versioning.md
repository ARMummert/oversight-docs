# OverSight ICS: RAG Knowledge Versioning Policy

**Scope:** Retrieval-Augmented Generation (RAG) and Manual Indexing

## 1. Versioning Strategy
To prevent "RAG Collisions" and "Update Collisions," where the AI references an outdated manual during an active 
incident, all grounded documentation must follow a strict versioning protocol.

### 1.1 Semantic Index Versioning
- **Manual IDs**: Every asset manual (PDF/Markdown) is assigned a unique `Knowledge-ID` and `Semantic-Hash`.
- **Sync Requirement**: The RAG vector database must match the current `Software_Revision` of the Field Asset.
- **Update Protocol**: When a manual is updated, the old index is not deleted. It is archived as `N-1`. The AI will 
  reference the manual version that was active at the time the incident was first detected.

## 2. Data Poisoning Protection
- **Instruction Injection Defense**: All documents entering the RAG pipeline are scanned for "Adversarial 
  Instruction Injection" (e.g., text hidden in white font designed to override AI safety constraints) [cite: 3.1].
- **Verification**: Only manuals signed by the **Safety Lead** or **Lead Software Engineer** are permitted in the `/api/v1/rag/context` retrieval path [cite: 3.1].
