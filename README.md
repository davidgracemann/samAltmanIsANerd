# Applied & Foundational AI ( Resource Hub )
<img width="1000" height="500" alt="image" src="https://media3.giphy.com/media/v1.Y2lkPTc5MGI3NjExdmk5OWYzZmJsZWNsdmE5NGNleXVhY281b2JjYmNyd2FjcjIwa2czOCZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/FqpCVcIHXJuuCfNa4b/giphy.gif"/>

> Personal AI infrastructure, tooling, and research notes — **Gracemann365**

---

## 1. Distributed Computing Ecosytem

### 1.1 alphaNode (Primary Research Station)

**Role:** Master node for heavy model orchestration, AI research, and high-performance computing.

| Component | Specification | Operational Notes |
| --- | --- | --- |
| **Device** | Lenovo Legion Pro 5 (83NN003CIN) | System optimized for high-concurrency research. |
| **OS** | **CachyOS** (Linux Kernel - Optimized) | Low-latency kernel for real-time task scheduling. |
| **CPU** | Intel Core i9-14900HX (24C / 32T) | Base: 2.2 GHz / Boost: 5.8 GHz. |
| **RAM** | 32 GB DDR5-5600 | High-bandwidth memory for local RAG and agent logic. |
| **GPU** | NVIDIA RTX 5070 (8 GB GDDR7 VRAM) | Max model size: ~7.5 GB fully offloaded to GPU. |
| **Storage** | 1 TB NVMe Gen4 SSD | High-speed I/O for large dataset processing. |
| **Network** | Realtek 8922AE WiFi 7 | Primary link for high-speed node communication. |

### 1.2 betaNode (Distributed Support Node)

**Role:** Repurposed legacy workstation for headless background tasks and task mirroring.

| Component | Specification | Operational Notes |
| --- | --- | --- |
| **Device** | Lenovo IdeaPad Gaming 3 (15IMH05) | Repurposed for secondary compute tasks. |
| **OS** | **CachyOS** (Linux Kernel - Optimized) | Unified environment for seamless SSH orchestration. |
| **CPU** | Intel Core i7-10750H (6C / 12T) | Base: 2.6 GHz / Boost: 5.0 GHz. |
| **RAM** | 8 GB DDR4-2933 | **~6 GB free at idle** (CachyOS optimization). |
| **GPU** | NVIDIA GTX 1650 (4 GB GDDR6 VRAM) | Limited to small-quantization models (≤3.5 GB). |
| **Storage** | SATA SSD | Secondary storage for logs and build artifacts. |
| **Network** | Intel Wi-Fi 6 AX201 | Linked via 50 Mbps Spectra WLAN. |

---

## 2. Pro Chats 

### 2.1 Google AI Pro

| Item | Detail |
| :--- | :--- |
| **Active Model** | Gemini 3.1 Pro |
| **Context Window** | 1,000,000 tokens |
| **Multimodal Inputs** | Text · Audio (8.4h) · Images (900/prompt) · Video (45min) · PDF · Code Repos |
| **Thinking** | Automatic Chain-of-Thought (task-adaptive) |
| **Monthly Credits** | 1,000 AI credits · 100 videos |
| **API Access** | Google AI Studio + Gemini CLI |
| **Storage** | 2 TB Google One |
| **Key Tools** | NotebookLM (1,500 pages) · Whisk Animate · Flow · Workspace Integration |

### 2.2 Perplexity Pro (Airtel Bundle)

| Item | Detail |
| :--- | :--- |
| **Validity** | 12 months — active until **June 2026** |
| **Daily Pro Searches** | 300+ (multi-step reasoning) |
| **Search Modes** | Web · Academic · Writing · Wolfram\|Alpha · YouTube · Reddit |
| **API Credits** | $5/month |
| **File Analysis** | 100 files/week (PDF, CSV, Code, Images) |
| **Image Generation** | Unlimited — DALL-E 3 · Flux.1 · Playground |

#### Models Available via Perplexity Pro (February 2026)

| Model | Type |
| :--- | :--- |
| Sonar | Perplexity in-house (search-native) |
| GPT-5.2 | OpenAI |
| Claude 4.6 Sonnet | Anthropic (Thinking + Non-Thinking) |
| Gemini 3.1 Pro | Google |
| Grok 4.1 | xAI |
| o3 / o3-pro | OpenAI |

---

## 3. CLI Ecosystem

### 3.1 Local Stack (Offline-Capable · GTX 1650 · Zero Cloud)

