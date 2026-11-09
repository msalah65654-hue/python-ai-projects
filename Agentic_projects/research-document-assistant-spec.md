# AI Research & Document Assistant: Project Spec

> Status: Draft v1. Items marked **(assumed)** are my suggestions, not decisions. Please review them.

## 1. Overview

A web chat app that lets a team ask questions about a library of research papers (PDFs). It summarizes documents, answers questions with citations, and drafts reports. It runs fully on a local machine using a free, open-source LLM.

## 2. Decisions Made

| Topic | Decision |
|---|---|
| Users | A company or team |
| Sources | PDFs and research papers (public papers only) |
| Languages | Arabic and English (documents and questions) |
| Core tasks | Summarize documents, answer questions with citations, write reports or drafts |
| Interface | Web chat app |
| LLM | Free, open-source, running locally |
| Hardware | Simple to moderate (assume no strong GPU) |
| Scale | Hundreds of documents |
| Citation style | File name + page number |
| Report output | Shown in the chat only (no file export) |

## 3. Goals and Non-Goals

**Goals**
- Answer questions grounded in the uploaded papers, never from guesswork.
- Always show where an answer came from (file name + page).
- Work well in both Arabic and English, including mixed questions.
- Run on a modest machine with no paid APIs.

**Non-Goals (for v1)**
- Exporting reports to Word or PDF.
- Handling confidential or private documents.
- Web search or fetching papers automatically.
- Multi-user accounts and permissions.

## 4. Agentic AI in Plain Words

An "agent" is an LLM that can **decide which tool to use** to finish a task, instead of only answering from its own memory. For this project the tools are plain Python functions.

| Tool | What it does |
|---|---|
| `search_documents(query)` | Finds the most relevant text chunks from the PDF library |
| `summarize_document(file_name)` | Summarizes one paper, section by section |
| `write_report(topic)` | Searches several papers, then drafts a report with citations |

The agent loop: **read the question → choose a tool → read the result → decide if more tools are needed → write the final answer with citations.**

## 5. Architecture

```
PDFs → Parser → Chunker → Embeddings → Vector DB
                                           │
User question → Agent (local LLM) ──calls──┘
                    │
                    └→ Answer + citations → Web chat UI
```

**Components (assumed)**

| Layer | Suggested choice | Why |
|---|---|---|
| Local LLM runner | Ollama | Easiest way to run free models locally |
| LLM | A small quantized model with good Arabic support (e.g. a Qwen 2.5 3B or 7B variant) | Fits modest hardware; check current options when you start |
| Embeddings | A multilingual model (e.g. multilingual-e5-small) | Needed so Arabic and English text match each other |
| PDF parsing | PyMuPDF | Fast, keeps page numbers |
| Vector store | ChromaDB | Simple, local, stores page metadata |
| Agent logic | Plain Python first, then optionally LangChain or LlamaIndex | Plain code teaches how agents really work |
| Web UI | Streamlit or Gradio | Chat interface in a few lines |

## 6. Data Pipeline

1. Upload or drop PDFs into a `papers/` folder.
2. Extract text page by page, keeping `file_name` and `page_number`.
3. Split into chunks (about 500 to 800 tokens, with a small overlap).
4. Create embeddings and store them with the metadata.
5. At question time, retrieve the top matching chunks and give them to the LLM.

**Arabic note:** check that PDF text extraction keeps Arabic in the right order. Scanned PDFs would need OCR (out of scope for v1).

## 7. Behavior Rules

- Answer only from retrieved text. If nothing relevant is found, say so.
- Every factual claim ends with a citation like `(paper_name.pdf, p. 7)`.
- Reply in the language of the question.
- Reports are written in the chat, with a short outline first, then the body.

## 8. Build Phases

1. **Phase 1, Setup:** install Ollama, pull a model, confirm it answers in Arabic and English.
2. **Phase 2, Retrieval:** parse PDFs, build the vector DB, test search from the command line.
3. **Phase 3, Q&A with citations:** answer questions from retrieved chunks and show file + page.
4. **Phase 4, Agent tools:** add `summarize_document` and `write_report`, and let the LLM choose tools.
5. **Phase 5, Web UI:** wrap everything in a chat interface.
6. **Phase 6, Evaluate:** test with 20 or more real questions and check citation accuracy.

## 9. Success Criteria

- Citations point to the correct page in at least 90% of test answers.
- Arabic and English questions both get useful answers.
- Typical answer time is acceptable on your machine (target: under about 30 seconds).
- No answers about things not in the papers.

## 10. Risks

| Risk | Mitigation |
|---|---|
| Small local models hallucinate or ignore Arabic nuance | Strict "answer only from sources" prompt, test several models |
| Slow responses on CPU | Smaller or more quantized model, fewer retrieved chunks |
| Poor PDF text (columns, formulas, tables) | Test parsers early, flag bad files |
| Weak cross-language retrieval | Use a multilingual embedding model and test Arabic queries over English papers |

## 11. Open Questions

- Exact hardware: RAM size and whether there is any GPU? (This decides the model size.)
- Roughly how many users at the same time?
- Should the chat remember earlier messages in a session?
- Should users upload PDFs from the web UI, or will an admin manage the folder?
- What are 5 to 10 example questions your team would really ask? (These become the test set.)
