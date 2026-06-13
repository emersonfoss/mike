# Mike — Developer Onboarding Guide

> Generated from the Understand-Anything knowledge graph (analyzed 2026-06-13, commit `d39f5806`).

Welcome to **Mike**, a legal document assistant. This guide gives you the lay of the land: what the product does, how the code is layered, the concepts that recur everywhere, a recommended reading path, and the files you'll touch most often.

---

## 1. Project Overview

| | |
|---|---|
| **Name** | mike |
| **What it is** | Legal document assistant: lawyers upload matter documents, chat with an LLM agent that can read/find/generate/edit those documents, run spreadsheet-style multi-document reviews, and share reusable prompt workflows. |
| **Shape** | Two apps in one repo: `frontend/` (Next.js 16, React 19, App Router, Tailwind v4, deployed to Cloudflare via OpenNext) and `backend/` (Express + TypeScript). |
| **Languages** | TypeScript (dominant), JavaScript, SQL, CSS, JSON/TOML config, Markdown |
| **Frameworks/Platforms** | Next.js, React, Express, Tailwind CSS, Supabase (Auth + Postgres) |
| **External services** | Supabase (auth, Postgres), Cloudflare R2 / S3-compatible object storage (document files), LLM providers (Anthropic Claude, Google Gemini, OpenAI — at least one key required), LibreOffice (DOCX→PDF conversion), Resend (email) |
| **Database convention** | `backend/schema.sql` bootstraps a fresh database; incremental migrations apply to existing ones. |

The mental model to keep: **everything the UI does becomes an HTTP request to one of eight Express routers, which delegate to `backend/src/lib/` modules, which talk to Supabase Postgres, R2 storage, and the LLM adapters.**

---

## 2. Architecture Layers

The graph identifies ten layers. Frontend layers sit on top; everything funnels through the backend API into services and data.

### Frontend

1. **Frontend Pages & App Shell** (`frontend/src/app/(pages)/`, `frontend/src/app/*.tsx`)
   Next.js App Router route pages, root layout, error/not-found boundaries, and global styles. Defines the assistant, projects, workflows, tabular-reviews, account, login/signup, and support screens. Route pages are deliberately thin — they unwrap params and render feature components.

2. **Frontend Feature Components** (`frontend/src/app/components/`)
   The bulk of the UI (≈70 files): assistant chat panels, tabular review grids, workflow builders, project file browsers, modals, and a large shared set (document viewers, uploads, citations). Organized by feature: `assistant/`, `projects/`, `tabular/`, `workflows/`, `shared/`, `modals/`.

3. **Frontend UI Primitives** (`frontend/src/components/`)
   shadcn-style design-system primitives (button, input, badge, dropdown, cite-button), the animated MikeIcon chat glyph, the provider composition root, and the site logo.

4. **Frontend State & Hooks** (`frontend/src/contexts/`, `frontend/src/app/contexts/`, `frontend/src/app/hooks/`)
   React contexts (auth session, user profile, chat history, sidebar) and custom hooks for assistant chat streaming, document version/bytes fetching, and model selection.

5. **Frontend Services & Utilities** (`frontend/src/app/lib/`, `frontend/src/lib/`)
   The client-side service layer: `mikeApi.ts` backend client, Supabase browser client, R2/S3 storage helpers, file conversion, formatting utilities, shared TypeScript types.

### Backend

6. **Backend API Layer** (`backend/src/index.ts`, `backend/src/middleware/`, `backend/src/routes/`)
   Express entry point, Supabase JWT auth middleware, and eight route modules: chat, project chat, documents, downloads, projects, tabular reviews, user, workflows. Routes stay thin and delegate to lib modules.

7. **Backend Services & LLM Adapters** (`backend/src/lib/`, `backend/src/lib/llm/`)
   Business logic: access control, the agentic chat engine (`chatTools.ts`), document conversion/versioning, the DOCX tracked-changes OOXML engine, uploads, R2 storage, user API keys/settings, built-in workflows — plus the multi-provider LLM adapter layer (Claude, Gemini, OpenAI behind one neutral contract).