> All models run via **Ollama v0.17.x**. One model loaded at a time. GPU-accelerated.

| Model | Size | VRAM Used | Role | Invoke |
| :--- | :--- | :--- | :--- | :--- |
| `qwen2.5-coder:3b` | 1.9 GB | ~1.9 GB GPU | **Programmer** — Python, scripting, debugging | `ollama run qwen2.5-coder:3b` |
| `deepseek-r1:1.5b` | 1.1 GB | ~1.1 GB GPU | **Mathematician** — GRE Quant, logic, CoT reasoning | `ollama run deepseek-r1:1.5b` |
| `phi3:mini` | 2.2 GB | ~2.2 GB GPU | **Fallback general** — fast summarization | `ollama run phi3:mini` |
| `gemma3:4b` | 2.5 GB | ~2.5 GB GPU | **Daily driver** — 128K context, 140+ languages, German | `ollama run gemma3:4b` |
| `moondream2` | 1.7 GB | ~1.7 GB GPU | **Vision** — screenshots, diagrams, handwritten text | `ollama run moondream2` |
| `qwen2.5:1.5b` | 0.9 GB | ~0.9 GB GPU | **Ultra-fast utility** — quick lookups, background agent | `ollama run qwen2.5:1.5b` |
| `nomic-embed-text` | 0.27 GB | ~0.27 GB GPU | **Embeddings only** — powers memory vault indexing | auto via `basic-memory` |

#### Memory Layer

| Tool | Role | Command |
| :--- | :--- | :--- |
| `basic-memory` | MCP server — indexes `EpiMemory/` vault, serves context to all LLM tools | `basic-memory mcp` |
| `EpiMemory/` vault | Persistent Markdown memory store — profile, sessions, learned facts, hallucination log | `C:\Users\Gracemann365\EpiMemory\` |
| `ask.py` | Memory-injected Ollama wrapper — enriches every prompt with vault context before inference | `python ask.py "query" --model phi3:mini` |

### 3.2 Cloud Stack (Internet-Required · API-Routed via LiteLLM)

> LiteLLM runs locally as a proxy (`localhost:8082`), translating Anthropic/OpenAI formats to any cloud model API.

| Model | Provider | Access Method | Best Used For |
| :--- | :--- | :--- | :--- |
| **Kimi K2.5** | Moonshot AI | LiteLLM → `moonshot/kimi-k2.5` | Heavy multimodal + 1T param agentic tasks |
| **Kimi K2 Thinking** | Moonshot AI | LiteLLM → `moonshot/kimi-k2-thinking` | Frontier-level chain-of-thought math |
| **Gemini 3.1 Pro** | Google | Gemini CLI (native) + LiteLLM | 1M context, document analysis, multimodal |

#### Gemini CLI

| Item | Detail |
| :--- | :--- |
| **Auth** | API Key via `api.google.dev` (Google AI Studio) |
| **Free Tier** | Gemini 3.1 Pro — 1,000 RPD · 15 RPM |
| **Agentic Features** | ReAct loop · Shell execution · File system read/write |
| **MCP Config** | `~/.gemini/settings.json` → points to `basic-memory mcp` server |

### 3.3 Agentic Coding Tools

| Tool | Backend | Memory Connected | Use Case |
| :--- | :--- | :--- | :--- |
| **Claude Code** | Kimi K2.5 via LiteLLM proxy | Via MCP → `basic-memory` | Primary agentic coding sessions |
| **Codex CLI** | GPT-5.2 / local fallback | `CODEX_SYSTEM_PROMPT` env var | Repo-level code modification |
| **Gemini CLI** | Gemini 3.1 Pro | Native MCP → `basic-memory` | Long-context analysis + shell tasks |
| **Ollama + ask.py** | Local GPU models | Direct vault injection | Offline coding + reasoning sessions |

---

## 4. Operational Decision Tree

```
Query received
     │
     ├── Offline / private?
     │        ├── Code task      → qwen2.5-coder:3b
     │        ├── Math / logic   → deepseek-r1:1.5b
     │        ├── Image / vision → moondream2
     │        ├── German / long  → gemma3:4b
     │        └── Quick lookup   → qwen2.5:1.5b
     │
     └── Online / heavy task?
              ├── Frontier reasoning  → Kimi K2 Thinking (LiteLLM)
              ├── Multimodal agentic  → Kimi K2.5 (LiteLLM)
              └── 1M context document → Gemini 3.1 Pro (Gemini CLI)
```


*Last updated: February 2026 · Hardware: Gracemann365 · Maintainer: David Grace*
