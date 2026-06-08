# Aria — Complete Product Specification
### *Napkin AI for System Design & Technical Writing*

> **Tagline:** You think in text. Aria thinks in diagrams.

---

## Table of Contents

1. [Vision & Philosophy](#1-vision--philosophy)
2. [The Problem — In Depth](#2-the-problem--in-depth)
3. [Product Overview](#3-product-overview)
4. [User Personas](#4-user-personas)
5. [Core Features — Full Detail](#5-core-features--full-detail)
6. [Diagram Types & Intelligence](#6-diagram-types--intelligence)
7. [User Experience & Flows](#7-user-experience--flows)
8. [Technical Architecture](#8-technical-architecture)
9. [AI Pipeline Design](#9-ai-pipeline-design)
10. [Data Model](#10-data-model)
11. [API Design](#11-api-design)
12. [Evaluation & Quality](#12-evaluation--quality)
13. [MVP Scope & Phased Roadmap](#13-mvp-scope--phased-roadmap)
14. [Monetization](#14-monetization)
15. [Launch Strategy](#15-launch-strategy)
16. [Success Metrics](#16-success-metrics)
17. [Tech Stack](#17-tech-stack)

---

## 1. Vision & Philosophy

### The Core Insight

Napkin AI proved a thesis: **if you understand what text means, you can skip the blank canvas entirely.** Users paste text, click one button, and get a visual. No prompting. No templates. No design skills. The magic is in the understanding layer.

But Napkin AI was built for *business people* writing *business prose*. It excels at org charts, process flows, and marketing frameworks. It completely fails for technical writers, engineers, and architects — who write in bullet points, pseudocode, markdown, and YAML, and whose mental models require sequence diagrams, ER diagrams, C4 architecture models, API flows, state machines, and network topology diagrams.

**Aria is what Napkin AI would be if it was built by engineers, for engineers.**

The surface-level product is simple: paste technical text, get a technical diagram. But underneath, Aria does something genuinely difficult — it understands the *semantic structure* of technical writing. It knows that "User sends request → Gateway validates token → Service queries DB → Response returned" is a sequence diagram, not a flowchart. It knows that "users table has many orders; orders belong to products" is an ER diagram. It knows that "the Load Balancer distributes to three EC2 instances behind a private subnet" is a network/infrastructure diagram.

That semantic understanding is the product.

### Design Principles

**Principle 1: No prompting.** Napkin AI's breakthrough UX was zero-prompt generation. Aria carries this forward. You write technical content the way you normally write it. Aria reads it. The diagram appears. No "generate a sequence diagram showing..." — just paste and click.

**Principle 2: The diagram serves the writing.** Aria is a writing tool, not a design tool. Diagrams live alongside text, not on a separate canvas. The mental model: you write a technical document, and diagrams materialize inline wherever they're useful — like illustrations in a textbook.

**Principle 3: Editable output is mandatory.** Generated diagrams are starting points, not final outputs. Every element must be editable directly in the UI, with changes reflected immediately. Generated Mermaid/SVG is also exposed for users who want to edit at the source level.

**Principle 4: Every format engineers actually use.** ER diagrams. Sequence diagrams. System architecture (C4 model). State machines. Network topology. API flow. Decision trees. Class diagrams. Deployment diagrams. If it appears in a technical design doc, Aria should render it.

**Principle 5: Multimodal input.** Text is the primary input but not the only one. Voice narration, whiteboard photos, and existing diagram screenshots are all valid starting points. Aria meets the engineer where their thoughts live.

---

## 2. The Problem — In Depth

### The Technical Documentation Crisis

There is a widely acknowledged, rarely solved problem in software engineering: technical documentation is either nonexistent or terrible. The main reason is friction. Creating good diagrams requires:

1. **Choosing a tool** (draw.io, Lucidchart, Mermaid, PlantUML, Excalidraw, Figma...)
2. **Learning the syntax or interface** (each tool has steep learning curves)
3. **Building from a blank canvas** (the most creative-block-inducing experience)
4. **Keeping diagrams in sync with text** (they live in different files/tools)
5. **Exporting and embedding** (format wars: PNG vs SVG vs embed code)

The result: engineers write text documentation and either skip diagrams entirely, paste in screenshots of quickly-drawn Excalidraw sketches that don't match the text, or spend 2 hours making a "proper" diagram in draw.io when they should have spent 20 minutes.

### The Specific Gaps Napkin AI Doesn't Fill

**Napkin AI's sweet spot:**
- Business process flows
- Comparison tables
- Marketing frameworks (SWOT, funnels)
- Org charts
- Simple numbered step sequences

**Napkin AI's failure modes for technical users:**
- Generates a generic flowchart when an engineer writes "User → API Gateway → Lambda → DynamoDB" instead of an architecture diagram with proper service icons
- Can't render ER diagrams with cardinality notation
- Doesn't understand HTTP method notation in API flows
- Treats a state machine description as a flowchart
- No concept of services, databases, queues, load balancers as visual primitives
- No Mermaid export
- No way to paste pseudocode and get a meaningful diagram

### Pain in Real Engineering Scenarios

**Scenario A — Architecture Review:** An engineer writes a 3-paragraph description of a new microservice architecture for a team design doc. They need a diagram for the review meeting. Currently: open draw.io, spend 45 minutes placing boxes and arrows. With Aria: paste the three paragraphs, click once, diagram ready in 8 seconds.

**Scenario B — API Documentation:** A backend engineer is writing API documentation for a new endpoint. The request-response flow involves 4 services and 3 async steps. Currently: draw a sequence diagram in Mermaid, debug the syntax for 20 minutes. With Aria: describe the flow in plain English, Aria generates correct Mermaid syntax and renders it.

**Scenario C — Database Design:** A team is designing a new schema. The PM has written a document describing the entities and their relationships. Currently: the engineer manually translates that to an ER diagram. With Aria: paste the PM's description, get a proper ER diagram with cardinality notation that the team can discuss and refine.

**Scenario D — Whiteboard to Document:** After a 1-hour whiteboard session, the engineer has a photo of a complex architecture diagram on a whiteboard. Currently: manually redraw in a tool. With Aria: upload the photo, AI redraw it as clean SVG, exportable and editable.

---

## 3. Product Overview

### What Aria Is

Aria is a technical diagramming tool powered by AI that generates the right diagram type from technical text, with zero prompting. It is built into a document editor — diagrams live alongside text, not in a separate application.

The core interaction is a single spark button (borrowed from Napkin AI's excellent UX) that appears next to any technical paragraph, list, or code block. Click it → Aria analyzes the text, selects the appropriate diagram type, and renders the diagram inline. The user can then edit, regenerate with a different type, or export.

### The 3 Things Aria Does That Don't Exist Together Anywhere

**1. Semantic diagram type detection for technical content.** Aria understands the difference between a service architecture, a data model, a process flow, and a state machine — and selects the right visualization without being told.

**2. Technical-primitive-aware rendering.** Aria knows what a load balancer, a message queue, a CDN, a database, a cache, an API gateway, and a Lambda function look like visually. It uses proper technical iconography (AWS icons, generic service icons) not generic shapes.

**3. Text + diagram parity.** When you edit the text, the diagram can update. When you edit the diagram, the underlying Mermaid/code updates. True bidirectional sync between prose and visual — something no tool has solved.

---

## 4. User Personas

### Persona 1: Riya — The Backend Engineer Writing Design Docs

*Senior backend engineer at a startup, writes architecture proposals for team review*

**Pain:** Spends more time fighting diagram tools than thinking about architecture. Good at writing but finds diagramming tedious. Design docs without diagrams don't communicate well in async reviews.

**Core jobs:**
- Turn a written architecture description into a diagram for a design doc
- Quickly visualize a sequence of API calls for a PR description
- Generate an ER diagram from a schema description to share with the team

**Key moment:** "I wrote a 400-word description of the new auth flow. I should not have to spend another 45 minutes drawing it."

### Persona 2: Dev — The Full-Stack Developer Teaching Others

*Developer creating technical tutorials, blog posts, and documentation*

**Pain:** Good technical writing needs diagrams. Making diagrams is not writing. Having to switch tools breaks flow. Most tutorial diagrams are ugly screenshots of draw.io.

**Core jobs:**
- Generate diagrams for technical blog posts inline with writing
- Create system architecture visuals for README files
- Make flow diagrams for step-by-step technical guides

### Persona 3: Anika — The Tech Lead Reviewing Architecture

*Tech lead at a mid-size company, receives architecture proposals from the team*

**Pain:** Architecture proposals come in as pure text. She spends time mentally visualizing what the author is describing. She wants diagrams but doesn't want to make them herself — she wants the author to make them faster.

**Core jobs:**
- Understand a proposed architecture quickly from a combined text + diagram view
- Comment on specific parts of an architecture diagram in a review
- Generate a diagram from a verbal description during a sync call

### Persona 4: Sanjay — The Engineering Student & Content Creator

*Final-year CS student documenting learnings, building portfolio, creating system design content for X/LinkedIn*

**Pain:** Makes system design content but is not a professional designer. Wants diagrams that look clean and professional. Current process: Excalidraw manually → screenshot → embed → looks amateur.

**Core jobs:**
- Generate clean system design diagrams for social media posts
- Visualize system design interview problems quickly
- Create portfolio documentation with professional-quality visuals

*(This is directly you, Parthu — you are your own first and most authentic user for Aria.)*

---

## 5. Core Features — Full Detail

### Feature 1: The Spark — Zero-Prompt Diagram Generation

**What it does:** The primary interaction of Aria. A spark button (⚡) appears next to any block of text in the editor. Clicking it triggers diagram generation — no configuration, no prompt, no diagram type selection required.

**What happens under the hood:**
1. The text block is sent to the Diagram Intelligence Engine
2. The engine classifies the content (see Feature 2)
3. The appropriate diagram type is selected
4. The diagram is rendered inline, below the source text
5. 3 alternative diagram types are offered as chips below the diagram

**The 3-alternative chips:** Like Napkin AI's multiple options, Aria always offers alternatives. If it generated a sequence diagram, it shows chips for "Architecture View," "Flowchart View," and "Show as Table" so users can pivot if the auto-selection was wrong.

**Keyboard shortcut:** `Cmd+Shift+D` on any selected text block triggers the spark.

### Feature 2: Diagram Intelligence Engine

**What it does:** The technical brain of Aria. Classifies input text into the correct diagram category, extracts the entities and relationships, and generates the appropriate diagram DSL.

**Diagram types Aria can generate:**

| Diagram Type | Trigger Patterns | Output Format |
|---|---|---|
| Sequence Diagram | "User sends X → Service Y validates → DB stores" | Mermaid sequenceDiagram |
| ER Diagram | "users table has many orders; orders belong to products" | Mermaid erDiagram |
| System Architecture | "Load balancer → 3 EC2 → RDS in private subnet" | Custom SVG with icons |
| Flowchart | "if user is logged in... else... then..." | Mermaid flowchart |
| State Machine | "states: pending, processing, complete, failed; transitions: on payment → processing" | Mermaid stateDiagram-v2 |
| Class Diagram | "User class has name, email; inherits from BaseModel" | Mermaid classDiagram |
| Network Topology | "VPC contains public/private subnets; NAT gateway; bastion host" | Custom SVG |
| API Flow | "POST /users → validate → create user → send welcome email" | Sequence variant |
| Deployment Diagram | "Docker containers: api, worker, redis; deployed to ECS; behind ALB" | Custom SVG |
| Decision Tree | "if X > threshold then A else if Y > threshold then B else C" | Mermaid flowchart variant |
| Timeline | "Phase 1: Auth (Jan), Phase 2: Payments (Feb), Phase 3: Analytics (Mar)" | SVG timeline |
| Mind Map | "Core feature: branches: users, products, orders; users sub-branches: auth, profile" | Mermaid mindmap |

**Classification approach:**
- Stage 1: Rule-based pre-classifier (fast, cheap) — catches obvious patterns (arrow notation, ER keywords, state machine keywords)
- Stage 2: LLM classifier for ambiguous cases — sends text + candidate types, gets ranked probabilities
- Stage 3: If classification confidence < 0.7, show the user 2-3 type options to choose from

### Feature 3: Technical-Aware Rendering Engine

**What it does:** Renders diagrams using technical conventions, not generic shapes. This is what makes Aria feel like it was built by engineers rather than designers.

**Technical primitives Aria knows:**

*Infrastructure components (with proper icons):*
- Load Balancer (ALB/NLB)
- API Gateway
- CDN (CloudFront)
- Lambda / Serverless Function
- EC2 / VM Instance
- Kubernetes Pod / Deployment
- Database (SQL, NoSQL, differentiated visually)
- Cache (Redis, Memcached)
- Message Queue (SQS, RabbitMQ, Kafka)
- Object Storage (S3)
- Container Registry
- VPC / Network boundary
- Subnet (public vs private, differentiated by color)
- NAT Gateway
- Bastion Host
- DNS

*Software components:*
- REST API endpoint (with HTTP method badge: GET/POST/PUT/DELETE)
- gRPC service
- GraphQL endpoint
- WebSocket connection
- Event bus / pub-sub
- Worker / Consumer
- Scheduler / Cron
- Authentication service
- Service Mesh

*Data components:*
- Table with columns and types
- Relationship cardinality (1:1, 1:N, M:N)
- Index notation
- Foreign key

*Sequence diagram actors:*
- Browser/Client
- Mobile App
- Server/Service
- Database
- External Service/3rd Party
- Message Queue
- Cache

**Rendering technology:** Custom SVG renderer built on top of a base SVG engine. Mermaid is used for pure diagram DSL cases; custom renderer for architecture and network diagrams where Mermaid's output is too generic.

### Feature 4: Multimodal Input

**What it does:** Accepts input beyond just typed text.

**Input modes:**

**Voice narration:** Click the microphone icon, describe your system design aloud. "So we have a React frontend that makes API calls to our FastAPI backend, which is behind an Nginx reverse proxy. The backend connects to Postgres for main data and Redis for caching. Background jobs run through Celery with RabbitMQ as the broker." → Architecture diagram generated from the transcript.

**Whiteboard photo upload:** Drag a photo of a whiteboard sketch. The vision model reads the diagram structure, identifies boxes and arrows, infers labels (even from handwriting), and redraws it as a clean, editable SVG. The redrawn version retains the original structure but elevates it to professional quality.

**Existing diagram upload (for redesign):** Upload a rough screenshot of a draw.io diagram or a bad Lucidchart export. Aria cleans it up — aligns elements, applies consistent spacing, adds proper technical iconography, and makes it editable.

**Code-to-diagram:** Paste code (a class definition, a database schema in SQL, an OpenAPI spec, a Terraform configuration) and get a diagram. Aria reads the code semantically and generates the appropriate visualization.

**Examples:**
```python
# Input: Python code
class Order(Base):
    id: int
    user_id: ForeignKey(User.id)
    status: Enum[pending, paid, shipped, delivered]
    items: List[OrderItem]

# Output: ER diagram with Order entity,
# relationship to User (many-to-one),
# embedded state machine for status field
```

```yaml
# Input: Terraform
resource "aws_lb" "main" { ... }
resource "aws_ecs_service" "api" { ... }
resource "aws_rds_cluster" "db" { ... }

# Output: AWS architecture diagram with
# ALB → ECS → RDS topology, VPC boundaries
```

### Feature 5: Document Editor Integration

**What it does:** Aria is not a standalone diagramming canvas. It is a document editor with AI diagram generation built in — diagrams live alongside prose, not in a separate tool.

**Editor capabilities:**
- Markdown-based document editor with live preview
- Blocks: paragraph, heading, code, list, quote, diagram (new block type)
- Diagrams are first-class blocks — they appear inline, rendered, and editable
- Can export the entire document as PDF (with diagrams embedded), Markdown (with Mermaid code blocks), or HTML

**The editorial workflow:**
1. User writes technical prose in the editor
2. Selects any text block and clicks spark ⚡
3. Diagram appears as the next block in the document
4. User can continue writing below the diagram
5. The document with diagrams is the deliverable

**Notion-style block system:** Every block is movable, deletable, and transformable. A diagram block can be converted back to a Mermaid code block (for manual editing) or to a description (for accessibility).

### Feature 6: Diagram Editor

**What it does:** After a diagram is generated, the user can edit it directly in the UI without touching any code.

**Editing capabilities:**
- Click any element to rename it
- Drag elements to reposition (layout is suggestive, not fixed)
- Add/remove nodes and edges via context menu
- Change element type (e.g., convert a generic box to a "Database" primitive)
- Change connection style (solid, dashed, arrow direction)
- Apply color themes (dark/light/brand colors)
- Show/hide labels
- Expand a node to show sub-elements

**Dual edit mode:** Toggle between:
- **Visual mode:** WYSIWYG diagram editor
- **Code mode:** Mermaid/DSL code editor with live preview

Changes in either mode sync to the other in real time.

### Feature 7: Export & Integration

**What it does:** Makes Aria's output useful everywhere engineers work.

**Export formats:**
- PNG (lossless, transparent background)
- SVG (vector, infinitely scalable, editable in Figma/Illustrator)
- PDF (for documents)
- Mermaid code (paste directly into GitHub/GitLab markdown, Notion, Confluence)
- PlantUML code (for teams using PlantUML in CI)
- Figma-ready (via Figma API — pushes the SVG directly to a Figma frame)

**Integrations:**
- **Notion:** Share a diagram → generates an embed link that works in Notion
- **Confluence:** Export as Confluence-compatible format
- **GitHub README:** Generates the Mermaid code block ready to paste into a README (GitHub renders Mermaid natively)
- **VS Code extension (Phase 2):** Sidebar panel where you can generate diagrams from code you're looking at
- **Figma plugin (Phase 2):** Pull diagrams from Aria directly into a Figma file

### Feature 8: Template Library

**What it does:** A library of starting-point templates for common technical diagrams, pre-populated with placeholder content that users edit.

**Template categories:**

*System Design (Interview-Ready):*
- URL Shortener
- Chat Application
- Rate Limiter
- Content Delivery Network
- Notification Service
- News Feed
- Distributed Cache
- Search Engine

*Architecture Patterns:*
- Microservices with API Gateway
- Event-Driven Architecture
- CQRS + Event Sourcing
- BFF (Backend for Frontend)
- Saga Pattern
- Circuit Breaker

*Cloud Infrastructure:*
- AWS Three-Tier Web App
- AWS Serverless Architecture
- AWS EKS Deployment
- GCP Standard Web App
- Multi-Region Active-Active

*Database Schemas:*
- E-commerce (users, products, orders, payments)
- SaaS Multi-tenancy
- Social Network
- CMS
- IoT timeseries

*API Documentation:*
- REST CRUD API flow
- OAuth2 authorization code flow
- GraphQL subscription pattern
- Webhook delivery flow

**Template interaction:** Selecting a template pre-populates a description document. The user edits the description and clicks spark to regenerate the diagram with their specific entities and services. The template is a scaffold, not a finished product.

---

## 6. Diagram Types & Intelligence

### The Semantic Understanding Model

The hardest engineering problem in Aria is not the rendering — it's the understanding. Here is the detailed design of how Aria reads technical text and extracts diagram-worthy structure.

**Stage 1: Entity Extraction**

The LLM identifies:
- **Actors** (who/what initiates actions): User, Browser, Mobile App, Service Name, External System
- **Systems** (passive participants): Database, Cache, Queue, Storage, External API
- **Actions** (what happens between entities): sends, validates, queries, returns, publishes, consumes, stores, fetches
- **Data** (what moves): request, response, token, event, message, payload, record
- **States** (for state machines): pending, active, failed, complete, and transitions
- **Conditions** (for flowcharts): if, when, else, unless, only if, fallback

**Stage 2: Relationship Mapping**

From extracted entities, the LLM builds a relationship graph:
```json
{
  "entities": [
    { "id": "user", "type": "actor" },
    { "id": "api_gateway", "type": "service" },
    { "id": "auth_service", "type": "service" },
    { "id": "user_db", "type": "database" }
  ],
  "relationships": [
    { "from": "user", "to": "api_gateway", "action": "POST /login", "type": "sync" },
    { "from": "api_gateway", "to": "auth_service", "action": "validate credentials", "type": "sync" },
    { "from": "auth_service", "to": "user_db", "action": "SELECT user WHERE email=?", "type": "sync" },
    { "from": "auth_service", "to": "api_gateway", "action": "return JWT", "type": "response" },
    { "from": "api_gateway", "to": "user", "action": "200 OK + token", "type": "response" }
  ]
}
```

**Stage 3: Diagram Type Selection**

Rules for type selection:
- Has `→` or sequential action notation + multiple distinct actors → **Sequence Diagram**
- Has "has many", "belongs to", "foreign key", table/column language → **ER Diagram**
- Has infrastructure keywords (VPC, subnet, ALB, EC2, Lambda, S3) → **Architecture Diagram**
- Has "state", "transition", "on event" language → **State Machine**
- Has "if", "else", "decision" → **Flowchart**
- Has "phase", "sprint", "milestone", "deadline" → **Timeline**
- Has class/interface/inherit/extends language → **Class Diagram**

If multiple patterns match, confidence scores determine the primary type, with alternatives offered.

---

## 7. User Experience & Flows

### Primary Flow: Write → Spark → Diagram

```
1. User opens Aria (or creates new document)
2. Types or pastes technical content
3. Hovers over any block → spark button ⚡ appears
4. Clicks spark
5. Loading state: "Reading structure..." (< 2 seconds)
6. Diagram appears inline below the text block
7. Alternative type chips shown: [Sequence ✓] [Architecture] [Flowchart]
8. User clicks a citation to see the source text that generated it
9. User clicks to edit individual elements
10. User exports as SVG or Mermaid
```

### Interface Layout

```
┌────────────────────────────────────────────────────────────────┐
│  Aria  │  [Document Title]              [Share] [Export ▾]     │
├──────────────────────────────┬─────────────────────────────────┤
│  EDITOR                      │  DIAGRAM (live preview)         │
│                              │                                  │
│  # Authentication Flow       │  ┌──────────────────────────┐  │
│                              │  │                          │  │
│  The login flow works as     │  │   [Sequence Diagram]     │  │
│  follows: User submits       │  │   Browser → API Gateway  │  │
│  email/password to the       │  │   API → Auth Service     │  │
│  API Gateway. Gateway        │  │   Auth → PostgreSQL      │  │
│  forwards to Auth Service.   │  │   ← JWT returned         │  │
│  Auth validates against  ⚡  │  │                          │  │
│  PostgreSQL. Returns JWT     │  └──────────────────────────┘  │
│  token on success. Gateway   │                                  │
│  attaches token and          │  Type: [Sequence ✓] [Flow]      │
│  returns to client.          │       [Architecture]            │
│                              │                                  │
│  ## Database Schema          │  Export: [SVG] [PNG] [Mermaid]  │
│                              │  Edit:   [Visual] [Code]        │
│  The users table stores  ⚡  │                                  │
│  id, email, hashed_pwd,      ├─────────────────────────────────┤
│  created_at, role (enum:     │  DIAGRAM EDITOR                 │
│  admin|user|readonly).       │                                  │
│  Each user has many          │  Click any element to edit.     │
│  sessions (1:N). Sessions    │  Drag to reposition.            │
│  store token_hash,           │  Right-click for options.       │
│  expires_at, device_info.    │                                  │
│                              │                                  │
└──────────────────────────────┴─────────────────────────────────┘
```

### Alternative Input Flows

**Voice flow:**
```
Click 🎤 → Browser mic permission → Speak description → Stop recording
→ Transcription shown → Spark auto-triggers → Diagram generated
```

**Whiteboard photo flow:**
```
Click 📷 → Upload image → "Analyzing whiteboard..." → 
Detected elements shown → Clean diagram rendered → Editable
```

**Code flow:**
```
Paste code block → Aria detects language → Schema/class/config recognized →
Auto-suggests diagram → One click to generate
```

### Onboarding Experience

First-time user sees a 3-step guided experience:

**Step 1:** Pre-populated example text (AWS architecture description). Spark button pulsing. "Click ⚡ to see what Aria can do."

**Step 2:** Diagram generates. "Aria understood your architecture. Now try editing it — click any component."

**Step 3:** "Now try your own content. Paste any technical description, code, or schema." → Blank editor, ready.

Total onboarding: 90 seconds to first wow moment.

---

## 8. Technical Architecture

### System Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT (Browser)                          │
│  React + Vite │ Document Editor (ProseMirror/TipTap)            │
│  SVG Renderer │ D3 Layout Engine │ Mermaid Renderer              │
└───────────────────────────┬─────────────────────────────────────┘
                            │ HTTPS
┌───────────────────────────▼─────────────────────────────────────┐
│                    API LAYER (FastAPI)                            │
│  Auth │ Rate Limiting │ Document CRUD │ Diagram Generation       │
└─────┬──────────────┬───────────────┬───────────────┬────────────┘
      │              │               │               │
┌─────▼──────┐ ┌─────▼──────┐ ┌─────▼──────┐ ┌────▼───────────┐
│ Diagram    │ │ Document   │ │ Vision     │ │ Export         │
│ Engine     │ │ Service    │ │ Service    │ │ Service        │
│            │ │            │ │ (Whisper + │ │ (SVG→PNG/PDF)  │
│ Classify   │ │ CRUD       │ │ GPT-4V)    │ │                │
│ Extract    │ │ History    │ │            │ │                │
│ Generate   │ │ Versioning │ │            │ │                │
└─────┬──────┘ └─────┬──────┘ └─────┬──────┘ └────────────────┘
      │              │               │
┌─────▼──────────────▼───────────────▼───────────────────────────┐
│                    LLM GATEWAY (LiteLLM)                         │
│  Claude Sonnet 4.6 (primary) │ GPT-4o (fallback)                │
│  Whisper API (audio) │ GPT-4-Vision (image analysis)            │
└─────────────────────────────────────────────────────────────────┘
      │
┌─────▼──────────────────────────────────────────────────────────┐
│                    DATA LAYER                                    │
│  PostgreSQL (documents, diagrams, users)                         │
│  Redis (sessions, caching, rate limiting)                        │
│  S3 (exported files, uploaded images)                            │
└────────────────────────────────────────────────────────────────┘
```

### Diagram Generation Pipeline (Detailed)

```
Input Text
    │
    ├── Stage 1: Pre-processing
    │   ├── Clean whitespace, normalize punctuation
    │   ├── Detect code blocks (extract separately)
    │   ├── Detect existing Mermaid/PlantUML (pass through)
    │   └── Language detection (for code blocks)
    │
    ├── Stage 2: Classification (fast)
    │   ├── Rule-based classifier (regex patterns, 5ms)
    │   ├── If ambiguous → LLM classifier (500ms)
    │   └── Returns: {type, confidence, alternatives}
    │
    ├── Stage 3: Entity & Relationship Extraction
    │   ├── LLM extraction prompt (structured output via Instructor)
    │   ├── Returns: {entities[], relationships[], metadata}
    │   └── Validation: schema check on extraction output
    │
    ├── Stage 4: DSL Generation
    │   ├── For Mermaid types: template-based generation + LLM fill
    │   ├── For custom SVG: layout algorithm + SVG template
    │   └── Returns: valid DSL string or SVG specification
    │
    ├── Stage 5: Validation
    │   ├── Mermaid syntax validation (parse, catch errors)
    │   ├── If invalid → regenerate with error context (1 retry)
    │   └── Semantic check: does diagram match source entities?
    │
    └── Stage 6: Rendering & Response
        ├── Mermaid → client-side render via mermaid.js
        ├── Custom SVG → server-side layout + client-side render
        └── Response: {dsl, svg_preview, type, alternatives, confidence}
```

### Vision Pipeline (Whiteboard)

```
Image Upload (JPEG/PNG/HEIC)
    │
    ├── Pre-processing
    │   ├── Resize to 2048px max dimension
    │   ├── Enhance contrast (for whiteboard photos)
    │   └── Convert HEIC → JPEG
    │
    ├── GPT-4V Analysis
    │   └── Prompt: "Describe every element in this diagram:
    │             boxes, labels, arrows, connections, layout.
    │             Output as structured JSON."
    │
    ├── Structure Extraction
    │   └── Map GPT-4V output to entity-relationship format
    │
    └── Diagram Generation
        └── Same pipeline as text input from Stage 3
```

---

## 9. AI Pipeline Design

### Classification Prompt (Stage 2)

```
SYSTEM:
You are a technical diagram classifier. Given a piece of technical
text, classify what type of diagram best represents it.

DIAGRAM TYPES:
- sequence: actor interactions in time order
- er_diagram: entities and their data relationships
- architecture: systems/services and their connections
- flowchart: decision trees, process flows with conditions
- state_machine: states and transitions for a system/entity
- class_diagram: OOP classes, inheritance, interfaces
- network: infrastructure, subnets, VPCs, network topology
- deployment: containerized/cloud deployment topology
- timeline: phases, milestones, time-ordered events
- mind_map: hierarchical topic breakdown

RULES:
- Return only the type name and a confidence 0.0-1.0
- If confidence < 0.65, return top 2 types
- When in doubt between sequence and architecture:
  sequence = time-ordered interactions; architecture = static topology

INPUT: {text}

OUTPUT FORMAT:
{"type": "sequence", "confidence": 0.89, "alternatives": ["architecture"]}
```

### Extraction Prompt (Stage 3)

```
SYSTEM:
Extract the entities and relationships from this technical description
for generating a {diagram_type} diagram.

For sequence diagrams, extract:
- participants (actors and systems, in order of first appearance)
- messages (from, to, label, type: sync/async/return, optional: note)

For ER diagrams, extract:
- entities (name, attributes with types)
- relationships (entity1, entity2, cardinality: 1:1/1:N/M:N, label)

For architecture diagrams, extract:
- components (name, type: service/database/cache/queue/gateway/cdn/storage)
- connections (from, to, protocol: HTTP/gRPC/SQL/AMQP/TCP, label)
- groups (VPC, subnet, availability zone boundaries)

Use Instructor-enforced Pydantic schema for output.

INPUT: {text}
DIAGRAM TYPE: {type}
```

### DSL Generation (Stage 4)

For sequence diagrams, the generation is template-driven:

```python
def generate_sequence_mermaid(extraction: SequenceExtraction) -> str:
    lines = ["sequenceDiagram"]
    
    # Declare participants in order
    for p in extraction.participants:
        prefix = "actor" if p.type == "actor" else "participant"
        lines.append(f"    {prefix} {p.id} as {p.label}")
    
    # Generate message lines
    for msg in extraction.messages:
        arrow = "-->>+" if msg.type == "async" else "->>"
        if msg.type == "return":
            arrow = "-->>-"
        lines.append(f"    {msg.from_id}{arrow}{msg.to_id}: {msg.label}")
        if msg.note:
            lines.append(f"    Note over {msg.from_id},{msg.to_id}: {msg.note}")
    
    return "\n".join(lines)
```

For architecture diagrams, a custom SVG layout algorithm:
1. Group components by type (services, databases, etc.)
2. Apply hierarchical layout: gateways/LBs at top, services in middle, databases at bottom
3. Assign positions using a force-directed algorithm constrained by layer
4. Render SVG with proper icons per component type

### Caching Strategy

**Identical input cache:** If the exact same text block generates a diagram, cache the output. Key: `hash(text + diagram_type)`. TTL: 24 hours. Invalidate on diagram type change.

**Semantic cache:** For similar (but not identical) text, check if a semantically similar input already has a cached output. Use embedding similarity with threshold 0.92. Saves ~30% of LLM calls for common architectural patterns.

---

## 10. Data Model

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email TEXT UNIQUE NOT NULL,
    name TEXT,
    avatar_url TEXT,
    plan TEXT DEFAULT 'free',  -- free|pro|team
    sparks_used INTEGER DEFAULT 0,
    sparks_limit INTEGER DEFAULT 50,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    title TEXT DEFAULT 'Untitled',
    content JSONB NOT NULL DEFAULT '{}',  -- ProseMirror document JSON
    is_public BOOLEAN DEFAULT false,
    share_token TEXT UNIQUE,  -- for public sharing
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE diagrams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID REFERENCES documents(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id),
    source_text TEXT NOT NULL,  -- the input text that generated this
    diagram_type TEXT NOT NULL,
    dsl TEXT NOT NULL,  -- Mermaid code or SVG spec
    svg_cache TEXT,  -- cached rendered SVG
    extraction JSONB,  -- the structured extraction result
    generation_model TEXT,
    generation_latency_ms INTEGER,
    user_edited BOOLEAN DEFAULT false,  -- was it manually edited after gen?
    confidence FLOAT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE diagram_feedback (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    diagram_id UUID REFERENCES diagrams(id),
    user_id UUID REFERENCES users(id),
    rating INTEGER CHECK (rating IN (1, -1)),  -- thumbs up/down
    wrong_type BOOLEAN DEFAULT false,
    comment TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE exports (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    diagram_id UUID REFERENCES diagrams(id),
    format TEXT NOT NULL,  -- svg|png|pdf|mermaid|figma
    s3_key TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_documents_user ON documents(user_id, updated_at DESC);
CREATE INDEX idx_diagrams_document ON diagrams(document_id);
CREATE INDEX idx_diagram_feedback_diagram ON diagram_feedback(diagram_id);
```

---

## 11. API Design

```
Authentication
  POST /auth/google          → Google OAuth (simplest for web)
  POST /auth/github          → GitHub OAuth
  POST /auth/refresh         → Refresh JWT

Documents
  GET  /documents            → List user's documents
  POST /documents            → Create new document
  GET  /documents/{id}       → Get document with diagrams
  PUT  /documents/{id}       → Update document content
  DEL  /documents/{id}       → Delete document
  POST /documents/{id}/share → Generate public share link

Diagram Generation (SSE Streaming)
  POST /diagrams/generate    → Generate diagram from text (streams progress)
  {
    "text": "...",
    "document_id": "uuid",
    "hint_type": null  // optional override
  }

  SSE Events:
  { "type": "classifying", "progress": 0.2 }
  { "type": "extracting", "progress": 0.5 }
  { "type": "generating", "progress": 0.8 }
  { "type": "done", "diagram": { "id": "...", "dsl": "...", "type": "..." } }
  { "type": "error", "code": "CLASSIFICATION_FAILED", "alternatives": [...] }

Diagram Operations
  GET  /diagrams/{id}           → Get diagram detail
  PUT  /diagrams/{id}           → Update diagram (after manual edit)
  POST /diagrams/{id}/regen     → Regenerate with different type
  POST /diagrams/{id}/feedback  → Submit thumbs up/down
  POST /diagrams/{id}/export    → Export in format (returns S3 URL)

Voice Input
  POST /voice/transcribe        → Upload audio, get transcript
  (Then pass transcript to /diagrams/generate)

Vision Input
  POST /vision/analyze          → Upload image, get extraction JSON
  (Then pass extraction to diagram generation as pre-extracted entities)

Templates
  GET  /templates               → List all templates
  GET  /templates/{id}          → Get template content
  POST /templates/{id}/use      → Create document from template
```

---

## 12. Evaluation & Quality

### Diagram Quality Metrics

**Type accuracy:** Does the generated diagram type match what the user intended?
- Measured via: user feedback (wrong type button) + A/B testing type options
- Target: > 88% correct type on first generation

**Structural completeness:** Are all entities from the source text represented in the diagram?
- Measured via: LLM-as-judge comparing entity list in extraction vs entities in diagram
- Target: > 92% of named entities appear in the diagram

**User edit rate:** What % of diagrams are edited after generation?
- Lower is better (means generation was good enough)
- Target: < 35% require any edit

**Export rate:** What % of diagrams get exported?
- Higher means users found them good enough to use
- Target: > 55% of generated diagrams exported at least once

### Golden Dataset

50 input texts × diagram type = 50 test cases per type × 12 types = 600 total test cases.

Each test case includes:
- Input text
- Expected diagram type
- Expected entities (minimum required)
- Human-rated reference diagram

Nightly eval run reports regression alerts if any metric drops > 3%.

### Feedback Loop

**In-product feedback:** After every diagram:
- Thumbs up / thumbs down
- "Wrong type?" button
- Optional: "What were you expecting?" free text

**Aggregate analysis:** Weekly review of:
- Most common wrong-type errors
- Most edited diagram elements (what did users fix?)
- Most common input patterns that produce poor diagrams

This data feeds directly into prompt improvements and classification refinement.

---

## 13. MVP Scope & Phased Roadmap

### MVP (Week 1–2): Prove the Core

**Goal:** Text → correct diagram → export. Nothing else.

**Included:**
- Basic document editor (textarea, not rich editor)
- Spark button on any selected text
- 4 diagram types: sequence, ER, architecture, flowchart
- Mermaid rendering for sequence/ER/flowchart
- Basic SVG rendering for architecture
- Export: Mermaid code copy, PNG download
- No auth (anonymous sessions, local storage)

**Excluded from MVP:**
- Document persistence
- Voice input
- Whiteboard photo
- Template library
- Diagram editor
- All other diagram types

**MVP success criteria:**
- Correctly identifies diagram type 80%+ of the time
- Generates valid Mermaid on first try 90%+ of the time
- Total generation time < 4 seconds
- 10 people use it and at least 6 say they'd use it weekly

### Phase 2 (Week 3–5): Full Feature Set

- Add remaining diagram types (state machine, class, network, deployment, timeline, mind map)
- Rich text document editor (TipTap or ProseMirror)
- Document persistence (auth via GitHub/Google)
- Visual diagram editor (click to edit elements)
- Voice input (Whisper transcription)
- Template library (20 templates)

### Phase 3 (Week 6–8): Polish & Integration

- Whiteboard photo upload
- Code-to-diagram (SQL schema, Python classes, Terraform)
- Figma export integration
- Notion embed link
- VS Code extension (sidebar panel)
- Collaborative editing (multiple cursors)

### Phase 4 (Week 9–12): Scale

- Team workspaces
- Brand theme customization
- API access for programmatic diagram generation
- GitHub Actions integration (auto-generate diagrams in CI)
- Confluence plugin
- Diagram versioning and diff

---

## 14. Monetization

### Pricing

**Free:**
- 50 sparks/month (diagram generations)
- 10 documents
- PNG + Mermaid export
- Public share links

**Pro — $15/month:**
- 500 sparks/month
- Unlimited documents
- All export formats (SVG, PDF, Figma)
- Voice + whiteboard input
- Template library access
- Document history (30 days)

**Team — $12/month per seat (min 3 seats):**
- Everything in Pro
- Shared workspace + documents
- Team template library
- Admin dashboard
- API access (1,000 API calls/month)
- Priority support

**API — Usage-based:**
- $0.05 per diagram generated via API
- For developers integrating Aria into their own tools

### Unit Economics

| Component | Cost per Diagram |
|---|---|
| LLM calls (classification + extraction + generation) | ~$0.008 |
| Infrastructure | ~$0.002 |
| **Total per diagram** | **~$0.01** |

Free tier: 50 diagrams × $0.01 = $0.50/user/month cost. Acceptable for acquisition.
Pro tier: 500 diagrams × $0.01 = $5.00 cost vs $15 revenue → healthy 67% gross margin.

---

## 15. Launch Strategy

### The Pre-Launch Content Play

Aria has a natural content loop with Parthu's existing X/LinkedIn audience:

**Week 1–2 (while building):** Tweet series on "Building Napkin AI for developers." Show the architecture decisions, the hardest problems, the diagrams Aria can generate that Napkin AI can't.

**Week 3:** Post first working demo — "I pasted a system design description. Aria generated this sequence diagram in 3 seconds. No prompting." Video of the spark interaction.

**Week 4:** Post the AWS architecture I'm studying for SAA-C03 → Aria generates the diagram → "I used my own tool to study for my AWS certification."

**Launch week:** Product Hunt + "Show HN: Aria — Napkin AI for engineers" + full technical writeup on how the diagram intelligence works.

### The Hook That Works for This Product

Every engineer has written a paragraph describing a system design and then had to manually draw the diagram. The demo should be exactly that — write one paragraph, click once, diagram appears. That 3-second clip is the entire marketing message.

### Distribution Channels

**Primary:**
- X/Twitter (technical audience, system design community)
- Hacker News Show HN
- Product Hunt

**Secondary:**
- Dev.to and Hashnode (technical article: "How I built the diagram intelligence engine")
- GitHub (open-source the Mermaid generation layer as a library)
- Reddit: r/ExperiencedDevs, r/programming, r/aws, r/systemdesign

**Flywheel:** Every diagram shared publicly links back to Aria. System design content creators (a large X community) are the highest-leverage distribution channel.

---

## 16. Success Metrics

### North Star
**Diagrams exported per week** — captures both new user growth (more diagrams generated) and quality (diagrams worth exporting)

### Supporting Metrics

| Metric | Target (Month 3) |
|---|---|
| Monthly active users | 1,000+ |
| Diagrams generated/month | 15,000+ |
| Export rate | > 55% |
| Type accuracy (user feedback) | > 85% |
| D7 retention | > 35% |
| Pro conversion rate | > 5% |
| Generation P95 latency | < 5s |
| LLM cost per diagram | < $0.012 |

---

## 17. Tech Stack

### Frontend
```
Framework:     Vite + React 18 + TypeScript
Editor:        TipTap (ProseMirror-based, extensible)
Diagram render: Mermaid.js (client-side) + custom SVG renderer
Visualization: D3.js (force-directed layout for architecture diagrams)
Icons:         Custom SVG icon set for technical primitives
               (AWS icons, generic service icons)
State:         Zustand
Styling:       Tailwind CSS
Export:        html2canvas (PNG), jsPDF (PDF)
```

### Backend
```
Framework:     FastAPI (Python 3.11+)
Task Queue:    Celery + Redis (for async export jobs)
Auth:          Google OAuth + GitHub OAuth + JWT
Storage:       AWS S3 (exports, uploaded images)
```

### AI Layer
```
Primary LLM:   Claude Sonnet 4.6 (structured output + extraction)
Fallback:      GPT-4o
Vision:        GPT-4-Vision (whiteboard analysis)
Audio:         OpenAI Whisper API (voice transcription)
Structured:    Instructor (Pydantic-enforced LLM outputs)
Gateway:       LiteLLM (unified API + cost tracking)
Tracing:       Langfuse
```

### Data
```
Database:      PostgreSQL 16 (Supabase)
Cache:         Redis (sessions, diagram cache, rate limiting)
File Storage:  AWS S3
```

### Infrastructure
```
Hosting:       Railway or AWS ECS
Container:     Docker
CI/CD:         GitHub Actions
Monitoring:    Langfuse (LLM) + Sentry (errors)
Analytics:     PostHog (product analytics)
```

---

## Appendix: Diagram Type Decision Tree

```
Does the text describe time-ordered interactions between multiple actors?
  YES → Sequence Diagram
  NO  ↓
Does it describe data entities with attributes and relationships?
  YES → ER Diagram
  NO  ↓
Does it describe infrastructure services and their connections?
  YES → Does it include network/subnet/VPC language?
         YES → Network Topology
         NO  → Architecture Diagram
  NO  ↓
Does it describe states and transitions for a system or object?
  YES → State Machine
  NO  ↓
Does it include conditional logic (if/else/when/decision)?
  YES → Flowchart
  NO  ↓
Does it describe classes, interfaces, or OOP relationships?
  YES → Class Diagram
  NO  ↓
Does it describe time phases or milestones?
  YES → Timeline
  NO  ↓
Does it describe hierarchical topic decomposition?
  YES → Mind Map
  NO  → Flowchart (default fallback)
```

---

*Document version: 1.0 | Built for production-level portfolio showcase and real product launch*