### Data, Config, Docs

8. **Data Layer** (`backend/schema.sql`)
   Consolidated Supabase Postgres schema: 16 tables covering user profiles and encrypted API keys, projects/subfolders, documents with versions and AI edits, chats/messages, workflows and sharing, and tabular reviews.

9. **Configuration & Tooling**
   Package manifests, TypeScript/Next.js/OpenNext/PostCSS/ESLint configs, env templates (`backend/.env.example`, `frontend/.env.local.example`), and `backend/nixpacks.toml` (bakes LibreOffice into the deploy image).

10. **Documentation**
    `README.md` (setup and repo layout), `CONTRIBUTING.md` (small focused PRs, build checks, security reporting), and `docs/safe-local-testing.md` (develop against disposable resources — read this before touching anything live).

---

## 3. Key Concepts

The domain graph groups the product into six domains; the patterns below are how those domains are realized in code.

### Product domains

- **Identity & Access Control** — Supabase email/password auth; per-user profiles with monthly message credits and subscription tier; encrypted bring-your-own LLM API keys; ownership and email-based sharing rules.
- **Matters & Document Management** — projects model a legal matter (client-matter number, `shared_with` collaborator list); documents are uploaded, converted to PDF, and stored in R2 under immutable version rows.
- **Agentic Assistant Chat** — the flagship feature: a streaming, tool-using LLM conversation over the user's legal documents.
- **Document Drafting & Redlining** — the assistant generates styled legal DOCX from scratch and proposes edits as genuine Word tracked changes that the user accepts or rejects.
- **Tabular Multi-Document Review** — spreadsheet-style diligence: each row a document, each column an extraction question; LLM-generated cells stream in with citations.
- **Workflow Template Library** — reusable legal expertise as prompts: chat- and tabular-typed templates with practice areas, built-in firm templates, and email-based sharing.

### Design patterns and decisions

- **Thin routes, fat lib.** Express routers validate and delegate; real work lives in `backend/src/lib/`. Follow that pattern when adding endpoints.
- **Provider-neutral LLM adapter.** `lib/llm/types.ts` defines one shape for messages, tools, and streaming callbacks; `claude.ts`/`gemini.ts`/`openai.ts` each translate it to their native API and drive a multi-turn tool-use loop; `lib/llm/index.ts` routes by model id. The rest of the codebase never mentions a specific provider.
- **The agentic tool loop.** `lib/chatTools.ts` is the brain: system prompt, tool schemas (read/find/generate/edit documents plus project, tabular, and workflow extras), document-context building, tool-call execution, and stream orchestration. `lib/llm/tools.ts` converts neutral tool definitions to provider-native schemas.
- **Tracked changes as a first-class feature.** `lib/docxTrackedChanges.ts` is a pure-TypeScript OOXML engine that parses `word/document.xml` and applies/resolves `w:ins`/`w:del` via diff-anchored paragraph rewriting. AI edits are persisted in `document_edits` and surfaced in the UI as accept/reject cards.
- **Immutable document versions.** Every upload or accepted edit creates a `document_versions` row pointing at the original file and converted PDF in R2 under deterministic keys. DOCX→PDF conversion runs through LibreOffice (`lib/convert.ts`); nixpacks installs LibreOffice in production.
- **SSE streaming with drip-feed rendering.** Chat and tabular responses stream as server-sent events; `useAssistantChat.ts` (and `TRChatPanel`) drip-feed content for a smooth typing effect, tracking citations and tool events along the way.
- **Auth flows end to end.** The Supabase browser client's session token becomes the bearer token on every `mikeApi.ts` request; `middleware/auth.ts` validates it and attaches `userId`/`userEmail`; `lib/access.ts` enforces ownership/sharing on each resource. Public downloads bypass sessions via HMAC-signed expiring tokens (`lib/downloadTokens.ts` → `routes/downloads.ts`).
- **Credits and BYO keys.** Message credits are enforced in the chat route and reset monthly via the user route; users can store their own provider API keys, encrypted AES-256-GCM at rest (`lib/userApiKeys.ts`), with environment-key fallbacks. Frontend model pickers gate models on key availability (`modelAvailability.ts`).
- **Module-level caches on the frontend.** `useDirectoryData.ts` (30s TTL document directory) and `useFetchDocxBytes.ts` (byte cache with in-flight dedupe) both export explicit invalidation functions — call them after mutations.
- **Two files unlock everything.** `frontend/src/app/components/shared/types.ts` (53 importers) and `frontend/src/app/lib/mikeApi.ts` (37 importers) are the shared vocabulary and the API surface. Master these and you can read almost any component.

