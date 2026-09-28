# RiftIQ Roadmap

> **RiftIQ** is a Retrieval-Augmented Generation (RAG) assistant that answers League of Legends
> game-knowledge questions (champions, abilities, items, runes, and what changed in each patch),
> grounded in official data and cited sources. It is built and understood locally first, then
> migrated to AWS.

**The resume line this project builds toward:**
*"Built and deployed RiftIQ, a RAG-based League of Legends assistant (Python, FastAPI, PostgreSQL/pgvector,
Claude, Next.js) on AWS (ECS Fargate, RDS, S3, CloudFront, CDK), with an automated patch-ingestion pipeline
and an evaluation suite achieving X% retrieval accuracy on a Y-question benchmark."*

---

## Table of Contents
- [How to Use This Roadmap](#how-to-use-this-roadmap)
- [Architecture at a Glance](#architecture-at-a-glance)
- [Tools & Dependencies](#tools--dependencies)
- [Accounts You'll Need](#accounts-youll-need)
- [Design Principle: Build for the Migration](#design-principle-build-for-the-migration)
- [Glossary](#glossary)
- **Part A: Build It Locally**
  - [Phase 0: Setup & Foundations](#phase-0-setup--foundations)
  - [Phase 1: Learn the Concepts](#phase-1-learn-the-concepts)
  - [Phase 2: Your First LLM Call](#phase-2-your-first-llm-call)
  - [Phase 3: Data Acquisition, Data Dragon](#phase-3-data-acquisition-data-dragon)
  - [Phase 4: Data Acquisition, Narrative Sources](#phase-4-data-acquisition-narrative-sources)
  - [Phase 5: Cleaning & Document Modeling](#phase-5-cleaning--document-modeling)
  - [Phase 6: Chunking](#phase-6-chunking)
  - [Phase 7: Embeddings & Vector Store](#phase-7-embeddings--vector-store)
  - [Phase 8: Retrieval](#phase-8-retrieval)
  - [Phase 9: Generation](#phase-9-generation)
  - [Phase 10: Evaluation](#phase-10-evaluation)
  - [Phase 11: Backend API](#phase-11-backend-api)
  - [Phase 12: Frontend Chat UI](#phase-12-frontend-chat-ui)
  - [Phase 13: Local Polish & Containerization](#phase-13-local-polish--containerization)
- **Part B: Migrate to AWS**
  - [Phase 14: AWS Foundations](#phase-14-aws-foundations)
  - [Phase 15: Storage to S3](#phase-15-storage-to-s3)
  - [Phase 16: (Optional) LLM & Embeddings to Bedrock](#phase-16-optional-llm--embeddings-to-bedrock)
  - [Phase 17: Infrastructure as Code with CDK](#phase-17-infrastructure-as-code-with-cdk)
  - [Phase 18: Deploy](#phase-18-deploy)
  - [Phase 19: Operate](#phase-19-operate)
  - [Phase 20: CI/CD to AWS](#phase-20-cicd-to-aws)
- **Part C: Resume Polish**
  - [Phase 21: Tell the Story](#phase-21-tell-the-story)
- [Stretch Goals](#stretch-goals)
- [Common Pitfalls](#common-pitfalls)
- [Reference Links](#reference-links)

---

## How to Use This Roadmap

- **Work top to bottom.** Each phase depends on the ones before it.
- Every phase has the same five parts:
  - **Goal:** what the phase accomplishes and why it matters.
  - **Tasks:** *what* to do, as checkboxes. The *how* is yours to figure out; that's where the learning is.
  - **Understand:** questions to research until you can answer them in your own words. Don't skip these; interviewers will ask exactly these kinds of questions.
  - **Docs:** official documentation to lean on.
  - **Done when:** concrete criteria. Don't move on until they're all true.
- **Keep a `docs/decisions.md` log.** Whenever you choose between options (chunk size, embedding model, index type), write down what you picked and why. It becomes your interview material.
- **Commit early and often.** A clean commit history is part of the portfolio.
- Time estimates assume part-time work (~8–10 hrs/week) and are rough.
- 🏁 marks a milestone: a point where you have something complete and demo-able.

---

## Architecture at a Glance

### Part A: Local

```
                     ┌──────────────────── INGESTION PIPELINE ────────────────────┐
 Data Dragon (JSON) ─┤                                                            │
 Patch notes (HTML) ─┼─► fetch ─► data/raw/<patch>/ ─► clean ─► chunk ─► embed ─┐ │
 LoL Wiki (HTML) ────┤                                                          │ │
                     └──────────────────────────────────────────────────────────┼─┘
                                                                                ▼
                                                              ┌───────────────────────────┐
                                                              │ PostgreSQL + pgvector     │
                                                              │ (Docker)                  │
                                                              └─────────────▲─────────────┘
                                                                            │ similarity search
 ┌────────────────┐   HTTP / SSE   ┌──────────────────────────┐             │
 │  Next.js chat  │ ─────────────► │ FastAPI backend          │ ────────────┘
 │  (localhost)   │ ◄───────────── │  embed query → retrieve  │
 └────────────────┘   streamed     │  → build prompt → LLM    │ ──► Claude (Anthropic API)
                      answer       └──────────────────────────┘
```

### Part B: AWS (target)

```
 Users ─► CloudFront ─► S3 (static Next.js site)
            │
            └──► Application Load Balancer ─► ECS Fargate (FastAPI) ─┬─► RDS PostgreSQL + pgvector (private subnet)
                                                                     ├─► Claude (Anthropic API or Amazon Bedrock)
                                                                     └─► Secrets Manager

 EventBridge Scheduler ─► ECS Fargate task (ingestion pipeline) ─► S3 (raw data) + RDS
 CloudWatch ◄── logs / metrics / alarms from everything
 GitHub Actions ──(OIDC)──► ECR + CDK deploy
```

---

## Tools & Dependencies

### Part A: Local stack

| Layer | Tool | Why this choice |
|---|---|---|
| Language (backend/pipeline) | **Python 3.12+** | The AI/ML ecosystem is Python-first |
| Env/dependency manager | **uv** | Fast, modern, handles Python versions and lockfiles |
| LLM | **Claude** via the Anthropic API (`anthropic` Python SDK) | High-quality answers, simple SDK, one API key to start |
| Embeddings | **sentence-transformers** (local, free), or **Voyage AI** (hosted, higher quality) | Local means no cost and no cloud dependency while learning |
| Vector database | **PostgreSQL + pgvector** (run in Docker) | Vectors and regular SQL in one DB, and the same engine you'll use on AWS RDS |
| DB driver | **psycopg** (v3) | The standard Postgres driver for Python |
| HTTP client | **httpx** | Modern requests-style client with async support |
| HTML parsing | **BeautifulSoup4** | Extracts text from patch notes and wiki pages |
| Data validation | **Pydantic** | Typed schemas for documents, chunks, and API payloads |
| Backend API | **FastAPI** + **Uvicorn** | Async, typed, auto-generated docs, easy streaming |
| Frontend | **Next.js** (React + TypeScript) + **Tailwind CSS** | Industry-standard web stack |
| Evaluation | A hand-built golden Q&A set, optionally **Ragas** | Proves the system works, and gives you numbers |
| Testing & quality | **pytest**, **ruff** (lint/format), **mypy** (types), ESLint/Prettier | Professional hygiene that recruiters notice |
| Containers | **Docker** + **Docker Compose** | One command runs the whole stack |
| CI | **GitHub Actions** | Automated checks on every push |

### Part B: AWS mapping

| Local piece | AWS replacement |
|---|---|
| `data/raw/` folder | **Amazon S3** |
| Postgres in Docker | **Amazon RDS for PostgreSQL** (pgvector supported) |
| Anthropic API / local embeddings | **Amazon Bedrock** (Claude + Titan/Cohere embeddings), optional |
| `.env` file | **AWS Secrets Manager** / SSM Parameter Store |
| `docker compose up` (API) | **Amazon ECR** (images) + **ECS on Fargate** behind an **Application Load Balancer** |
| Next.js dev server | Static export on **S3 + CloudFront**, with **ACM** certificate (+ optional **Route 53** domain) |
| cron / manual script | **EventBridge Scheduler** → ECS task |
| Terminal logs | **CloudWatch** Logs, Metrics, Alarms |
| Clicking things by hand | **AWS CDK** (Infrastructure as Code) |
| CI only | GitHub Actions + **OIDC** role → automatic deploys |

---

## Accounts You'll Need

| When | Account | Notes |
|---|---|---|
| Phase 0 | GitHub | Public repo = portfolio |
| Phase 2 | Anthropic Console (platform.claude.com) | API key + a small amount of prepaid credit; set a spend limit |
| Phase 7 (optional) | Voyage AI | Only if you choose hosted embeddings |
| Phase 14 | AWS | Needs a credit card; set up billing alerts **immediately** |

**Data Dragon needs no Riot API key.** It's a public static CDN. You only need a Riot developer key if
you later add live player/match lookups (a stretch goal).

---

## Design Principle: Build for the Migration

Because you'll move to AWS later, bake in these habits from day one. They're also just good engineering:

1. **All configuration comes from environment variables** (DB URL, API keys, model names, storage location). Nothing environment-specific is hard-coded.
2. **Put swappable things behind small interfaces:**
   - A **storage** layer (`save`/`load` raw files) with a *local-disk* implementation now and an *S3* implementation later.
   - An **LLM client** with an *Anthropic* implementation now and an optional *Bedrock* implementation later.
   - An **embedder** with a *local* implementation now and an optional *Bedrock/Voyage* implementation later.
3. **Keep the pipeline and the API separate.** Ingestion runs as a batch job, and the API only reads.
4. **Make ingestion idempotent.** Running it twice shouldn't duplicate data.

If you do this, Part B is a matter of adding implementations and changing config, not rewriting.

---

## Glossary

| Term | Meaning |
|---|---|
| **LLM** | Large Language Model: predicts text, one token at a time |
| **Token** | A chunk of text (~¾ of a word) that LLMs read and write; also the billing unit |
| **Context window** | The maximum number of tokens an LLM can consider in one request |
| **Hallucination** | When an LLM states something false with confidence |
| **Grounding** | Forcing answers to be based on provided source material |
| **Embedding** | A list of numbers (a vector) representing a text's meaning |
| **Vector / dimension** | The embedding itself; its length (e.g. 384, 1024) is its dimension |
| **Cosine similarity** | A measure of how closely two vectors point in the same direction, i.e. how similar in meaning |
| **Chunk** | A small, self-contained piece of a document that gets embedded and retrieved |
| **Top-k** | Retrieve the *k* most similar chunks |
| **Hybrid search** | Combining keyword (full-text) search with vector (semantic) search |
| **Reranking** | A second pass that reorders retrieved chunks with a more accurate model |
| **ANN index (HNSW)** | Approximate nearest neighbor index: makes vector search fast at scale |
| **System prompt** | Instructions that set the LLM's role and rules |
| **Streaming / SSE** | Sending the answer token by token as it's generated (Server-Sent Events) |
| **Idempotent** | Running an operation many times has the same result as running it once |
| **Container** | A packaged app plus its dependencies that runs the same everywhere |
| **IaC** | Infrastructure as Code: defining cloud resources in version-controlled code |

---

# Part A: Build It Locally

Everything in Part A runs on your laptop. **No AWS account needed.**

---

## Phase 0: Setup & Foundations
⏱ ~1 week

**Goal:** A clean, professional project skeleton you won't have to restructure later.

**Tasks**
- [ ] Install the core tooling: Git, Python 3.12+ (via uv), Node.js LTS, Docker Desktop, and an editor (VS Code or similar).
- [ ] Initialize a Git repository and create a public GitHub repo for RiftIQ.
- [ ] Create a monorepo layout with top-level folders for `backend/`, `pipeline/`, `frontend/`, `data/`, `docs/`, and `evals/`.
- [ ] Set up a Python project with uv (dependency file + lockfile).
- [ ] Create a `.gitignore` that excludes secrets, virtual environments, `node_modules`, and large raw data.
- [ ] Create a `.env.example` listing every config variable you expect (with no real values), and a real `.env` that is git-ignored.
- [ ] Write a README stub: project name, one-paragraph description, "status: in progress".
- [ ] Read Riot's developer policies and add Riot's required "Legal Jibber Jabber" disclaimer to the README.
- [ ] Create `docs/decisions.md` for your decision log.

**Understand**
- What's the difference between a virtual environment and a system Python install, and why does it matter?
- What's a lockfile, and why do you commit it?
- Why must secrets never be committed, even to a private repo? What should you do if you accidentally commit one?
- What does Riot allow and forbid for fan projects?

**Docs**
- [Pro Git book](https://git-scm.com/book/en/v2)
- [uv documentation](https://docs.astral.sh/uv/)
- [Riot Developer Policies (includes Legal Jibber Jabber)](https://developer.riotgames.com/policies/general)
- [Riot Games legal](https://www.riotgames.com/en/legal)

**Done when**
- [ ] The repo is on GitHub with the folder layout, README (with disclaimer), `.gitignore`, and `.env.example`.
- [ ] `git status` shows no `.env` file being tracked.

---

## Phase 1: Learn the Concepts
⏱ ~1 week · *No code in this phase*

**Goal:** Understand what you're building before you build it. RAG makes much more sense once the pieces click.

**Tasks**
- [ ] Read an introduction to LLMs and how they generate text.
- [ ] Read about embeddings and semantic search.
- [ ] Read the original RAG paper's abstract and introduction (skim the rest).
- [ ] Read Anthropic's "Contextual Retrieval" post end to end.
- [ ] Skim the pgvector README to see what a vector database does.
- [ ] Draw your own RAG diagram (paper is fine) and explain it out loud in under 2 minutes.
- [ ] Write a `docs/concepts.md` page summarizing RAG in your own words.

**Understand**
- Why can't an LLM just "know" the current patch's champion stats?
- What problem does RAG solve that fine-tuning doesn't (and vice versa)?
- What is an embedding, intuitively? Why do "Ahri's ultimate" and "Spirit Rush" end up close together in vector space?
- Why is keyword search alone not enough? Why is semantic search alone not enough?
- What are the two halves of RAG, and which one usually causes bad answers?
- What does a context window limit mean for how much retrieved text you can include?

**Docs**
- [Intro to Claude](https://platform.claude.com/docs/en/intro)
- [Anthropic embeddings guide](https://platform.claude.com/docs/en/build-with-claude/embeddings)
- [Introducing Contextual Retrieval (Anthropic)](https://www.anthropic.com/news/contextual-retrieval)
- [RAG paper: Lewis et al., 2020](https://arxiv.org/abs/2005.11401)
- [pgvector README](https://github.com/pgvector/pgvector)

**Done when**
- [ ] You can explain the whole RAG flow (ingest → chunk → embed → store → retrieve → generate) without notes.
- [ ] `docs/concepts.md` is committed.

---

## Phase 2: Your First LLM Call
⏱ ~3–5 days

**Goal:** Talk to Claude from Python and see its limitations firsthand.

**Tasks**
- [ ] Create an Anthropic Console account, add a small amount of credit, set a monthly spend limit, and generate an API key (stored in `.env`).
- [ ] Install the Anthropic Python SDK.
- [ ] Write a script that sends a question and prints the answer.
- [ ] Experiment with a system prompt, `temperature`, and `max_tokens`, and note how each changes the output.
- [ ] Print the token usage for each call and estimate its cost using the pricing page.
- [ ] Make a streaming version that prints the answer as it arrives.
- [ ] Ask it about a recent LoL patch and write down where it's wrong or vague. **This is the problem RiftIQ solves.**
- [ ] Wrap your LLM calls behind your own small `LLMClient` interface (see the [design principle](#design-principle-build-for-the-migration)).
- [ ] Pick a model (e.g. Claude Haiku 4.5 for cheap iteration, Claude Sonnet 5 for quality) and make the model name a config value.

**Understand**
- What's the difference between the system prompt and a user message?
- What does temperature control? When would you want it low?
- How are input and output tokens billed differently?
- Why does streaming improve perceived speed without changing total generation time?
- What is a model's "training cutoff", and how does it relate to patch data?

**Docs**
- [Getting started](https://platform.claude.com/docs/en/get-started)
- [Messages API reference](https://platform.claude.com/docs/en/api/messages)
- [Client SDKs](https://platform.claude.com/docs/en/api/client-sdks) · [Python SDK on GitHub](https://github.com/anthropics/anthropic-sdk-python)
- [Models overview](https://platform.claude.com/docs/en/about-claude/models/overview)
- [Pricing](https://platform.claude.com/docs/en/about-claude/pricing)
- [Streaming](https://platform.claude.com/docs/en/build-with-claude/streaming)

**Done when**
- [ ] A script can ask Claude a question with both streaming and non-streaming output.
- [ ] The API key lives only in `.env`, and the model name is configurable.
- [ ] You've documented at least three examples of Claude being wrong or outdated about LoL.

---

## Phase 3: Data Acquisition, Data Dragon
⏱ ~1 week

**Goal:** Pull Riot's official structured game data for the current patch.

**Tasks**
- [ ] Explore Data Dragon in your browser: find the versions list and open a champion JSON file by hand.
- [ ] Write a fetcher that finds the latest patch version.
- [ ] Download the full champion list, then each champion's detailed JSON (abilities, passives, stats, tips, lore).
- [ ] Download the items, runes (runesReforged), and summoner spells JSON.
- [ ] Save everything under `data/raw/<patch>/ddragon/` **through your storage interface** (local implementation).
- [ ] Record metadata about each run (patch version, timestamp, file count).
- [ ] Add polite behavior: a reasonable delay between requests, retries on failure, and skip files you already have.
- [ ] Look at the ability tooltips and note the HTML-like tags and `{{ placeholders }}` you'll need to clean later.

**Understand**
- What's the difference between Data Dragon and the Riot API? Why doesn't Data Dragon need a key?
- How are Data Dragon URLs structured (version, language, file type)?
- Why save raw data before processing it, instead of transforming it on the fly?
- What does "idempotent fetching" mean for this script?
- What is CommunityDragon, and when might its data be more complete than Data Dragon's?

**Docs**
- [Riot Developer Docs: League of Legends (Data Dragon section)](https://developer.riotgames.com/docs/lol)
- [Data Dragon versions list](https://ddragon.leagueoflegends.com/api/versions.json)
- [CommunityDragon](https://www.communitydragon.org/)
- [HTTPX](https://www.python-httpx.org/)

**Done when**
- [ ] One command downloads all the champion, item, rune, and summoner data for the latest patch.
- [ ] Running it again doesn't re-download or duplicate anything.
- [ ] The raw data is git-ignored (or only a tiny sample is committed for tests).

---

## Phase 4: Data Acquisition, Narrative Sources
⏱ ~1 week

**Goal:** Add the "why" and "what changed" context that structured JSON lacks.

**Tasks**
- [ ] Find the official patch notes listing and pick the last several patches to ingest.
- [ ] Check `robots.txt` and the terms of every site before fetching anything from it.
- [ ] Fetch the patch notes pages and save the raw HTML under `data/raw/<patch>/patch-notes/`.
- [ ] Decide which LoL Wiki pages add value (e.g. champion pages, mechanics pages) and fetch a limited set.
- [ ] Read the wiki's license (CC BY-SA) and plan how you'll attribute it in answers and in the UI.
- [ ] Save the source URL, retrieval date, and license with every raw document.
- [ ] Rate-limit and identify yourself with a descriptive User-Agent.

**Understand**
- What does robots.txt communicate, and why should you respect it?
- What does CC BY-SA require of you when you display wiki content?
- Why is patch notes text harder to parse than JSON? What structure does it have (headers per champion, bullet changes, before → after values)?
- What are the risks of scraping (breaking layout changes, bans, legal)? How do you minimize them?

**Docs**
- [League of Legends patch notes](https://www.leagueoflegends.com/en-us/news/tags/patch-notes/)
- [League of Legends Wiki](https://wiki.leagueoflegends.com/en-us/) · [Wiki copyright/license](https://wiki.leagueoflegends.com/en-us/League_of_Legends_Wiki:Copyrights)
- [Beautiful Soup documentation](https://www.crummy.com/software/BeautifulSoup/bs4/doc/)

**Done when**
- [ ] Raw HTML for several patches' notes and your selected wiki pages is saved, with source metadata.
- [ ] Your attribution plan is written in `docs/decisions.md`.

---

## Phase 5: Cleaning & Document Modeling
⏱ ~1 week

**Goal:** Turn messy raw files into clean, uniform documents with rich metadata.

**Tasks**
- [ ] Design a `Document` schema: the text plus metadata such as `source_type` (champion / item / rune / patch_note / wiki), `champion`, `patch`, `title`, `url`, and `license`.
- [ ] Write converters that turn each Data Dragon JSON type into readable prose (e.g. a champion's passive and each ability as natural-language text).
- [ ] Strip or translate the HTML tags and tooltip placeholders in ability descriptions.
- [ ] Extract clean text from the patch notes HTML, preserving structure (which champion, which ability, old → new values).
- [ ] Extract the main content from wiki pages, dropping navigation, ads, and boilerplate.
- [ ] Deduplicate and validate every document against your schema.
- [ ] Save the processed documents to `data/processed/<patch>/` (e.g. JSON Lines).
- [ ] Write unit tests for your converters, using small sample fixtures.

**Understand**
- Why is metadata as important as the text itself? (Hint: filtering and citations.)
- Why convert JSON to prose instead of embedding raw JSON?
- What makes a good document boundary for this domain?
- What does Pydantic validation catch that you'd otherwise miss?

**Docs**
- [Pydantic](https://docs.pydantic.dev/latest/)
- [pytest](https://docs.pytest.org/)

**Done when**
- [ ] Every raw source produces validated documents with complete metadata.
- [ ] You can open a processed file and read a champion's kit as clean English.
- [ ] The converter tests pass.

---

## Phase 6: Chunking
⏱ ~3–5 days

**Goal:** Split documents into pieces that retrieve well.

**Tasks**
- [ ] Measure how long your documents are in tokens.
- [ ] Implement at least two chunking strategies. For example:
  - **Structural:** one chunk per ability, per item, per champion's section of the patch notes.
  - **Fixed-size:** token windows with overlap.
- [ ] Make sure every chunk carries its parent document's metadata, plus a chunk ID.
- [ ] Try a "contextual header": prepend a short line of context (e.g. champion + patch + section) to each chunk.
- [ ] Inspect 20 random chunks from each strategy by eye. Are they self-contained and understandable alone?
- [ ] Record your chosen default and why in `docs/decisions.md` (you'll validate it in Phase 10).

**Understand**
- What goes wrong when chunks are too big? Too small?
- What is chunk overlap for?
- Why does a chunk like "Cooldown reduced from 12 to 10 seconds" retrieve poorly without context, and how does contextual retrieval fix that?
- Why should chunk IDs be deterministic?

**Docs**
- [Introducing Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval)
- [Token counting](https://platform.claude.com/docs/en/build-with-claude/token-counting)

**Done when**
- [ ] Chunks are produced with deterministic IDs and full metadata.
- [ ] Your chunking decision is documented.

---

## Phase 7: Embeddings & Vector Store
⏱ ~1–2 weeks

**Goal:** Store every chunk as a searchable vector in Postgres.

**Tasks**
- [ ] Run PostgreSQL with the pgvector extension via Docker Compose, using a persistent volume.
- [ ] Choose an embedding model (start with a small sentence-transformers model), note its dimension, and put it behind your `Embedder` interface.
- [ ] Design a table for the chunks: ID, text, metadata columns (and/or a JSON column), patch, and the embedding vector.
- [ ] Manage your schema with SQL migration files (versioned, committed), not ad hoc changes.
- [ ] Embed all the chunks in batches and insert them. Make the insert an **upsert** so re-runs are safe.
- [ ] Add an HNSW index on the embedding column, plus regular indexes on the fields you'll filter by.
- [ ] Build a single `ingest` command that runs fetch → clean → chunk → embed → store end to end.
- [ ] Record the counts (documents, chunks) and how long ingestion takes.

**Understand**
- Why must the query and the documents be embedded with the *same* model?
- What does the vector dimension affect (storage, speed, quality)?
- What's the difference between exact and approximate nearest-neighbor search? What does HNSW trade off?
- What happens to your stored vectors if you switch embedding models?
- Why use migrations instead of editing the schema by hand?

**Docs**
- [pgvector README](https://github.com/pgvector/pgvector)
- [Sentence Transformers](https://sbert.net/) · [Voyage AI embeddings](https://docs.voyageai.com/docs/embeddings)
- [Docker Compose](https://docs.docker.com/compose/)
- [psycopg 3](https://www.psycopg.org/psycopg3/docs/)

**Done when**
- [ ] `docker compose up` starts Postgres with pgvector.
- [ ] One `ingest` command fills the database, and running it twice produces no duplicates.
- [ ] You can query the chunk count and see the embeddings stored.

---

## Phase 8: Retrieval
⏱ ~1 week

**Goal:** Given a question, reliably find the right chunks.

**Tasks**
- [ ] Write a retrieval function: embed the question → top-k similarity search → return the chunks with scores and metadata.
- [ ] Build a small CLI to type a question and see the retrieved chunks.
- [ ] Try 15–20 realistic questions (ability details, item stats, "what changed for X in patch Y") and note the failures.
- [ ] Add metadata filtering (e.g. restrict to a champion or patch when the question names one).
- [ ] Add Postgres full-text search, and combine it with vector search (hybrid) using a rank-fusion method.
- [ ] (Optional) Add a reranking step and compare the results.
- [ ] Make `k` and the retrieval mode configurable.

**Understand**
- Cosine distance vs. inner product vs. L2: which does your index use, and does it matter for normalized vectors?
- Why do exact names (e.g. "Kraken Slayer") sometimes fail with pure vector search? How does hybrid search fix that?
- What is Reciprocal Rank Fusion?
- What's the trade-off in choosing *k*?
- Precision vs. recall: which matters more for RAG retrieval, and why?

**Docs**
- [pgvector README (hybrid search section)](https://github.com/pgvector/pgvector)
- [PostgreSQL full-text search](https://www.postgresql.org/docs/current/textsearch.html)

**Done when**
- [ ] For most of your test questions, the correct chunk appears in the top-k.
- [ ] Hybrid search is implemented and you've written down whether it helped.

---

## Phase 9: Generation
⏱ ~1 week

**Goal:** Turn retrieved chunks into a grounded, cited answer. This completes the RAG loop.

**Tasks**
- [ ] Write a system prompt that defines RiftIQ's role, tone, and rules.
- [ ] Build a prompt template that inserts the retrieved chunks (with their source IDs) clearly separated from the question.
- [ ] Instruct the model to answer **only** from the provided context and to say when it doesn't know.
- [ ] Make answers cite their sources (chunk/source IDs mapped back to titles and URLs).
- [ ] Handle out-of-scope questions (not about LoL) and questions about patches you haven't ingested.
- [ ] Wire everything into an end-to-end CLI: question → retrieve → generate → answer + sources.
- [ ] Log each request's retrieved chunk IDs, token usage, and latency.
- [ ] Keep the prompts in versioned files, not buried in code.

**Understand**
- Why does telling the model to "only use the context" reduce hallucination but not eliminate it?
- Why do XML-style tags help the model separate instructions, context, and the question?
- Where in the prompt should long context go relative to the question?
- How would you detect an answer that cites a source which doesn't support it?

**Docs**
- [Prompting best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices)
- [Prompt engineering overview](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/overview)
- [Citations](https://platform.claude.com/docs/en/build-with-claude/citations)

**Done when**
- [ ] The CLI answers LoL questions correctly, with citations, for the patches you ingested.
- [ ] It says "I don't know" instead of making things up when the context lacks the answer.

🏁 **Milestone 1: working RAG in the terminal.** Commit, tag a release, and update the README.

---

## Phase 10: Evaluation
⏱ ~1 week

**Goal:** Measure quality with numbers, so you can improve it deliberately (and put the numbers on your resume).

**Tasks**
- [ ] Build a golden dataset of 30–50 questions in `evals/`, each with an expected answer and the source(s) that should be retrieved. Cover different types: ability facts, item stats, patch changes, comparisons, and unanswerable questions.
- [ ] Measure **retrieval quality**: how often the expected source appears in the top-k (hit rate / recall@k).
- [ ] Measure **answer quality**: correctness and faithfulness (grounded in the context). Use manual grading, an LLM-as-judge, or Ragas.
- [ ] Make the evaluation a single repeatable command that outputs a results table.
- [ ] Run experiments: chunking strategy, *k*, hybrid vs. vector-only, contextual headers, and prompt variations. Record the results.
- [ ] Save a results history (e.g. `evals/results/<date>.md`) so you can show improvement over time.

**Understand**
- Why evaluate retrieval and generation separately?
- What are the risks of using an LLM to grade an LLM?
- What's "faithfulness" vs. "correctness"?
- Why include questions that *should* be unanswerable?
- How would you avoid overfitting your prompts to your eval set?

**Docs**
- [Define success criteria & build evaluations](https://platform.claude.com/docs/en/test-and-evaluate/develop-tests)
- [Ragas documentation](https://docs.ragas.io/)

**Done when**
- [ ] One command runs the eval and prints the retrieval and answer metrics.
- [ ] You have a before/after comparison from at least two experiments.

---

## Phase 11: Backend API
⏱ ~1–2 weeks

**Goal:** Expose RiftIQ as a proper web API.

**Tasks**
- [ ] Create a FastAPI app in `backend/` that reuses your retrieval and generation code (don't duplicate it).
- [ ] Add a `/health` endpoint that also checks database connectivity.
- [ ] Add a `/chat` endpoint that accepts a question and **streams** the answer via Server-Sent Events, then sends the sources.
- [ ] Validate the requests (max question length, required fields) with Pydantic models.
- [ ] Add clear error handling (LLM failure, DB down, bad input) with proper status codes.
- [ ] Configure CORS for your frontend's origin (from config, not hard-coded).
- [ ] Add basic rate limiting per client, to protect your API credits.
- [ ] Add structured logging (JSON logs with a request ID, latency, and tokens).
- [ ] Write API tests (with the LLM mocked).
- [ ] Explore the auto-generated OpenAPI docs page.

**Understand**
- What is async in Python, and why does it matter for an API that waits on an LLM?
- How do SSE and WebSockets differ? Why is SSE enough here?
- What is CORS, and why does the browser enforce it?
- Why mock the LLM in tests?
- What's the difference between the pipeline (batch writes) and the API (reads only)?

**Docs**
- [FastAPI](https://fastapi.tiangolo.com/)
- [MDN: Server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)
- [pytest](https://docs.pytest.org/)

**Done when**
- [ ] You can hit `/chat` with a tool like curl or the docs page and watch the tokens stream in.
- [ ] The tests pass, and bad inputs return helpful errors.

---

## Phase 12: Frontend Chat UI
⏱ ~2 weeks

**Goal:** A clean, usable chat interface that anyone (like a recruiter) can use.

**Tasks**
- [ ] Create a Next.js app (TypeScript + Tailwind) in `frontend/`.
- [ ] Build a chat layout: message list, input box, send button.
- [ ] Consume the streaming `/chat` response and render the tokens as they arrive.
- [ ] Render the answers as Markdown, and show the cited sources as clickable links under each answer.
- [ ] Add loading, error, and empty states, plus a few example questions to click.
- [ ] Make it responsive (works on a phone).
- [ ] Add an About section and footer: what RiftIQ is, which patch the data covers, attribution for the wiki (CC BY-SA), and Riot's legal disclaimer.
- [ ] Read the API URL from environment config.
- [ ] Plan for a **static export** build (you'll host static files on S3 later), which means avoiding server-only Next.js features.

**Understand**
- What does React state do, and how does a streamed response update it?
- How do you read a streaming HTTP response in the browser?
- What does a Next.js static export give up, and why is that fine here?
- Why must you never put secret keys in frontend code?

**Docs**
- [Next.js docs](https://nextjs.org/docs) · [Static exports](https://nextjs.org/docs/app/guides/static-exports)
- [React: Learn](https://react.dev/learn)
- [Tailwind CSS](https://tailwindcss.com/docs)
- [MDN: Using readable streams](https://developer.mozilla.org/en-US/docs/Web/API/Streams_API/Using_readable_streams)

**Done when**
- [ ] You can have a streamed, cited conversation with RiftIQ in the browser, locally.
- [ ] It looks good on both desktop and mobile widths.

---

## Phase 13: Local Polish & Containerization
⏱ ~1–2 weeks

**Goal:** The entire app runs with one command, stays up to date, and is checked automatically.

**Tasks**
- [ ] Write a Dockerfile for the backend API and one for the ingestion pipeline (they can share a base).
- [ ] Extend Docker Compose so `docker compose up` runs Postgres + API + frontend together.
- [ ] Build a patch-refresh job: detect when Data Dragon has a new version, then ingest only the new or changed data.
- [ ] Decide how old patches are handled (keep with a `patch` tag vs. replace) and document it.
- [ ] Set up GitHub Actions CI to run lint (ruff/ESLint), type checks (mypy/tsc), and tests on every push and PR.
- [ ] (Optional) Run a small subset of the eval suite in CI.
- [ ] Update the README with setup instructions: prerequisites, `.env` setup, the one-command start, and how to ingest.
- [ ] Have a friend (or a fresh clone) follow the README to verify it works.

**Understand**
- What's the difference between an image and a container?
- What are multi-stage Docker builds, and why do they make images smaller?
- How do containers in the same Compose network find each other?
- Incremental vs. full re-index: what are the trade-offs?
- What should CI block a merge on?

**Docs**
- [Docker Compose](https://docs.docker.com/compose/)
- [GitHub Actions](https://docs.github.com/en/actions)
- [Ruff](https://docs.astral.sh/ruff/) · [mypy](https://mypy.readthedocs.io/)

**Done when**
- [ ] A fresh clone plus a `.env` plus one command gives you a working app.
- [ ] CI is green on GitHub.

🏁 **Milestone 2: complete app running locally.** You now understand every piece. Record a short demo video now; it's useful even before deployment.

---

# Part B: Migrate to AWS

**Mindset:** You're *moving* a working app, not building a new one. Each phase replaces one local piece with
an AWS service. The interfaces and env-var config from Part A pay off here.

> ⚠️ **Cost warning.** Some resources (the load balancer, RDS, NAT gateways) bill **per hour even when idle**.
> Set up billing alerts before creating anything, check the Billing console often, and know how to tear
> everything down. When you aren't demoing, you can destroy the stack and redeploy it from code later.

---

## Phase 14: AWS Foundations
⏱ ~1 week

**Goal:** A secure AWS account and a working mental model of AWS basics.

**Tasks**
- [ ] Create an AWS account and enable MFA on the root user. Then stop using root.
- [ ] Set up IAM Identity Center (or an admin IAM user) for daily use, with MFA.
- [ ] Create an **AWS Budgets** alert at a low monthly amount you're comfortable with.
- [ ] Install and configure the AWS CLI with a named profile, and verify your identity from the terminal.
- [ ] Choose one region for the whole project (one that supports Bedrock, if you plan to do Phase 16).
- [ ] Skim the Well-Architected Framework's pillars.

**Understand**
- What are IAM users, groups, roles, and policies? Why are roles preferred for applications?
- What is least privilege?
- What is the shared responsibility model?
- What are regions and availability zones?
- Why is using the root account day to day dangerous?

**Docs**
- [IAM security best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [IAM Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html)
- [AWS Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html)
- [AWS CLI getting started](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html)
- [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)

**Done when**
- [ ] Root has MFA and isn't used, a budget alert exists, and the CLI works with your profile.

---

## Phase 15: Storage to S3
⏱ ~3–5 days

**Goal:** Your first real migration: raw data moves from a local folder to S3.

**Tasks**
- [ ] Create an S3 bucket (by hand in the console this once, so you learn it; it moves to CDK in Phase 17).
- [ ] Configure it: block all public access, enable versioning, and consider a lifecycle rule for old patches.
- [ ] Add an S3 implementation to your storage interface.
- [ ] Switch storage backends via config only, run the fetch step, and confirm the files land in S3 with the same `<patch>/` layout.
- [ ] Confirm the rest of the pipeline works unchanged.

**Understand**
- What are buckets, keys, and prefixes? (S3 has no real folders.)
- How does your local code authenticate to AWS (credential chain)?
- What does versioning protect against?
- Why block public access on a data bucket?

**Docs**
- [Amazon S3 user guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)
- [Boto3 documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html)

**Done when**
- [ ] Flipping one config value switches between local and S3 storage, and both work.

---

## Phase 16: (Optional) LLM & Embeddings to Bedrock
⏱ ~1 week

**Goal:** Run Claude (and optionally embeddings) inside AWS, using IAM instead of API keys.

> You can skip this and keep using the Anthropic API from AWS. Doing it adds a strong "Amazon Bedrock" line to your resume and teaches IAM-based auth.

**Tasks**
- [ ] Open the Bedrock console in your region and make sure you have access to a Claude model and an embedding model (e.g. Amazon Titan Text Embeddings V2).
- [ ] Add a Bedrock implementation to your `LLMClient` and switch to it via config.
- [ ] Run your eval suite and compare the answer quality with the Anthropic API version.
- [ ] (Optional) Add a Bedrock embedder, **re-embed** everything into a separate table or column, and compare the retrieval metrics against your local model.
- [ ] Check the Bedrock quotas (requests/tokens per minute) and handle throttling with retries.
- [ ] Record your findings (quality, latency, cost, operational simplicity) in `docs/decisions.md`.

**Understand**
- How does Bedrock authentication differ from an API key?
- What is the Converse API, and why does it make swapping models easier?
- Why do you have to re-embed everything when you change embedding models?
- What are service quotas, and what happens when you hit them?

**Docs**
- [What is Amazon Bedrock?](https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html)
- [Bedrock model access](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html)
- [Bedrock Converse API](https://docs.aws.amazon.com/bedrock/latest/userguide/conversation-inference.html)
- [Claude on Amazon Bedrock (Anthropic docs)](https://platform.claude.com/docs/en/build-with-claude/claude-on-amazon-bedrock)
- [Titan text embeddings](https://docs.aws.amazon.com/bedrock/latest/userguide/titan-embedding-models.html)
- [Bedrock quotas](https://docs.aws.amazon.com/bedrock/latest/userguide/quotas.html)

**Done when**
- [ ] One config change switches the LLM (and optionally the embeddings) between Anthropic and Bedrock.
- [ ] You have eval numbers comparing the two.

---

## Phase 17: Infrastructure as Code with CDK
⏱ ~2–3 weeks · *The biggest learning curve in Part B*

**Goal:** Define all of RiftIQ's cloud infrastructure in version-controlled code.

**Tasks**
- [ ] Work through the CDK getting-started guide (and ideally the CDK Workshop). Pick TypeScript or Python for your CDK code.
- [ ] Create an `infra/` folder with a CDK app and bootstrap your account/region.
- [ ] Define the **networking**: a VPC with public and private subnets. Keep NAT costs in mind.
- [ ] Define the **database**: RDS PostgreSQL in private subnets, with credentials in Secrets Manager and a security group that only allows the app.
- [ ] Define the **storage**: import or recreate your S3 bucket in code.
- [ ] Define the **container registry**: ECR repositories for the API and pipeline images.
- [ ] Define the **compute**: an ECS cluster and a Fargate service for the API behind an Application Load Balancer, with health checks pointing at `/health`.
- [ ] Define **IAM**: task roles granting only what's needed (read the secret, read/write the bucket, invoke Bedrock if used).
- [ ] Define the **logging**: CloudWatch log groups with a retention period.
- [ ] Split the infrastructure into logical stacks (e.g. network / data / app).
- [ ] Synthesize and review the generated CloudFormation before you deploy anything.

**Understand**
- How does CDK relate to CloudFormation?
- What are constructs (L1/L2/L3), stacks, and apps?
- What are VPCs, subnets (public vs. private), route tables, and security groups?
- Why should the database never be in a public subnet?
- ECS task role vs. task execution role: what's each for?
- What is a NAT gateway, why does it cost money, and what are the alternatives?
- How do you enable the pgvector extension on RDS?

**Docs**
- [AWS CDK getting started](https://docs.aws.amazon.com/cdk/v2/guide/getting-started.html) · [CDK Workshop](https://cdkworkshop.com/)
- [CDK ecs-patterns](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ecs_patterns-readme.html) · [CDK rds](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_rds-readme.html)
- [Amazon VPC](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html)
- [PostgreSQL on Amazon RDS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_PostgreSQL.html) · [RDS PostgreSQL extensions (pgvector)](https://docs.aws.amazon.com/AmazonRDS/latest/PostgreSQLReleaseNotes/postgresql-extensions.html)
- [ECS on Fargate](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate.html) · [ECS task IAM roles](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html)
- [Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html)
- [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)

**Done when**
- [ ] `cdk synth` succeeds and you can explain every resource it creates.
- [ ] The infrastructure deploys cleanly **and tears down cleanly**. Test both.

---

## Phase 18: Deploy
⏱ ~1–2 weeks

**Goal:** RiftIQ is live on the internet.

**Tasks**
- [ ] Build and push the API and pipeline images to ECR.
- [ ] Run your database migrations and a full ingestion as a **one-off ECS task** against RDS.
- [ ] Deploy the API service and confirm the health checks pass through the load balancer.
- [ ] Add HTTPS to the API (an ACM certificate on the load balancer).
- [ ] Build the frontend as a static export and host it on S3 behind CloudFront (private bucket, CloudFront-only access), with HTTPS.
- [ ] (Optional) Register a domain and point it at CloudFront and the API with Route 53.
- [ ] Point the frontend's API URL at the deployed API, and update CORS for the production origin.
- [ ] Schedule the patch-refresh pipeline with EventBridge Scheduler running the pipeline ECS task.
- [ ] Put every new resource in CDK. No hand-made resources should remain (except, optionally, the domain).

**Understand**
- How does a request travel from a user's browser to your API and database?
- Why does CloudFront sit in front of S3?
- What does a load balancer health check do when a container fails?
- How do secrets get from Secrets Manager into your running container?
- Why run migrations as a separate task instead of on API startup?

**Docs**
- [CloudFront + S3 getting started](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/GettingStarted.SimpleDistribution.html)
- [AWS Certificate Manager](https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html)
- [EventBridge Scheduler](https://docs.aws.amazon.com/scheduler/latest/UserGuide/what-is-scheduler.html)
- [Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html)

**Done when**
- [ ] A friend on another network can open the URL and have a streamed, cited conversation.
- [ ] The scheduled refresh has run successfully at least once.

🏁 **Milestone 3: live on AWS.**

---

## Phase 19: Operate
⏱ ~1 week

**Goal:** Know when something breaks, and keep costs under control.

**Tasks**
- [ ] Confirm the logs from the API and pipeline reach CloudWatch and are searchable.
- [ ] Publish custom metrics: request latency, tokens per request, and retrieval time.
- [ ] Create a CloudWatch dashboard for these metrics.
- [ ] Create alarms for API errors, unhealthy targets, and failed pipeline runs.
- [ ] Protect against abuse: app-level rate limiting, and optionally an AWS WAF rate-based rule.
- [ ] Write a runbook (`docs/runbook.md`) covering how to deploy, roll back, re-ingest, pause (scale to zero / destroy), and restore.
- [ ] Review your actual spend in Cost Explorer and write down the monthly cost.

**Understand**
- What's the difference between logs, metrics, and alarms?
- What would you look at first if users reported slow answers?
- Which resources dominate your bill, and how could you reduce them?

**Docs**
- [Amazon CloudWatch](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html)
- [AWS WAF rate-based rules](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-rate-based.html)
- [AWS Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html)

**Done when**
- [ ] You have a dashboard, working alarms (test one on purpose), and a runbook.

---

## Phase 20: CI/CD to AWS
⏱ ~1 week

**Goal:** Merging to `main` deploys automatically, with no AWS keys stored in GitHub.

**Tasks**
- [ ] Set up GitHub's OIDC provider in AWS, plus an IAM role that only your repo's `main` branch can assume.
- [ ] Extend GitHub Actions so that on merge to `main` it builds and pushes the images to ECR, runs `cdk deploy`, builds the frontend and syncs it to S3, and invalidates the CloudFront cache.
- [ ] Keep PR checks (lint, types, tests, evals) required before merge.
- [ ] Add a manual-approval or environment protection step for production deploys (optional).
- [ ] Add a CI status badge to the README.

**Understand**
- Why are OIDC roles safer than storing access keys as GitHub secrets?
- What's the difference between continuous integration and continuous deployment?
- How would you roll back a bad deploy?

**Docs**
- [GitHub: Configuring OpenID Connect in AWS](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
- [aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials)
- [GitHub Actions](https://docs.github.com/en/actions)

**Done when**
- [ ] A merged PR shows up live on the site with no manual steps.

---

# Part C: Resume Polish

## Phase 21: Tell the Story
⏱ ~1 week

**Goal:** Make the project legible to a recruiter in 30 seconds and impressive to an engineer in 10 minutes.

**Tasks**
- [ ] Rewrite the README: a one-line pitch, live demo link, demo GIF or video, both architecture diagrams, the tech stack, eval metrics, and quickstart instructions.
- [ ] Add a **Design Decisions** section: chunking, embedding model, hybrid search, pgvector vs. a dedicated vector DB, and ECS vs. alternatives.
- [ ] Add a **Local → AWS Migration** section explaining how the interfaces made the move a config change.
- [ ] Include your eval results table and how the metrics improved over time.
- [ ] Write a short blog post or write-up (dev.to, Medium, LinkedIn, or `docs/`).
- [ ] Draft 3–4 resume bullets using real numbers. Template: *"[Action verb] [what] using [tech], resulting in [measurable outcome]."*
- [ ] Prepare interview talking points: the hardest bug, a trade-off you made, what you'd do with more time, and how you'd scale to 10× the users.
- [ ] Pin the repo on your GitHub profile.

**Done when**
- [ ] Someone unfamiliar with the project can understand what it does, how it works, and why it's impressive from the README alone.

---

## Stretch Goals
- **Conversation memory:** handle follow-up questions ("what about her W?").
- **Query rewriting:** have the LLM rewrite vague questions before retrieval.
- **Agentic tools:** let the model call the live Riot API for player or match lookups (requires a Riot developer key and following Riot's policies).
- **Multi-patch comparisons:** answer "how has Yasuo changed over the last 5 patches?"
- **A Discord bot** front end reusing the same API.
- **Bedrock Knowledge Bases comparison:** build the managed-RAG version and write up how it compares with your hand-built pipeline.
- **A serverless variant:** API on Lambda + API Gateway, compared on cost and latency.
- **Certification:** AWS Certified Cloud Practitioner or Solutions Architect – Associate complements this project well.

## Common Pitfalls
- **Committing API keys.** If it happens, *rotate the key immediately*. Deleting the commit isn't enough.
- **Skipping evaluation.** Without numbers, you can't tell whether a change helped.
- **Chunks without context.** "Damage increased from 60 to 70" means nothing without the champion, ability, and patch.
- **Mixing embedding models.** Every vector in a table must come from the same model.
- **Stale data.** Always show which patch the answers are based on.
- **Scraping carelessly.** Respect robots.txt, rate limits, and licenses; attribute the wiki.
- **Forgotten AWS resources.** Idle load balancers, databases, and NAT gateways quietly run up bills. Tear down what you're not using.
- **Building AWS before the app works.** That's why Part A comes first. Don't skip ahead.
- **Over-engineering early.** Get the simplest version working end to end, then improve.

---

## Reference Links

**Anthropic / Claude**
- [Intro to Claude](https://platform.claude.com/docs/en/intro) · [Getting started](https://platform.claude.com/docs/en/get-started)
- [Messages API](https://platform.claude.com/docs/en/api/messages) · [Client SDKs](https://platform.claude.com/docs/en/api/client-sdks) · [Python SDK](https://github.com/anthropics/anthropic-sdk-python)
- [Models overview](https://platform.claude.com/docs/en/about-claude/models/overview) · [Pricing](https://platform.claude.com/docs/en/about-claude/pricing)
- [Streaming](https://platform.claude.com/docs/en/build-with-claude/streaming) · [Token counting](https://platform.claude.com/docs/en/build-with-claude/token-counting) · [Citations](https://platform.claude.com/docs/en/build-with-claude/citations) · [Embeddings](https://platform.claude.com/docs/en/build-with-claude/embeddings)
- [Prompting best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices) · [Building evals](https://platform.claude.com/docs/en/test-and-evaluate/develop-tests)
- [Introducing Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval)
- [Claude on Amazon Bedrock](https://platform.claude.com/docs/en/build-with-claude/claude-on-amazon-bedrock)

**RAG & Search**
- [RAG paper (Lewis et al., 2020)](https://arxiv.org/abs/2005.11401)
- [pgvector](https://github.com/pgvector/pgvector) · [PostgreSQL full-text search](https://www.postgresql.org/docs/current/textsearch.html)
- [Sentence Transformers](https://sbert.net/) · [Voyage AI](https://docs.voyageai.com/docs/embeddings)
- [Ragas](https://docs.ragas.io/)

**League of Legends Data**
- [Riot Developer Docs: LoL / Data Dragon](https://developer.riotgames.com/docs/lol) · [Riot Developer Policies](https://developer.riotgames.com/policies/general)
- [CommunityDragon](https://www.communitydragon.org/)
- [Patch notes](https://www.leagueoflegends.com/en-us/news/tags/patch-notes/) · [LoL Wiki](https://wiki.leagueoflegends.com/en-us/) · [Wiki license](https://wiki.leagueoflegends.com/en-us/League_of_Legends_Wiki:Copyrights)

**Python & Backend**
- [uv](https://docs.astral.sh/uv/) · [FastAPI](https://fastapi.tiangolo.com/) · [Pydantic](https://docs.pydantic.dev/latest/) · [HTTPX](https://www.python-httpx.org/) · [Beautiful Soup](https://www.crummy.com/software/BeautifulSoup/bs4/doc/) · [psycopg 3](https://www.psycopg.org/psycopg3/docs/)
- [pytest](https://docs.pytest.org/) · [Ruff](https://docs.astral.sh/ruff/) · [mypy](https://mypy.readthedocs.io/)

**Frontend**
- [Next.js](https://nextjs.org/docs) · [Static exports](https://nextjs.org/docs/app/guides/static-exports) · [React](https://react.dev/learn) · [Tailwind CSS](https://tailwindcss.com/docs)
- [MDN: Server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events) · [MDN: Readable streams](https://developer.mozilla.org/en-US/docs/Web/API/Streams_API/Using_readable_streams)

**DevOps**
- [Pro Git](https://git-scm.com/book/en/v2) · [Docker Compose](https://docs.docker.com/compose/) · [GitHub Actions](https://docs.github.com/en/actions)

**AWS**
- Foundations: [IAM best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html) · [IAM Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html) · [Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html) · [CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html) · [Well-Architected](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)
- Storage & data: [S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html) · [RDS PostgreSQL](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_PostgreSQL.html) · [RDS extensions](https://docs.aws.amazon.com/AmazonRDS/latest/PostgreSQLReleaseNotes/postgresql-extensions.html) · [Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html) · [Boto3](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html)
- AI: [Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html) · [Model access](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html) · [Converse API](https://docs.aws.amazon.com/bedrock/latest/userguide/conversation-inference.html) · [Titan embeddings](https://docs.aws.amazon.com/bedrock/latest/userguide/titan-embedding-models.html) · [Quotas](https://docs.aws.amazon.com/bedrock/latest/userguide/quotas.html) · [Knowledge Bases](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html)
- Compute & network: [VPC](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html) · [ECS Fargate](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate.html) · [Task IAM roles](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html) · [ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html) · [CloudFront + S3](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/GettingStarted.SimpleDistribution.html) · [ACM](https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html)
- Ops: [EventBridge Scheduler](https://docs.aws.amazon.com/scheduler/latest/UserGuide/what-is-scheduler.html) · [CloudWatch](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html) · [WAF rate-based rules](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-rate-based.html)
- IaC & CI/CD: [CDK getting started](https://docs.aws.amazon.com/cdk/v2/guide/getting-started.html) · [CDK Workshop](https://cdkworkshop.com/) · [CDK ecs-patterns](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ecs_patterns-readme.html) · [CDK rds](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_rds-readme.html) · [GitHub OIDC for AWS](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services) · [configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials)
