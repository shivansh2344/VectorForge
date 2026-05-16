# VectorForge — by SHIVANSH GARG

## Simple summary (for non-technical readers)
- VectorForge is a small local app that helps you find and use meaning in text and small data. It turns text into numbers (called embeddings), stores those numbers so similar texts live near each other, and provides fast search and question-answering over those texts.
- You can use the web UI at http://localhost:8080 to add documents, search, and ask questions. Everything runs locally on your PC.

## What this project does?
- Converts text into numeric vectors (embeddings).
- Stores those vectors in efficient indexes so you can find similar items quickly.
- For documents, it builds an index and can answer natural-language questions by retrieving relevant passages and running a small local generation model to produce human-readable answers.

## High-level architecture 
- Components required (all required for full functionality):
  - `vectorforge.exe` (the C++ server built from `main.cpp`): serves the web UI and the REST API.
  - Ollama (local model server): provides embedding and generation models. Document ingestion and Q&A depend on these models being installed and available locally.
  - Browser UI (`index.html`) to interact with the server.

- Data flow:
  1. You paste or upload text into the UI.
  2. The app asks Ollama to convert that text into numbers (embeddings).
  3. Those numbers are inserted into an index that groups similar texts together.
  4. When you ask a question, the app looks up the most relevant chunks, builds a prompt, and asks Ollama's generation model to produce a concise answer.

## Technical architecture:
- Index types included:
  - BruteForce: exact but slow for many items
  - KD-Tree: fast for low-dimensional vectors
  - HNSW: approximate, high-performance graph-based index for real-world workloads
- Concurrency: in-memory stores are thread-safe via mutexes.
- External dependency: Ollama (runs locally) provides models for both embeddings and generation.

## Why Ollama is required?
- Document embedding and Q&A rely on local models. To have working `doc/insert` and `doc/ask` endpoints, you must install Ollama and pull the embedding and generation models listed below. The demo vector search and basic UI (viewing demo vectors) work without Ollama, but any document ingestion or natural-language answers will fail without it.

### Required models (exact):
- Embedding model: `nomic-embed-text` (required for `doc/insert`)
- Generation model: `llama3.2:3b` (required for `doc/ask`)

### Low-level technical deep-dive
- For a detailed mathematical and algorithmic explanation (HNSW internals, index complexity, embedding math, chunking heuristics, tuning knobs, and future work), see the low-level goldmine: `README-LOWLEVEL.md` in this repository.

## Quickstart (Windows, copy-paste commands)

Open PowerShell as Administrator and run these commands exactly.

1) Install MSYS2 (for a modern g++ on Windows):

```powershell
winget install --id MSYS2.MSYS2 -e --accept-package-agreements --accept-source-agreements
```

Open the "MSYS2 MinGW 64-bit" shell and run:

```bash
pacman -Syu --noconfirm
# If it asks to close and reopen, do so, then:
pacman -Su --noconfirm
pacman -S --noconfirm mingw-w64-x86_64-toolchain base-devel mingw-w64-x86_64-pkg-config
```

Notes: use the MinGW64 shell so you get a 64-bit g++ that supports std::thread and modern C++.

2) Build `vectorforge.exe` (from the project folder):

In the MSYS2 `mingw64.exe` shell:

```bash
cd /c/Users/Asus/Downloads/Your-OWN-AI-main/Your-OWN-AI-main
/mingw64/bin/g++ -std=c++17 -O2 -D_WIN32_WINNT=0x0A00 main.cpp -o vectorforge.exe -lws2_32 -lcrypt32 -pthread
```

Or from PowerShell (make sure MSYS2 mingw64 bin is first in PATH):

```powershell
$env:PATH='C:\msys64\mingw64\bin;C:\msys64\usr\bin;' + $env:PATH
C:\msys64\mingw64\bin\g++.exe -std=c++17 -O2 -D_WIN32_WINNT=0x0A00 main.cpp -o vectorforge.exe -lws2_32 -lcrypt32 -pthread
```

3) Install Ollama and pull models (exact):

```powershell
winget install --id Ollama.Ollama -e --accept-package-agreements --accept-source-agreements
ollama pull nomic-embed-text
ollama pull llama3.2:3b
ollama list
```

4) Set runtime environment variables (optional overrides):

```powershell
$env:SERVICE_HOST='127.0.0.1'
$env:SERVICE_PORT='11434'
$env:EMBED_MODEL='nomic-embed-text'
$env:GEN_MODEL='llama3.2:3b'
```

5) Start `vectorforge.exe` and open the UI:

```powershell
$env:PATH='C:\msys64\mingw64\bin;C:\msys64\usr\bin;'+$env:PATH
./vectorforge.exe
# Then point browser to:
# http://localhost:8080
```

API smoke tests (copy-paste PowerShell commands)

Status:

```powershell
Invoke-WebRequest -UseBasicParsing http://127.0.0.1:8080/status | Select-Object -ExpandProperty Content
```

Insert a simple demo vector:

```powershell
$payload='{"metadata":"smoke-test","category":"test","embedding":[0.01,0.02,0.03,0.04,0.05,0.06,0.07,0.08,0.09,0.10,0.11,0.12,0.13,0.14,0.15,0.16]}'
Invoke-WebRequest -UseBasicParsing -Method Post -Uri http://127.0.0.1:8080/insert -ContentType 'application/json' -Body $payload
```

Insert a document (requires embedding model):

```powershell
$doc='{"title":"Smoke Doc","text":"This is a short document for endpoint validation."}'
Invoke-WebRequest -UseBasicParsing -Method Post -Uri http://127.0.0.1:8080/doc/insert -ContentType 'application/json' -Body $doc
```

Ask a question (requires generation model):

```powershell
$ask='{"question":"Summarize the smoke doc in one short sentence.","k":3}'
Invoke-WebRequest -UseBasicParsing -Method Post -Uri http://127.0.0.1:8080/doc/ask -ContentType 'application/json' -Body $ask
```

## Troubleshooting (concise fixes)

- Missing std::thread / std::mutex: use the MSYS2 MinGW-w64 g++ in C:\msys64\mingw64\bin.
- Linker permission denied: stop running server before rebuild.
- 0xC000007B: ensure no 32-bit DLLs are shadowing mingw64; put C:\msys64\mingw64\bin first in PATH.
- Embedding/generation unavailable: verify ollama list and re-run ollama pull for the required models.

## Files of interest

- `main.cpp` — server, indexes, and `ServiceClient` abstraction
- `index.html` — browser UI
- `httplib.h` — single-header HTTP server library (included)

### Legal

- MIT License (see LICENSE)