---

## 4. Guided Tour

A recommended 15-step reading path. Backend first, then the frontend that consumes it.

1. **Project Overview** — Read `README.md`: what Mike is, the two-app shape, prerequisites (Supabase, R2 bucket, one LLM key), and the schema.sql-vs-migrations convention.

2. **Backend Entry Point** — `backend/src/index.ts` is the map of the whole API: helmet, CORS, JSON limits, tiered rate limiters, then eight feature routers. Everything the UI does arrives here.

3. **The API Route Map** — Skim `routes/chat.ts` (streaming LLM loop with credit enforcement), `routes/documents.ts` (upload, conversion, versioning, tracked-change accept/reject), `routes/projects.ts` (matters, folders, sharing, batch uploads), `routes/tabular.ts` (multi-document extraction), `routes/workflows.ts` (shareable prompt templates). Notice the thin-routes pattern.

4. **Authentication & Access Control** — `middleware/auth.ts` (bearer-token validation), `lib/supabase.ts` (service-role client), `lib/access.ts` (ownership + email sharing rules). Nearly every data access flows through these.

5. **Document Storage & Processing** — `lib/storage.ts` (R2 client, deterministic keys), `lib/upload.ts` (multer), `lib/convert.ts` (LibreOffice DOCX→PDF), `lib/documentVersions.ts` (version resolution), and the standout `lib/docxTrackedChanges.ts` (OOXML tracked-changes engine).

6. **Multi-Provider LLM Adapters** — `lib/llm/types.ts` (neutral contracts), `claude.ts`/`gemini.ts`/`openai.ts` (per-provider tool loops), `lib/llm/index.ts` (router facade keyed on model id).

7. **The Agentic Chat Engine** — `lib/chatTools.ts` + `lib/llm/tools.ts`: system prompt, tool schemas, context building, tool execution, and the streaming orchestration that turns a chat endpoint into an agent that reads and edits stored documents.

8. **The Database Schema** — `backend/schema.sql`: all 16 tables, the auth signup trigger, and indexes. Pay attention to `projects` (sharing), `documents`/`document_versions` (storage mirroring), and `chats`.

9. **Frontend App Shell** — `src/app/layout.tsx` (fonts, global providers), `src/app/page.tsx` (redirect to /assistant), `src/app/(pages)/layout.tsx` (auth guard, contexts), `components/shared/AppSidebar.tsx` (navigation mapping one-to-one onto the backend routers).

10. **Session & State Contexts** — `contexts/AuthContext.tsx` (23 importers — the backbone), `contexts/UserProfileContext.tsx` (credits, tier, key state), `app/contexts/ChatHistoryContext.tsx` (paginated chat list, optimistic updates), `lib/supabase.ts` (the browser client whose token Step 4 validates).

11. **Shared Types & API Client** — `components/shared/types.ts` (53 importers) and `app/lib/mikeApi.ts` (37 importers, SSE streaming, auto-attached auth token), plus `app/lib/modelAvailability.ts` (model→provider gating).

12. **The Assistant Chat Experience** — `(pages)/assistant/chat/[id]/page.tsx` → `hooks/useAssistantChat.ts` (SSE consumption, drip-feed) → `components/assistant/ChatView.tsx` (message list + side panels) → `ChatInput.tsx` (attachments, workflows, model toggle). The full loop: input, backend tool loop, streamed render.

