# AI Engineer Roadmap — 10-Day Study Plan
### Based on roadmap.sh/ai-engineer | Tailored for a Python Backend Developer

---

> **The goal:** High-level intuitive understanding of every major topic on the roadmap.
> Not deep mastery — that comes later. Think of this as building a mental map so nothing feels
> alien when you go deeper.

---

## Guiding Principles

- **1 topic cluster per day**, with a primary resource and a fallback if you want more.
- **Spend time building intuition, not memorizing.** If a concept clicks, move on.
- **Use Claude actively** — after each section, explain the concept back in your own words and ask Claude to fill in gaps or challenge your understanding.
- **Your Python/backend background is an asset.** Many concepts will map naturally onto things you already know.

---

## Day 1 — Foundations & Orientation
**Topics:** What is an AI Engineer, AI Engineer vs ML Engineer, LLMs (high level), Common Terminology, AI vs AGI

This is your orientation day. The goal is to understand *where you stand* in the AI landscape and what the role of an AI Engineer actually is vs a researcher or ML engineer.

### Resources
- **Primary:** [roadmap.sh/ai-engineer](https://roadmap.sh/ai-engineer) — read the intro nodes carefully
- **Watch:** Andrej Karpathy's "Intro to Large Language Models" (1hr YouTube) — the single best high-level LLM explainer that exists
- **Read:** OpenAI's "How ChatGPT Works" blog post (15 min)

### Key things to understand by end of day
- The difference between training a model vs using a pre-trained model
- What inference means in plain terms
- What tokens are and why they matter
- The difference between AI Engineer / ML Engineer / Researcher roles

---

## Day 2 — Pre-trained Models & the Model Landscape
**Topics:** Pre-trained Models, Benefits/Limitations, Popular Models (OpenAI, Claude, Gemini, Mistral, Cohere, Azure AI, Hugging Face)

You're already a user of these models. Today you become someone who understands *what* they are and *how to choose* between them.

### Resources
- **Primary:** Hugging Face's "NLP Course" Chapter 1 only — [huggingface.co/learn/nlp-course/chapter1](https://huggingface.co/learn/nlp-course/chapter1) (free, 1-2 hrs)
- **Browse:** The model cards on Hugging Face Hub for GPT-4o, Claude 3.5, Mistral 7B — just read the descriptions and capability sections
- **Reference:** [platform.openai.com/docs/models](https://platform.openai.com/docs/models) — skim the model list and context window sizes

### Key things to understand by end of day
- What "context length" means and why it matters practically
- The open vs closed source trade-offs (cost, privacy, customization)
- Why you'd pick Mistral over GPT-4o or vice versa
- What a "knowledge cutoff" is and its implications

---

## Day 3 — OpenAI API & Prompt Basics
**Topics:** OpenAI API, Chat Completions API, Writing Prompts, OpenAI Playground, Managing Tokens, Pricing

This is where things get hands-on. You're a developer — reading docs and making API calls is your natural habitat.

### Resources
- **Primary:** [platform.openai.com/docs/quickstart](https://platform.openai.com/docs/quickstart) — work through the quickstart with Python
- **Play:** Spend 30–45 min in the OpenAI Playground experimenting with system prompts and temperature settings
- **Read:** OpenAI tokenizer page + tiktoken library docs (30 min)

### Key things to understand by end of day
- How the messages array (system/user/assistant) structures a conversation
- How temperature and top_p affect outputs
- How to count tokens and estimate API costs
- The difference between a system prompt and a user prompt

---

## Day 4 — Prompt Engineering
**Topics:** Full Prompt Engineering section — zero-shot, few-shot, chain-of-thought, ReAct prompting

Prompt engineering sounds soft but it has real engineering depth. Today you go from "writing prompts" to understanding *why* certain prompting strategies work.

### Resources
- **Primary:** [promptingguide.ai](https://www.promptingguide.ai) — read the core techniques section (zero-shot, few-shot, chain-of-thought, ReAct). Budget 2–3 hrs.
- **Supplement:** Anthropic's Prompt Engineering docs at [docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)

### Key things to understand by end of day
- Why chain-of-thought prompting improves reasoning tasks
- What ReAct prompting is and how it connects to agents (preview of Day 8)
- How few-shot examples shape model behavior
- The difference between a well-structured and a lazy prompt

---

## Day 5 — Embeddings & Semantic Search
**Topics:** What are Embeddings, Use Cases (semantic search, recommendations, anomaly detection), OpenAI Embeddings API, Open-source alternatives (Sentence Transformers)

Embeddings are the bridge between text and math. This is where your research curiosity and engineering path start to converge.

### Resources
- **Primary:** [platform.openai.com/docs/guides/embeddings](https://platform.openai.com/docs/guides/embeddings) — read fully (45 min)
- **Watch:** 3Blue1Brown's "Word Embeddings" video or Jay Alammar's "The Illustrated Word2Vec" — both build genuine intuition
- **Code:** Write a small Python snippet that embeds 5 sentences and computes cosine similarity between them using the OpenAI API

### Key things to understand by end of day
- What an embedding vector *is* geometrically
- Why semantically similar sentences are "close" in vector space
- How embeddings enable semantic search vs keyword search
- The difference between OpenAI embeddings and Sentence Transformers

---

## Day 6 — Vector Databases
**Topics:** Vector DBs (Chroma, Pinecone, Qdrant, FAISS, Weaviate), Indexing Embeddings, Similarity Search

This is the storage layer that makes embeddings practically useful at scale.

### Resources
- **Primary:** Chroma's getting started docs — [docs.trychroma.com](https://docs.trychroma.com) — Chroma is the simplest to run locally, start here
- **Read:** Pinecone's "What is a Vector Database?" conceptual guide (pinecone.io/learn/vector-database) — 30 min
- **Code:** Extend yesterday's embeddings script to store and retrieve from a local Chroma instance

### Key things to understand by end of day
- Why you can't just use PostgreSQL for this (and when you can, with pgvector)
- What HNSW indexing is conceptually (approximate nearest neighbour)
- Trade-offs between Chroma (local/simple) vs Pinecone (managed/scalable) vs Qdrant (self-hosted/production)

---

## Day 7 — RAG (Retrieval Augmented Generation)
**Topics:** RAG usecases, RAG vs Fine-tuning, Chunking, Embedding, Retrieval, Generation, Implementing RAG, LangChain/LlamaIndex intro

RAG is the most practically important concept on this entire roadmap right now. Almost every real-world LLM application uses some form of it.

### Resources
- **Primary:** LangChain's RAG tutorial — [python.langchain.com/docs/tutorials/rag](https://python.langchain.com/docs/tutorials/rag) — work through it in Python
- **Read:** "RAG vs Fine-tuning" — any good blog post on this (Pinecone has a clear one)
- **Conceptual:** Think of RAG as the same pattern as dependency injection in FastAPI — you're injecting external context into a function (the LLM call) at runtime. That mental model will serve you well.

### Key things to understand by end of day
- The full RAG pipeline: document → chunk → embed → store → retrieve → augment prompt → generate
- Why chunking strategy matters (chunk too large = noise, too small = lost context)
- When RAG is the right choice vs fine-tuning
- What LangChain actually provides (plumbing/orchestration, not intelligence)

---

## Day 8 — AI Agents
**Topics:** What are Agents, Agent use cases, ReAct prompting in practice, OpenAI Functions/Tools, Building AI Agents

Agents are LLMs that can take actions — calling APIs, running code, searching the web. This is where LLMs go from answering questions to doing work.

### Resources
- **Primary:** OpenAI's function calling docs — [platform.openai.com/docs/guides/function-calling](https://platform.openai.com/docs/guides/function-calling) — implement a simple tool-calling example
- **Watch:** A short YouTube walkthrough of building a basic ReAct agent from scratch (search "ReAct agent Python from scratch" — Karpathy or similar)
- **Conceptual:** Think of an agent as a while loop: LLM decides action → action executes → result fed back to LLM → repeat until done

### Key things to understand by end of day
- The difference between a chatbot and an agent
- How tool/function calling works mechanically in the API
- What the ReAct loop (Reason + Act) looks like in practice
- Why agents can fail and how to guard against it (hallucinated tool calls, infinite loops)

---

## Day 9 — AI Safety, Ethics & Multimodal AI
**Topics:** AI Safety Issues, Prompt Injection, Bias and Fairness, Security/Privacy, Multimodal AI (vision, image gen, audio, speech)

Two distinct topic clusters in one day — neither requires deep implementation, more conceptual grounding.

### Resources

**AI Safety (morning, ~2 hrs)**
- **Read:** Anthropic's usage policies and model card — understanding how frontier labs think about safety is itself useful signal
- **Read:** OWASP Top 10 for LLMs — [owasp.org/www-project-top-10-for-large-language-model-applications](https://owasp.org/www-project-top-10-for-large-language-model-applications) — practical security lens

**Multimodal (afternoon, ~2 hrs)**
- **Primary:** OpenAI Vision API docs + DALL-E API docs + Whisper API docs — read all three conceptually, implement one small example of your choice
- **Browse:** Hugging Face's multimodal model collection to see what's available open source

### Key things to understand by end of day
- What prompt injection is and why it's dangerous in agent systems
- The three main multimodal modalities: vision input, image generation, audio/speech
- When to use Whisper vs a cloud speech API
- What "grounding" means in multimodal context

---

## Day 10 — Open Source AI & Synthesis
**Topics:** Open vs Closed Source, Hugging Face Hub, Ollama (running models locally), Transformers.js, review and connect everything

Your final day is about filling gaps, running something locally, and connecting all the dots.

### Resources
- **Primary:** Install Ollama ([ollama.ai](https://ollama.ai)) and run a model locally — `ollama run llama3.2` or `ollama run mistral`. Make an API call to it from Python.
- **Explore:** Hugging Face Hub — browse Tasks section, understand what's available for text, image, audio, multimodal
- **Synthesis:** Write a 1-page (or markdown file) summary of the full pipeline: how would you build a RAG-powered assistant that can answer questions about a PDF document? Trace every component from Day 5 onwards.

### Key things to understand by end of day
- How Ollama lets you run open source models with an OpenAI-compatible API (maps directly to your FastAPI knowledge)
- The Hugging Face ecosystem — Hub, Transformers library, Inference API
- How all the pieces connect: embeddings → vector DB → RAG → agent → multimodal

---

## The Full Picture (What You've Covered)

```
LLMs & Models → Prompt Engineering → Embeddings → Vector DBs
       ↓                                               ↓
   API Usage                                          RAG
       ↓                                               ↓
  Fine-tuning                                       Agents
       ↓                                               ↓
  Safety/Ethics                               Multimodal AI
       ↓
  Open Source (Ollama, HuggingFace)
```

---

## After These 10 Days

You'll have a working mental model of the entire AI engineering stack. The natural next step is picking **one area to go deep on** — RAG and agents is the most practical for a backend developer. Build one complete project that stitches everything together.

For the research track you're keeping parallel: Karpathy's Neural Networks: Zero to Hero will start making much more sense once you've gone through these 10 days. The gap between "using models" and "understanding models" narrows significantly with this foundation.

---

*Generated from roadmap.sh/ai-engineer | June 2026*
