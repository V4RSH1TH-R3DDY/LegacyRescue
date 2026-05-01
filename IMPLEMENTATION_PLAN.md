# Legacy Rescue AI — Detailed Implementation Plan
### "Transform undocumented legacy systems into modern, production-ready architectures in hours instead of months."

---

## Overview

| Attribute | Detail |
|---|---|
| **Total Phases** | 6 |
| **Estimated Duration** | 16–20 weeks (full build) / 4–5 days (hackathon MVP) |
| **Core Tech** | IBM watsonx (Bob), React frontend, Node.js/Python backend, Docker/K8s |
| **Primary Personas** | Enterprise architects, senior engineers, DevOps leads |

---

## Phase 0 — Foundation & Project Setup
**Duration:** 2–3 days (hackathon) / 1 week (full build)  
**Goal:** Establish repo structure, tooling, team roles, and a working skeleton before any feature code is written.

---

### Subphase 0.1 — Repository & Monorepo Setup
- Initialize a monorepo (Nx or Turborepo) with three top-level workspaces:
  - `/apps/frontend` — React dashboard
  - `/apps/backend` — API server (Node.js + FastAPI sidecar for AI calls)
  - `/apps/worker` — async job processor for repo analysis
- Configure ESLint, Prettier, Husky pre-commit hooks
- Set up `.env` templates for watsonx API keys, DB URIs, and storage credentials
- Create `docker-compose.yml` for local development (PostgreSQL, Redis, MinIO for file storage)
- Initialize CI/CD pipeline (GitHub Actions): lint → test → build → Docker push

### Subphase 0.2 — Architecture Decision Records (ADRs)
- ADR-001: Monorepo vs polyrepo → choose monorepo for hackathon velocity
- ADR-002: Synchronous vs async analysis → choose async with job queue (Bull/BullMQ on Redis)
- ADR-003: Visualization library → choose ReactFlow for architecture graphs
- ADR-004: LLM routing → watsonx `ibm/granite-34b-code-instruct` as primary, fallback to `ibm/granite-8b-code-instruct` for speed
- ADR-005: Storage for uploaded repos → MinIO (S3-compatible) locally; IBM Cloud Object Storage in production

### Subphase 0.3 — IBM watsonx Integration Spike
- Create a throwaway script that calls the watsonx.ai `/v1/generate` endpoint with a raw Java snippet
- Validate authentication, latency, and token limits
- Identify chunking strategy for large files (max tokens per call)
- Document prompt templates that return consistent JSON outputs
- Validate watsonx Orchestrate access if available for workflow automation

### Subphase 0.4 — Sample Legacy Corpus Assembly
- Curate 3 sample legacy repositories for demo use:
  - **BankCore** — Java 6 monolith with JDBC, interest calculation, account management
  - **ClaimsProcessor** — Java 7 insurance claims engine with downstream payment calls
  - **InventoryLedger** — COBOL batch processor with file-based I/O
- These become the fixed demo artifacts used in every presentation/test

---

## Phase 1 — File Ingestion & Repository Parsing Engine
**Duration:** 3–4 days (hackathon) / 2 weeks (full build)  
**Goal:** Accept any uploaded codebase and produce a normalized, queryable internal representation.

---

### Subphase 1.1 — Upload Service
- Build a multipart file upload endpoint (`POST /api/v1/repos/upload`)
  - Accept `.zip`, `.tar.gz`, or direct GitHub repo URL via OAuth token
  - Validate file size limits (50 MB for hackathon; 2 GB for production)
  - Store raw archive to MinIO/COS under `uploads/{jobId}/raw`
- Return a `jobId` immediately (202 Accepted) — all analysis is async
- Emit a `repo.uploaded` event to the job queue

### Subphase 1.2 — Extraction & File Tree Normalization
- Worker picks up `repo.uploaded` event
- Decompress archive into a temp workspace directory
- Walk the file tree and produce a **Normalized File Manifest (NFM)**:
  ```json
  {
    "jobId": "abc123",
    "files": [
      { "path": "src/main/java/claims/ClaimsEngine.java", "language": "java", "sizeBytes": 4210, "lines": 180 },
      { "path": "scripts/deploy.sh", "language": "bash", "sizeBytes": 890, "lines": 42 }
    ],
    "stats": { "totalFiles": 143, "totalLines": 28400, "languages": ["java", "bash", "sql"] }
  }
  ```