13. **Project Workspaces** — `components/projects/ProjectsOverview.tsx` (dashboard), `ProjectPage.tsx` (the deliberately monolithic workspace: folder tree, drag-and-drop, version history, zip download, ~20 mikeApi calls), `shared/FileDirectory.tsx` and `shared/AddDocumentsModal.tsx` (the document pickers reused app-wide).

14. **Tabular Reviews & Workflows** — `components/tabular/TabularReviewView.tsx` (highest fan-out component: streaming cells, drag-drop docs, Excel export), `TRChatPanel.tsx` (citation-aware review chat), `components/workflows/WorkflowList.tsx` and `(pages)/workflows/[id]/page.tsx` (the library and editor; workflows launch chats or spawn tabular reviews).

15. **Configuration & Running It Safely** — `backend/.env.example` (every external dependency), `frontend/.env.local.example`, `backend/nixpacks.toml` (LibreOffice in the image), `docs/safe-local-testing.md` and `CONTRIBUTING.md` (disposable resources; small focused PRs).

---

## 5. File Map

Key files by layer. Complexity: **C** = complex, M = moderate, S = simple.

### Backend API (`backend/src/`)

| File | Cx | What it does |
|---|---|---|
| `index.ts` | M | Express entry: helmet, CORS, rate limiters; mounts the eight routers. |
| `middleware/auth.ts` | S | Validates Supabase bearer token; attaches `userId`/`userEmail`; 401 otherwise. |
| `routes/chat.ts` | **C** | Chat/message CRUD, title generation, streaming chat endpoint with credit enforcement and the LLM tool loop. |
| `routes/projectChat.ts` | M | Project-scoped streaming chat: access check, project-wide document context, persistence. |
| `routes/documents.ts` | **C** | Upload + DOCX→PDF conversion, version list/upload/restore, signed previews/downloads, tracked-change accept/reject, rename, delete. |
| `routes/projects.ts` | **C** | Project CRUD + email sharing, subfolders, batch uploads, document moves and listings. |
| `routes/tabular.ts` | **C** | Review/column/row CRUD, per-cell and whole-row LLM queries over extracted markdown, citation-aware review chat. |
| `routes/workflows.ts` | **C** | Workflow CRUD with email sharing and per-share edit permissions. |
| `routes/user.ts` | **C** | Profile read/update, monthly credit reset, tier handling, BYO API key status/save. |
| `routes/downloads.ts` | M | Public token-gated download streaming (HMAC verified, no session). |

### Backend Services & LLM (`backend/src/lib/`)

| File | Cx | What it does |
|---|---|---|
| `chatTools.ts` | **C** | Core chat engine: system prompt, tool schemas (TOOLS / PROJECT_EXTRA_TOOLS / TABULAR_TOOLS / WORKFLOW_TOOLS), context building, tool execution, stream orchestration. |
| `docxTrackedChanges.ts` | **C** | Pure-TS OOXML engine: parses `word/document.xml`, applies/resolves `w:ins`/`w:del` via diff-anchored paragraph rewriting. |
| `access.ts` | M | Ownership + email-sharing authorization for projects, documents, tabular reviews. |
| `storage.ts` | M | R2/S3 client: upload/download/delete, signed URLs, deterministic key builders. |
| `upload.ts` | S | Multer memory-storage middleware with size limits and JSON error mapping. |
| `convert.ts` | M | DOCX→PDF via LibreOffice, with zip path normalization for odd DOCX archives. |
| `documentVersions.ts` | M | Resolves active version rows; batch-attaches storage paths and edit version numbers. |
| `downloadTokens.ts` | M | HMAC-signed, expiring download tokens for the public downloads endpoint. |
| `userApiKeys.ts` | M | BYO key management: AES-256-GCM at rest, env fallbacks, provider normalization. |
| `userSettings.ts` | S | Effective model settings (tabular/title models) and merged API keys per user. |
| `supabase.ts` | S | Service-role client factory + bearer-token user-id extraction. |
| `builtinWorkflows.ts` | M | Three built-in legal workflow templates shipped with the backend. |
| `llm/index.ts` | S | Provider router facade: dispatches to Claude/Gemini/OpenAI by model id. |
| `llm/types.ts` | M | Provider-neutral contracts: messages, tools, calls/results, streaming callbacks. |
| `llm/claude.ts` | M | Anthropic streaming adapter with multi-turn tool-use loop and reasoning callbacks. |
| `llm/gemini.ts` | M | Gemini adapter mapping tool calls to function-calling parts with synthesized ids. |
| `llm/openai.ts` | **C** | OpenAI Responses API adapter over raw fetch with hand-rolled SSE parsing. |
| `llm/tools.ts` | M | Converts neutral tool definitions to Claude/Gemini native schemas. |
| `llm/models.ts` | S | Model catalog: allowed ids per tier, defaults, provider resolution. |

### Data Layer (`backend/schema.sql`)

One consolidated schema (**C**) defining 16 tables:

- **Users**: `user_profiles` (tier, credits, reset date, tabular model — auto-created by signup trigger), `user_api_keys` (AES-GCM ciphertext per provider).
- **Matters**: `projects` (client-matter number, `shared_with` list), `project_subfolders` (self-referencing tree).
- **Documents**: `documents` (metadata, parse tree, status, folder), `document_versions` (immutable rows → R2 paths for original + PDF), `document_edits` (AI tracked edits with context and accept/reject status).
- **Chat**: `chats` (optionally project-scoped), `chat_messages` (role, content, files, citations).
- **Workflows**: `workflows` (chat/tabular templates), `hidden_workflows`, `workflow_shares` (email grants with edit flag).
- **Tabular**: `tabular_reviews` (columns, doc ids, source workflow, sharing), `tabular_cells` (per doc × column, citations, status), `tabular_review_chats`, `tabular_review_chat_messages`.

### Frontend Pages & App Shell (`frontend/src/app/`)

| File | Cx | What it does |
|---|---|---|
| `layout.tsx` | M | Root layout: Inter + EB Garamond fonts, metadata, global auth/profile providers. |
| `page.tsx` | S | Redirects `/` → `/assistant`. |
| `(pages)/layout.tsx` | M | Authenticated shell: route guard, ChatHistory + Sidebar contexts, AppSidebar. |
| `(pages)/assistant/page.tsx` | S | Initial prompt view; creates a chat and redirects to its route. |
| `(pages)/assistant/chat/[id]/page.tsx` | M | Loads a chat by id (or seeds from handoff) and renders ChatView. |
| `(pages)/projects/page.tsx` | S | Renders ProjectsOverview. |
| `(pages)/projects/[id]/page.tsx` (+ `/assistant`, `/tabular-reviews` siblings) | S | Unwrap params; render ProjectPage on the right tab. |
| `(pages)/projects/[id]/assistant/chat/[chatId]/page.tsx` | **C** | Resizable three-pane project chat: file explorer, streaming chat with citations, PDF/DOCX viewer tabs. |
| `(pages)/tabular-reviews/page.tsx` | **C** | Reviews listing: search, tabs, creation modal, owner-only guards. |
| `(pages)/tabular-reviews/[id]/page.tsx` | S | Standalone review route → TRView. |
| `(pages)/workflows/page.tsx` | S | Renders WorkflowList. |
| `(pages)/workflows/[id]/page.tsx` | **C** | Workflow editor: autosaved prompt markdown, sortable columns, sharing. |
| `(pages)/account/page.tsx`, `account/models/page.tsx`, `account/layout.tsx` | C/C/M | Account settings: profile/credits/delete; per-provider API keys + tabular model. |
| `login/page.tsx`, `signup/page.tsx`, `support/page.tsx` | M/C/C | Supabase auth screens and the feedback form. |
| `error.tsx`, `global-error.tsx`, `not-found.tsx` | S/M/S | Error and 404 boundaries. |
| `globals.css` | **C** | Tailwind v4 design tokens: oklch palette, dark variants, brand blue, fonts. |