- Store NFM to PostgreSQL `jobs` table, update job status to `PARSED`

### Subphase 1.3 — Language Detection & Classification
- Run language detection using `linguist` (GitHub's library, via a Python subprocess) or `enry`
- Tag each file with:
  - `language` (java, cobol, rpg, bash, sql, xml, yaml)
  - `role` classification (entry_point, config, migration, test, build_script, data_model)
- Rule-based role classification:
  - Files with `main()` or `public static void main` → `entry_point`
  - Files matching `*Dao.java`, `*Repository.java`, `*.sql` → `data_model`
  - Files matching `Dockerfile`, `pom.xml`, `build.gradle` → `build_script`
- Persist classifications back to NFM

### Subphase 1.4 — Static AST Parsing (No LLM Required)
- For Java files: use `tree-sitter-java` (via Python `tree_sitter` bindings) to extract:
  - Class names, method signatures, field declarations
  - Import statements (external dependencies)
  - Method call graph within the same file
- For SQL files: use `sqlparse` to extract table names, JOIN targets, stored procedure names
- For Bash/Shell: regex-based extraction of function definitions and called binaries
- For COBOL: regex-based DIVISION/SECTION/PARAGRAPH extraction
- Store AST summaries per file in PostgreSQL `file_analysis` table

---

## Phase 2 — Module 1: Code Understanding Engine
**Duration:** 3–4 days (hackathon) / 2 weeks (full build)  
**Goal:** Use watsonx to read parsed code and generate a human-readable system summary.

---

### Subphase 2.1 — Chunking Strategy for Large Repos
- Problem: a 28,000-line codebase cannot be passed to an LLM in one call
- Strategy: **hierarchical summarization**
  1. **File-level summaries** — summarize each file individually (one LLM call per file, batched)
  2. **Module-level summaries** — cluster files by directory/package, summarize clusters
  3. **System-level summary** — synthesize all module summaries into one final description
- Implement a `chunker.py` service that:
  - Respects max token limits per call (e.g., 4096 tokens for Granite 8B)
  - Adds surrounding context (imports, class name) when slicing large files

### Subphase 2.2 — File Summary Prompt Engineering
- Design a strict prompt template:
  ```
  You are a senior software architect. Analyze the following {language} source file.
  
  File: {file_path}
  Code:
  {code_content}
  
  Respond ONLY in valid JSON with this exact schema:
  {
    "purpose": "one sentence describing what this file does",
    "keyFunctions": ["function1 does X", "function2 does Y"],
    "externalDependencies": ["List of external systems, APIs, or DBs referenced"],
    "businessConcepts": ["List of domain concepts like 'insurance claim', 'payment', 'eligibility']"
  }
  ```
- Validate all responses against JSON schema; retry up to 3 times on malformed output
- Store results in `file_analysis.llm_summary` (JSONB column)

### Subphase 2.3 — Cross-File Relationship Resolution
- After all file summaries complete, build a **cross-reference map**:
  - Class A imports Class B → directed edge A → B
  - Method in File X calls method in File Y (resolved by import + method name matching)
- Use AST data from Phase 1.4 for this — no LLM needed here
- Store edges in a `call_graph` table: `{ source_file, target_file, edge_type: import|call|inherit|implement }`

### Subphase 2.4 — System-Level Natural Language Summary
- Collect all `businessConcepts` across files, deduplicate and rank by frequency
- Feed the top module summaries + ranked concepts to watsonx with a synthesis prompt:
  ```
  Given these module summaries: {module_summaries}
  Generate a single paragraph (3–5 sentences) describing what this entire system does,
  who uses it, and what downstream systems it connects to.
  ```
- Output example:
  > "This system is a Java-based insurance claims processing engine. It validates customer eligibility against a policy database, calculates claim amounts using tiered business rules, and dispatches approved payments to downstream banking APIs via JDBC. The system appears to serve internal claims adjusters and integrates with at least two external financial institutions."
- Store as `jobs.system_summary`

### Subphase 2.5 — Entry Point & Boundary Detection
- Identify **entry points** (files/methods that receive external input):
  - HTTP controllers (`@Controller`, `@RestController`, `HttpServlet`)
  - Batch job launchers (`main()` methods, cron-triggered classes)
  - Message queue consumers (`@KafkaListener`, JMS `MessageListener`)
- Identify **system boundaries** (external system calls):
  - JDBC calls to databases
  - HTTP client calls (`RestTemplate`, `HttpClient`, `URLConnection`)
  - File I/O operations (`FileReader`, `BufferedReader`)
- Tag these in the NFM and surface them prominently in the UI

---

## Phase 3 — Module 2: Architecture Reconstruction & Visualization
**Duration:** 2–3 days (hackathon) / 2 weeks (full build)  
**Goal:** Generate an interactive, zoomable architecture diagram from the parsed code graph.

---

### Subphase 3.1 — Graph Data Model Design
- Define a `ArchitectureGraph` JSON schema:
  ```json
  {
    "nodes": [
      { "id": "auth-service", "label": "Auth Module", "type": "service", "files": ["AuthController.java"] },
      { "id": "claims-engine", "label": "Claims Engine", "type": "service", "files": ["ClaimsEngine.java", "EligibilityValidator.java"] },
      { "id": "payment-db", "label": "Payment DB", "type": "database" }
    ],
    "edges": [
      { "from": "auth-service", "to": "claims-engine", "label": "validates then routes", "type": "sync" },
      { "from": "claims-engine", "to": "payment-db", "label": "JDBC write", "type": "database" }
    ]
  }
  ```
- Node types: `frontend`, `service`, `database`, `queue`, `external_api`, `batch_job`, `file_system`
- Edge types: `sync`, `async`, `database`, `file`, `event`

### Subphase 3.2 — LLM-Assisted Cluster Detection
- Problem: 143 Java files don't map to "services" automatically
- Strategy: use watsonx to group related files into logical clusters
  - Input: list of files with their `purpose` summaries
  - Prompt: "Group these files into 4–8 logical services based on their responsibilities. Each service should have a name and list of files."
  - Output: JSON array of clusters
- Fallback heuristic: group by Java package (`com.company.claims`, `com.company.auth`)

### Subphase 3.3 — ReactFlow Visualization Component
- Build `<ArchitectureGraph />` React component using **ReactFlow**:
  - Nodes rendered with icons based on type (database cylinder, service rectangle, queue icon)
  - Color coding: green for services, blue for databases, orange for external APIs, grey for legacy modules
  - Directed edges with animated dashes for async, solid lines for sync
  - Click a node → slide-out panel shows files within that node, with their summaries
  - Zoom/pan, mini-map, auto-layout using Dagre layout algorithm
- Add a "highlight path" feature: click any two nodes to highlight the call path between them

### Subphase 3.4 — Dependency Graph View (File-Level Drill-Down)
- Secondary view: a collapsible tree showing file-level dependencies
- Toggle between "Service View" (clustered) and "File View" (raw call graph)
- Show circular dependency warnings highlighted in red (cycles in the call graph)
- Export graph as PNG or SVG for use in documentation

### Subphase 3.5 — Call Flow Sequence Diagram Generation
- For each entry point detected in Phase 2.5, generate a Mermaid.js sequence diagram:
  ```
  sequenceDiagram
    Client->>AuthController: POST /login
    AuthController->>UserRepository: findByUsername()
    UserRepository->>Database: SELECT * FROM users
    AuthController->>ClaimsEngine: route(claimRequest)
    ClaimsEngine->>PaymentEngine: dispatchPayment()
  ```
- Render Mermaid diagrams inline in the UI using `mermaid.js`
- Allow user to select which entry point to trace

---

## Phase 4 — Module 3 & 4: Business Logic Translation + Modernization Recommendations
**Duration:** 2–3 days (hackathon) / 3 weeks (full build)  
**Goal:** Explain what the code *means* in plain English, then prescribe a concrete modernization path.

---

### Subphase 4.1 — Business Rule Extraction Engine
- Identify "decision points" in the code using AST analysis:
  - `if/else` chains
  - `switch` statements
  - Ternary expressions
  - Comparison operators on domain fields (balance, rate, type, status)
- For each decision cluster, invoke watsonx with a targeted prompt:
  ```
  The following code is from a {language} legacy system in the domain of {businessDomain}.
  
  Code:
  {decision_block}
  
  Explain what business rule this code encodes in plain English. 
  Be specific about thresholds, conditions, and outcomes.
  Respond in one to three sentences. No code. No jargon.
  ```
- Store each rule as a `BusinessRule` record:
  ```json
  { "id": "rule_042", "file": "ClaimsEngine.java", "line": 84, "explanation": "Premium customers with balances above 10,000 receive a 12% interest rate.", "code": "if(cType.equals('A') && bal > 10000){ rate = 0.12; }" }
  ```

### Subphase 4.2 — Business Logic Document Generation
- Aggregate all `BusinessRule` records into a structured **Business Logic Document**
- Group by: module → class → rule
- Auto-generate a navigable HTML/Markdown document the team can export
- Example output structure:
  ```
  Claims Engine
  ├── Eligibility Rules
  │   ├── Rule 001: Active policy holders only...
  │   └── Rule 002: Claims submitted within 90 days...
  ├── Pricing Rules
  │   ├── Rule 042: Premium customers with balance > 10k...
  │   └── Rule 043: Standard rate for others is 8%...
  └── Rejection Rules
      └── Rule 019: Auto-reject claims with missing SSN...
  ```

### Subphase 4.3 — Technical Debt Scoring
- For each file, compute a **Technical Debt Score (TDS)** using static heuristics:
  - Cyclomatic complexity (number of branches)
  - Method length (methods > 50 lines flagged)
  - God class detection (classes > 500 lines with > 20 methods)
  - Magic numbers/strings (hardcoded values like `0.12`, `"A"`)
  - Commented-out code blocks
  - TODO/FIXME/HACK comment density
- Normalize to a 0–100 score, color-coded red/yellow/green
- Display as a heat-map overlay on the file tree

### Subphase 4.4 — Modernization Assessment Engine
- Feed the system summary, tech debt scores, and architecture graph to watsonx with a structured prompt:
  ```
  You are a principal enterprise architect. Given the following legacy system analysis:
  
  System Summary: {system_summary}
  Tech Stack: {detected_languages_and_frameworks}
  Architecture Pattern: {detected_pattern: monolith|layered|batch}
  Top Debt Areas: {top_5_debt_files}
  
  Produce a modernization recommendation in JSON with this schema:
  {
    "currentState": "...",
    "recommendedArchitecture": "...",
    "migrationStrategy": "strangler_fig | big_bang | parallel_run",
    "prioritizedSteps": [
      { "step": 1, "action": "...", "rationale": "...", "effort": "low|medium|high", "risk": "low|medium|high" }
    ],
    "recommendedStack": { "containerization": "Docker", "orchestration": "Kubernetes", ... },
    "estimatedEffort": "X engineer-months"
  }
  ```

### Subphase 4.5 — Migration Strategy Explainer UI
- Build a "Modernization Roadmap" panel in the UI:
  - Visual timeline showing migration steps as swim lanes
  - Each step is clickable for detail: rationale, effort, risk badge
  - "Why this matters" tooltip explaining each recommendation in non-technical terms
  - Side-by-side: **Current Architecture** vs **Target Architecture** graph comparison

---

## Phase 5 — Module 5: Auto Migration Code Generator
**Duration:** 3–4 days (hackathon — generate 1 service) / 4 weeks (full build)  
**Goal:** Take a single legacy class/file and generate a fully production-ready modern microservice.

---

### Subphase 5.1 — Migration Target Selection UI
- After the modernization plan is shown, present a "Migrate a Service" panel
- List all detected modules/clusters, each with:
  - File count
  - Tech debt score
  - Bob's suggested migration target (Java 21 service, Rust microservice, Python FastAPI)
- User selects one module to migrate
- Optionally: specify target language (Java 21, Kotlin, Rust, Python, Go)

### Subphase 5.2 — Migration Prompt Architecture
- A single legacy class migration is broken into 6 sequential LLM calls (pipeline):

  | Step | Input | Output |
  |---|---|---|
  | 1. Domain Model | Legacy class fields | Modern data classes / records |
  | 2. Business Logic | Core method bodies | Clean, well-named methods |
  | 3. API Layer | Entry points detected | REST endpoint stubs with OpenAPI annotations |
  | 4. Data Layer | DB calls (JDBC) | Repository pattern (Spring Data / SQLAlchemy) |
  | 5. Tests | Generated service code | Unit + integration test skeletons |
  | 6. Containerization | Full service code | Dockerfile + docker-compose snippet |

- Each step's output feeds into the next step as context (chained prompts)
- Each prompt enforces: "Respond ONLY with valid {language} code. No explanation. No markdown fences."

### Subphase 5.3 — Java 21 Migration Template
- For Java → Java 21 migrations, enforce modern idioms:
  - Replace raw JDBC with Spring Data JPA repositories
  - Replace mutable state with Java Records for data classes
  - Replace legacy `if/else` chains with pattern matching (`switch` expressions)
  - Replace `HttpServlet` with Spring Boot `@RestController`
  - Add structured logging (SLF4J + Logback)
  - Add `@Valid` bean validation on request bodies
- Generated file structure:
  ```
  migrated-claims-service/
  ├── src/main/java/com/legacyrescue/claims/
  │   ├── ClaimsServiceApplication.java
  │   ├── controller/ClaimsController.java
  │   ├── service/ClaimsService.java
  │   ├── repository/ClaimsRepository.java
  │   ├── model/Claim.java
  │   └── config/AppConfig.java
  ├── src/test/java/...
  │   └── ClaimsServiceTest.java
  ├── Dockerfile
  ├── docker-compose.yml
  └── pom.xml
  ```

### Subphase 5.4 — Rust Microservice Migration Template (Stretch Goal)
- For high-performance candidates (file parsers, data processors), offer Rust target:
  - `Axum` for HTTP routing
  - `SQLx` for async DB access
  - `Serde` for JSON serialization
  - `Tokio` async runtime
  - Generated with full `Cargo.toml`, error types, and integration test scaffolding

### Subphase 5.5 — Migration Diff Viewer
- Display a side-by-side diff view in the UI:
  - **Left panel:** original legacy code (syntax-highlighted, read-only)
  - **Right panel:** generated modern code (syntax-highlighted, editable)
  - Line-level annotations: hover over a generated line to see "This replaces line 84 of ClaimsEngine.java"
  - "Copy to clipboard" and "Download as ZIP" buttons for the full migrated service

### Subphase 5.6 — Validation & Confidence Scoring
- Run basic static validation on generated code:
  - Java: attempt `javac` compilation check (syntax only)
  - All: check for placeholder text (`TODO`, `FIXME`, `YOUR_VALUE_HERE`)
  - All: check for hallucinated imports (libraries not referenced in the original)
- Assign a **Migration Confidence Score** (0–100%):
  - 100%: compiles, no placeholders, all original business rules present
  - 70–99%: compiles, minor placeholders remain
  - < 70%: show warning banner, flag for human review
- Display confidence prominently in the UI with a breakdown explanation

---

## Phase 6 — Frontend Dashboard, UX Polish & Demo Prep
**Duration:** 2–3 days (hackathon) / 2 weeks (full build)  
**Goal:** Deliver a polished, demo-ready UI that tells the story end-to-end.

---

### Subphase 6.1 — Dashboard Layout & Navigation
- Build a multi-panel dashboard with a persistent left sidebar:
  ```
  [Bob Logo]
  ─────────────────
  📁 Repository
  🗺️  Architecture
  🧠 Business Logic
  🔧 Modernize
  ✨ Migrate
  ─────────────────
  [Job Status: ✅]
  ```
- Each sidebar item corresponds to one Module (1–5)
- Top bar shows: repo name, analysis timestamp, overall health score
- Add a real-time progress bar during analysis (driven by WebSocket events from the job worker)

### Subphase 6.2 — Repository Overview Screen (Module 1 UI)
- Language breakdown donut chart
- Key metrics cards: total files, total lines, languages detected, entry points found
- System summary paragraph (the natural language description) in a highlighted box
- File tree with search, filter by language, filter by role
- Click any file → show its LLM summary + raw code side by side

### Subphase 6.3 — Architecture Screen (Module 2 UI)
- Full-screen ReactFlow graph (primary visualization)
- Toolbar: toggle Service View / File View, highlight dependencies, zoom controls
- Right-click a node → context menu: "Explain this module", "Show call flows", "Migrate this"
- Call flow sequence diagram below the graph, with entry point selector dropdown
- Export buttons: PNG, SVG, JSON (raw graph data)

### Subphase 6.4 — Business Logic Screen (Module 3 UI)
- Left: collapsible rule tree (module → class → rule)
- Right: rule detail card showing:
  - Plain English explanation
  - Original code snippet (syntax-highlighted)
  - File path + line number
  - "Potentially risky" flag if rule involves money, compliance, or auth
- Search bar: "Find rules about 'payment'" or "Find rules about 'eligibility'"
- Export as PDF business requirements document

### Subphase 6.5 — Modernization Screen (Module 4 UI)
- Current vs Target architecture comparison (two ReactFlow graphs, side by side)
- Migration strategy badge (Strangler Fig / Big Bang / Parallel Run) with tooltip explanation
- Prioritized step cards: numbered, color-coded by risk (green/yellow/red)
- Recommended stack chips: Docker, Kubernetes, REST, Event-driven — each with a "Why?" tooltip
- Estimated effort display ("~8 engineer-months") with caveat

### Subphase 6.6 — Migration Screen (Module 5 UI)
- Service selector (list of detected modules with debt scores)
- Target language selector (Java 21, Kotlin, Rust, Python)
- "Generate Migration" button → triggers async job, shows progress
- Side-by-side diff viewer (legacy vs generated)
- Confidence score badge
- "Download Service ZIP" button

### Subphase 6.7 — Bob Chat Interface (Copilot Overlay)
- A floating chat bubble in the bottom-right corner of every screen
- Bob is always available as a chat assistant:
  - "Explain the ClaimsEngine to me like I'm a new engineer"
  - "What would break if I removed the EligibilityValidator?"
  - "Generate a migration plan for just the auth module"
- Backed by a stateful conversation context that includes the repo's full analysis as system context
- Streamed responses (SSE/WebSocket) for real-time typewriter effect

### Subphase 6.8 — Demo Mode & Hackathon Polish
- Add a "Load Demo Repo" button on the upload screen (loads BankCore or ClaimsProcessor instantly — no upload needed)
- Pre-warm analysis results so demo shows results in < 2 seconds
- Record a 90-second screen-capture walkthrough video as fallback
- Add subtle animations: graph nodes fade in one-by-one during analysis, progress steps check off in real time
- Add IBM / watsonx branding where appropriate (logo, color palette, "Powered by watsonx" badge)

---

## Cross-Cutting Concerns

### Error Handling & Resilience
- Every LLM call has a 3-retry policy with exponential backoff
- If watsonx returns malformed JSON, attempt repair with a follow-up "fix this JSON" call
- If a file cannot be parsed, mark it as `SKIPPED` and continue — never block the whole job
- All async jobs have a timeout (10 minutes for full analysis); show a partial results screen on timeout

### Security
- API keys stored in server-side environment variables only — never exposed to frontend
- Uploaded repos are stored in isolated per-job MinIO buckets with TTL (auto-delete after 24h)
- No code is logged or stored beyond the job's analysis artifacts
- Add a "Delete my data" button that purges all job artifacts immediately

### Scalability (Post-Hackathon)
- Job worker is stateless and horizontally scalable (N workers on Kubernetes)
- Introduce a priority queue: small repos (<500 files) get fast-lane processing
- Cache LLM responses by file content hash — identical files across repos are never re-analyzed

---

## Hackathon-Specific Delivery Checklist

| Item | Status |
|---|---|
| Upload + parse a real Java repo | Must have |
| Architecture graph renders correctly | Must have |
| System summary generated by Bob | Must have |
| At least 3 business rules explained | Must have |
| Modernization recommendations shown | Must have |
| 1 migrated service generated | Must have |
| Bob chat overlay works | Should have |
| Diff viewer for migration | Should have |
| COBOL / RPG support | Nice to have |
| Rust migration target | Nice to have |
| watsonx Orchestrate workflow | Nice to have |

---

## Suggested Team Split (4-Person Hackathon Team)

| Person | Owns |
|---|---|
| **Engineer 1 (Backend/AI)** | Phases 1 + 2: ingestion, AST parsing, watsonx summarization |
| **Engineer 2 (Backend/AI)** | Phases 4 + 5: business logic, modernization, code generation |
| **Engineer 3 (Frontend)** | Phase 6: all UI screens, ReactFlow, diff viewer |
| **Engineer 4 (Fullstack/Infra)** | Phase 3: architecture graph data model + Phase 0: infra, CI, demo prep |

---

*Plan version 1.0 — Legacy Rescue AI*