### Frontend Feature Components (`frontend/src/app/components/`)

**assistant/** — `ChatView.tsx` (**C**, conversation surface + side-panel tabs), `AssistantMessage.tsx` (**C**, markdown with citation chips, reasoning/tool blocks, edit accept/reject), `ChatInput.tsx` (**C**, attachments, workflow select, model toggle, key gating), `AssistantSidePanel.tsx` (M, drag-to-resize DocPanel tabs), `EditCard.tsx` (M, per-edit accept/reject + optimistic DOM resolution), `InitialView.tsx` (M), `ModelToggle.tsx` (M, MODELS catalog), `AddDocButton.tsx` (M), `UserMessage.tsx` (S), `SelectAssistantProjectModal.tsx` (M), `AssistantWorkflowModal.tsx` (M).

**projects/** — `ProjectPage.tsx` (**C**, the monolithic workspace), `ProjectExplorer.tsx` (**C**, folder tree with drag-and-drop and cycle prevention), `ProjectsOverview.tsx` (**C**, dashboard), `ProjectPageParts.tsx` (**C**, header/skeleton/version-rows building blocks), `ProjectAssistantTab.tsx` / `ProjectReviewsTab.tsx` (M, presentational tabs), `NewProjectModal.tsx` (M).

**tabular/** — `TabularReviewView.tsx` (**C**, orchestrates documents/columns/cells, streaming generation, Excel export), `TRTable.tsx` (**C**, the grid), `TabularCell.tsx` (**C**, citation markers + tag pills), `TRSidePanel.tsx` (**C**, full cell answer + regenerate), `TRChatPanel.tsx` (**C**, SSE review chat), `AddNewTRModal.tsx` (**C**, multi-step creation), `AddColumnModal.tsx` / `TREditColumnMenu.tsx` (**C**, column editing with AI prompt generation), plus utilities: `citation-utils.ts` (S, `<<n + quote>>` parsing), `columnFormat.ts` (S), `columnPresets.ts` (M), `pillUtils.ts` (M), `prompt-generator.ts` (M), `exportToExcel.ts` (M).

**workflows/** — `WorkflowList.tsx` (**C**, library with search/practice filters), `DisplayWorkflowModal.tsx` (**C**, preview + run into chat or tabular review), `NewWorkflowModal.tsx` (**C**), `WFEditColumnModal.tsx` (**C**) / `WFColumnViewModal.tsx` (M), `ShareWorkflowModal.tsx` (M), `WorkflowPromptEditor.tsx` (M, TipTap markdown editor), `builtinWorkflows.ts` (**C**, static template catalog), `practices.ts` (S).

**shared/** — `types.ts` (**C**, 53 importers — the shared vocabulary), `AppSidebar.tsx` (**C**, navigation + recent chats + account menu), `DocView.tsx` (**C**, pdfjs viewer with highlights and zoom), `DocxView.tsx` (**C**, docx-preview with tracked changes and edit-id stamping), `DocPanel.tsx` (**C**, routes a tab to DocxView/DocView with citation/edit headers), `FileDirectory.tsx` (**C**, multi-select document directory), `AddDocumentsModal.tsx` (**C**, attach-from-directory + upload), `PeopleModal.tsx` (**C**, sharing UI), `useDirectoryData.ts` (M, cached directory + invalidation), `highlightQuote.ts` / `highlightDocxQuote.ts` (M, fuzzy quote highlighting in PDF/DOCX), `RowActions.tsx` (M), `RenameableTitle.tsx` (M), `AddProjectDocsModal.tsx` (M), `UploadNewVersionModal.tsx` (M), `DocViewModal.tsx` (M), `ProjectPicker.tsx` (M), `EmailPillInput.tsx` (M), `PreResponseWrapper.tsx` (M), `OwnerOnlyModal.tsx` (M), `ApiKeyMissingModal.tsx` (M), `SidebarChatItem.tsx` (M), `HeaderSearchBtn.tsx` (M), `DocumentCard.tsx` / `ToolbarTabs.tsx` / `VersionChip.tsx` (S).

**modals/** — `credits-exhausted-modal.tsx`, `delete-chats-modal.tsx`, `simple-link-dialog.tsx` (all M).

### Frontend UI Primitives (`frontend/src/components/`)

`providers.tsx` (S, composition root), `site-logo.tsx` (M), `chat/mike-icon.tsx` (**C**, animated status glyph), `ui/button.tsx` (M), `ui/input.tsx` (S), `ui/badge.tsx` (S), `ui/dropdown-menu.tsx` (**C**, Radix suite), `ui/cite-button.tsx` (M), `ui/text-search-widget.tsx` (M, find-in-text).

### Frontend State & Hooks

| File | Cx | What it does |
|---|---|---|
| `src/contexts/AuthContext.tsx` | M | Supabase auth session tracking; 23 importers — the backbone of the authenticated UI. |
| `src/contexts/UserProfileContext.tsx` | **C** | Display name, credits, tier, tabular model, per-provider key state + save helpers. |
| `src/app/contexts/ChatHistoryContext.tsx` | M | Paginated chat list, optimistic updates, new-chat message handoff. |
| `src/app/contexts/SidebarContext.tsx` | S | Open/close setter for the app sidebar. |
| `src/app/hooks/useAssistantChat.ts` | **C** | Core chat hook: SSE streaming (standalone or project), drip-feed rendering, citations/tool events, create/rename, cancellation. |
| `src/app/hooks/useFetchDocxBytes.ts` | M | DOCX byte streaming with module cache + in-flight dedupe; exports `invalidateDocxBytes`. |
| `src/app/hooks/useFetchSingleDoc.ts` | M | Loads a document's display representation (PDF buffer or DOCX fallback signal). |
| `src/app/hooks/useDocumentVersions.ts` | M | Version history fetching with refresh trigger. |
| `src/app/hooks/useGenerateChatTitle.ts` | S | Best-effort backend title generation + rename. |
| `src/app/hooks/useSelectedModel.ts` | S | Persists selected model id to localStorage with validation. |

### Frontend Services & Utilities

| File | Cx | What it does |
|---|---|---|
| `src/app/lib/mikeApi.ts` | **C** | The API client (37 importers): projects, folders, documents/versions, SSE chats, tabular, workflows; attaches the Supabase token to every request. |
| `src/app/lib/modelAvailability.ts` | S | Maps model ids to providers; gates models on saved key status. |
| `src/lib/supabase.ts` | S | Shared Supabase browser client (sessions + bearer tokens). |
| `src/lib/auth.ts` | S | Server-side bearer-token validation helper. |
| `src/lib/storage.ts` | M | R2 utilities (upload/download/presigned URLs), gated by `storageEnabled`. |
| `src/lib/fileConverter.ts` | M | Upload prep: base64 data URLs, mammoth .docx text extraction, txt/md normalization. |
| `src/lib/utils.ts` | S | `cn` class merging + Dice-coefficient fuzzy matching. |
| `src/lib/types.ts`, `src/lib/label.ts`, `src/lib/slug.ts` | M/S/M | Statute-browsing API types and slug/label helpers (legacy of the statute-browsing surface). |
| `src/scripts/convert-courts-to-ts.js` | M | One-off build script regenerating typed court-data constants. |

### Configuration & Documentation

| File | What it does |
|---|---|
| `backend/.env.example` | Every backend dependency: Supabase service creds, R2 keys, Gemini/Anthropic/OpenAI/Resend keys, download HMAC secret, API-key encryption secret. |
| `frontend/.env.local.example` | Public Supabase URL/key + backend API base URL. |
| `backend/nixpacks.toml` | Adds LibreOffice to the deploy image for DOCX→PDF. |
| `backend/package.json` / `frontend/package.json` | Backend: Anthropic + Google GenAI SDKs, S3/R2, docx/mammoth/libreoffice-convert/jszip, Resend. Frontend: Next.js 16 + React 19 (React Compiler), TipTap, pdfjs-dist, mammoth, exceljs, OpenNext/wrangler. |
| `frontend/next.config.ts`, `open-next.config.ts`, `postcss.config.mjs`, `eslint.config.mjs`, `components.json`, `tsconfig.json` (×2) | Build/deploy/lint toolchain (Cloudflare Workers via OpenNext; Tailwind v4; shadcn new-york style; `@/*` aliases). |
| `README.md` | Setup: repo layout, prerequisites, schema bootstrap, env, run commands. |
| `CONTRIBUTING.md` | Small focused PRs, pre-PR build checks, private security reporting. |
| `docs/safe-local-testing.md` | How to develop without touching production: disposable resources, synthetic docs, secrets hygiene. |

---

## 6. Complexity Hotspots

51 of 190 file-level nodes are rated **complex**. These are the ones to approach with the most care:

### Backend

- **`backend/src/lib/chatTools.ts`** — the agentic chat engine. Touches everything: system prompt, four tool-schema sets, context building, tool execution, streaming. Most chat behavior changes land here; regressions ripple across standalone chat, project chat, and tabular chat.
- **`backend/src/lib/docxTrackedChanges.ts`** — hand-rolled OOXML manipulation with diff-anchored rewriting. Subtle: malformed XML or anchor drift corrupts legal documents. Test against real DOCX files before shipping changes.
- **`backend/src/lib/llm/openai.ts`** — raw-fetch SSE parsing rather than an SDK; the most fragile of the three adapters.
- **`backend/src/routes/chat.ts`, `documents.ts`, `projects.ts`, `tabular.ts`** — the four big routers; each mixes CRUD, access checks, storage, conversion, and (for chat/tabular) streaming LLM loops.
- **`backend/schema.sql`** — the single source of truth for fresh databases; remember the convention: edit schema.sql *and* write an incremental migration for existing databases.

### Frontend

- **`components/projects/ProjectPage.tsx`** — deliberately monolithic workspace driving three tabs and ~20 API calls. Understand its state flow before refactoring anything project-related.
- **`components/tabular/TabularReviewView.tsx`** — the highest fan-out component in the codebase: streaming cell generation, column CRUD, drag-drop, export, and two side panels.
- **`hooks/useAssistantChat.ts`** — SSE consumption, drip-feed timing, citation/tool-event tracking, cancellation. Chat UX bugs usually trace here.
- **`components/assistant/ChatView.tsx` + `AssistantMessage.tsx`** — auto-scroll, side-panel tab management, edit resolution, docx cache invalidation; markdown + citations + tool blocks rendering.
- **`components/shared/DocView.tsx` / `DocxView.tsx` / `DocPanel.tsx`** — pdfjs and docx-preview rendering with fuzzy quote highlighting and edit-id DOM stamping; sensitive to library versions and timing.
- **`components/shared/types.ts` and `app/lib/mikeApi.ts`** — not intrinsically hard, but with 53 and 37 importers respectively, any change here has the widest blast radius in the repo.

### Practical cautions

- **Cache invalidation pairs.** When you mutate documents or the directory, call `invalidateDirectoryCache` / `invalidateDocxBytes` — stale module-level caches are a recurring bug source.
- **Owner-only guards.** Shared resources (projects, reviews, workflows, chats) gate destructive actions to owners; replicate the guard pattern (`OwnerOnlyModal`) on any new mutating UI.
- **LibreOffice dependency.** DOCX→PDF requires LibreOffice locally and in the image (`nixpacks.toml`); conversion failures degrade viewers and tabular extraction.
- **Read `docs/safe-local-testing.md` first.** Use disposable Supabase/R2 resources and synthetic documents; never point local dev at production data.
