# CLAUDE.md - Resume Matcher

> **Context file for Claude Code.** Full documentation at [docs/agent/README.md](../docs/agent/README.md).

---

## Project Overview

Resume Matcher is an AI-powered application for tailoring resumes to job descriptions.

| Layer | Stack |
|-------|-------|
| **Backend** | FastAPI + Python 3.13+, LiteLLM (multi-provider AI) |
| **Frontend** | Next.js 16 + React 19, Tailwind CSS v4 |
| **Database** | TinyDB (JSON file storage) |
| **PDF** | Headless Chromium via Playwright |
| **Package Management** | npm (frontend), uv (backend) |
| **Languages** | en, es, zh, ja, pt-BR |

---

## First Steps

**Before exploring code, read the [navigator skill](/.claude/skills/navigator/SKILL.md)** for codebase orientation.

---

## Skills-First Workflow

**Before starting any task, identify which skills apply and invoke them.** This is mandatory, not optional.

### How to Use Skills

1. **Read the task** — understand what the user is asking.
2. **Scan the catalog below** — identify every skill that could apply.
3. **Invoke the matching skills** (via the Skill tool) before writing any code or giving answers.
4. **Follow skill instructions** — each skill has its own process; respect it.

### Skills Catalog

| Skill | Invoke command | When to use |
|-------|---------------|-------------|
| **using-superpowers** | `/using-superpowers` | At conversation start — establishes skill discovery and usage |
| **navigator** | `/navigator` | First step when exploring code, finding files, or understanding project structure |
| **codebase-navigator** | `/codebase-navigator` | Advanced code search with ripgrep — find functions, classes, endpoints, trace data flows |
| **brainstorming** | `/brainstorming` | Before any creative work — creating features, building components, adding functionality |
| **simple** | `/simple` | Lightweight brainstorming for small/medium creative or architectural decisions |
| **writing-plans** | `/writing-plans` | When you have a spec or requirements for a multi-step task, before touching code |
| **systematic-debugging** | `/systematic-debugging` | When encountering any bug, test failure, or unexpected behavior — before proposing fixes |
| **code-review** | `/code-review` | When receiving code review feedback — verify before implementing, don't blindly agree |
| **requesting-code-review** | `/requesting-code-review` | After completing tasks or major features — verify work meets requirements before merging |
| **design-principles** | `/design-principles` | When designing new UI components or modifying existing component styles (Swiss International Style) |
| **react-patterns** | `/react-patterns` | React/Next.js performance optimization — local/offline or Docker-deployed apps |
| **nextjs-performance** | `/nextjs-performance` | Next.js 15 critical fixes — components, data fetching, Server Actions, bundle size |
| **tailwind-pattern** | `/tailwind-pattern` | Tailwind CSS patterns — layouts, cards, navigation, forms, buttons, typography |
| **fastapi** | `/fastapi` | Python APIs with FastAPI, Pydantic v2, JWT auth — prevents 7 documented errors |

### Task-to-Skills Mapping

| Task type | Skills to invoke (in order) |
|-----------|----------------------------|
| **New feature** | brainstorming (or simple) → writing-plans → navigator → [frontend/backend skills] → requesting-code-review |
| **Bug fix** | systematic-debugging → codebase-navigator → [frontend/backend skills] → requesting-code-review |
| **Frontend UI work** | design-principles → react-patterns → nextjs-performance → tailwind-pattern |
| **Backend API work** | fastapi → codebase-navigator |
| **Code exploration** | navigator → codebase-navigator |
| **Responding to review** | code-review |
| **Multi-step implementation** | writing-plans → [relevant skills per step] |

---

## Non-Negotiable Rules

1. **All frontend UI changes** MUST follow [Swiss International Style](../docs/agent/design/style-guide.md)
2. **All Python functions** MUST have type hints
3. **Run `npm run lint`** before committing frontend changes
4. **Run `npm run format`** (Prettier) before committing
5. **Log detailed errors server-side**, return generic messages to clients
6. **Do NOT modify** `.github/workflows/` files without explicit request

---

## Essential Commands

```bash
# Install all dependencies
npm run install

# Development (both servers)
npm run dev

# Individual servers
npm run dev:backend   # FastAPI on :8000
npm run dev:frontend  # Next.js on :3000

# Quality checks
npm run lint          # Lint frontend
npm run format        # Format with Prettier

# Testing
npm run test          # Frontend tests (Vitest)
cd apps/backend && uv run pytest  # Backend tests

# Build
npm run build
```

---

## Project Structure

```
apps/
├── backend/                    # FastAPI + Python 3.13+
│   ├── app/
│   │   ├── main.py             # Entry point, CORS, lifespan, router includes
│   │   ├── config.py           # Settings (LLM, server, CORS, API key persistence)
│   │   ├── database.py         # TinyDB wrapper (resumes, jobs, improvements tables)
│   │   ├── llm.py              # LiteLLM wrapper (multi-provider, JSON extraction)
│   │   ├── pdf.py              # Playwright PDF renderer (lazy-init, system Chrome detect)
│   │   ├── routers/
│   │   │   ├── health.py       # GET /health, GET /status
│   │   │   ├── resumes.py      # CRUD + improve + PDF + cover letter + outreach
│   │   │   ├── jobs.py         # Job description upload and retrieval
│   │   │   ├── config.py       # LLM config, feature flags, language, prompts, API keys
│   │   │   └── enrichment.py   # AI enrichment (feature flag controlled)
│   │   ├── services/
│   │   │   ├── parser.py       # PDF/DOCX→markdown, markdown→structured JSON
│   │   │   ├── improver.py     # Resume tailoring, keyword extraction, diff calculation
│   │   │   ├── refiner.py      # Multi-pass refinement (keyword injection, AI phrase removal, alignment)
│   │   │   └── cover_letter.py # Cover letter, outreach message, title generation
│   │   ├── schemas/
│   │   │   ├── models.py       # Core Pydantic v2 models (ResumeData, responses, diffs)
│   │   │   ├── enrichment.py   # Enrichment question/answer schemas
│   │   │   └── refinement.py   # RefinementConfig, alignment report models
│   │   └── prompts/
│   │       ├── templates.py    # Resume/improve schema examples, prompt templates
│   │       ├── refinement.py   # Keyword injection, AI phrase removal, alignment prompts
│   │       └── enrichment.py   # Enrichment feature prompts
│   ├── data/                   # TinyDB storage (database.json, config.json)
│   └── pyproject.toml          # Python dependencies (uv)
│
└── frontend/                   # Next.js 16 + React 19
    ├── app/
    │   ├── (default)/
    │   │   ├── page.tsx        # Home/landing page
    │   │   ├── dashboard/      # Resume and job listing
    │   │   ├── builder/        # Resume editor with drag-and-drop
    │   │   ├── tailor/         # Job description + tailoring interface
    │   │   ├── resumes/[id]/   # Resume detail view
    │   │   └── settings/       # LLM config, feature flags, language
    │   └── print/
    │       ├── resumes/[id]/   # Resume PDF rendering page
    │       └── cover-letter/[id]/ # Cover letter PDF rendering page
    ├── components/
    │   ├── builder/            # Resume builder (forms, drag-and-drop, template selector)
    │   ├── resume/             # Resume templates (modern, single-column, two-column)
    │   ├── ui/                 # Reusable primitives (button, input, dialog, rich-text-editor)
    │   ├── enrichment/         # AI enrichment modal (question flow, preview)
    │   ├── tailor/             # Diff preview modal
    │   ├── settings/           # API key menu
    │   ├── dashboard/          # Resume/job listing
    │   ├── home/               # Hero section, feature grid
    │   ├── preview/            # Print/preview components
    │   └── common/             # Error boundary, resume previewer context
    ├── lib/
    │   ├── api/                # API client (client.ts, resume.ts, jobs.ts, enrichment.ts)
    │   ├── i18n/               # Translation utilities, locale detection
    │   ├── types/              # TypeScript types (template-settings.ts)
    │   ├── utils/              # section-helpers, keyword-matcher, html-sanitizer, download
    │   ├── context/            # language-context.tsx, status-cache.tsx
    │   ├── config/             # version.ts
    │   └── constants/          # page-dimensions.ts
    ├── hooks/                  # Custom React hooks
    ├── messages/               # i18n translations (en, es, zh, ja, pt-BR)
    └── package.json            # Node dependencies
```

---

## API Routes Reference

All routes prefixed with `/api/v1`:

### Health
| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/health` | LLM connectivity + basic health |
| `GET` | `/status` | Full status: LLM config, master resume, DB stats |

### Resumes
| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/resumes/upload` | Upload PDF/DOCX → async parse to JSON |
| `GET` | `/resumes/{id}` | Fetch resume + cover letter + outreach |
| `GET` | `/resumes/list` | List all resumes (opt: include master) |
| `DELETE` | `/resumes/{id}` | Delete resume |
| `POST` | `/resumes/improve/preview` | Preview tailored resume (not persisted) |
| `POST` | `/resumes/improve/confirm` | Persist previewed tailored resume |
| `POST` | `/resumes/improve` | Preview + confirm in one call |
| `GET` | `/resumes/{id}/pdf` | Generate PDF with template options |
| `GET` | `/resumes/{id}/cover-letter/pdf` | Generate cover letter PDF |
| `POST` | `/resumes/{id}/generate-cover-letter` | On-demand cover letter |
| `POST` | `/resumes/{id}/generate-outreach` | On-demand outreach message |
| `GET` | `/resumes/{id}/job-description` | Get original JD for tailored resume |
| `PATCH` | `/resumes/{id}` | Update structured resume data |
| `PATCH` | `/resumes/{id}/cover-letter` | Update cover letter text |
| `PATCH` | `/resumes/{id}/outreach-message` | Update outreach message |
| `PATCH` | `/resumes/{id}/title` | Update title (max 80 chars) |
| `POST` | `/resumes/{id}/retry-processing` | Retry failed/stuck processing |

### Jobs
| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/jobs/upload` | Upload one or more job descriptions |
| `GET` | `/jobs/{id}` | Get job description + cached keywords |

### Config
| Method | Path | Description |
|--------|------|-------------|
| `GET/POST` | `/config/llm-api-key` | Get/set LLM provider, model, key |
| `GET/POST` | `/config/feature-flags` | Get/set cover letter + outreach flags |
| `GET/POST` | `/config/language` | Get/set content language |
| `GET/POST` | `/config/prompt-options` | Get available prompts / set default |
| `GET/POST` | `/config/api-keys` | Get all keys (masked) / update keys |
| `DELETE` | `/config/api-keys/{provider}` | Remove provider key |
| `POST` | `/config/reset-database` | Wipe all resumes, jobs, improvements |

---

## Data Models

### ResumeData (core schema)
```python
ResumeData:
├── personalInfo: {name, email, phone, location, links[]}
├── summary: str
├── workExperience[]: {company, position, years, description[], metadata?}
├── education[]: {institution, degree, years, description[]}
├── personalProjects[]: {name, description[], years?}
├── additional: {technicalSkills[], certificationsTraining[], languages[], awards[]}
└── customSections: dict[str, CustomSection]  # user-defined, LLM-protected
```

### Database Tables (TinyDB)
```
resumes: id, content, content_type, processed_data, is_master, parent_id,
         processing_status, cover_letter, outreach_message, title,
         original_markdown, created_at, updated_at

jobs: id, content, resume_id?, job_keywords, job_keywords_hash,
      preview_hash, preview_hashes{}, created_at

improvements: id, original_resume_id, tailored_resume_id, job_id,
              improvements[], created_at
```

---

## Resume Tailoring Pipeline

Understanding this flow is critical for working on the improve/refine code:

```
1. Extract job keywords (LLM)
2. Improve resume with selected prompt (LLM)
3. Preserve: personalInfo, dates, skills, custom sections
4. Multi-pass refinement (services/refiner.py):
   a. Keyword injection — insert missing job keywords
   b. AI phrase removal — remove AI-sounding language
   c. Alignment validation — ensure consistency with master
5. Parallel generation: cover letter + outreach message + title
6. Calculate diff (original vs tailored)
7. Calculate keyword match percentage
8. Persist (on confirm)
```

---

## LLM Integration

**Supported providers** (via LiteLLM): `openai`, `anthropic`, `gemini`, `openrouter`, `deepseek`, `ollama`

**Timeouts**: health=30s, completion=120s, JSON=180s

**Safety limits**: JSON depth max 10 levels, content max 1MB

**Environment variables**:
```bash
LLM_PROVIDER=openai          # or anthropic, gemini, openrouter, deepseek, ollama
LLM_MODEL=gpt-4o             # model identifier
LLM_API_KEY=sk-...           # API key
LLM_API_BASE=                # optional, for Ollama or custom proxy endpoints
```

API keys can also be set at runtime via `POST /api/v1/config/api-keys` and are persisted to `apps/backend/data/config.json`.

---

## Documentation by Task

### For Backend Changes
1. [Backend guide](../docs/agent/architecture/backend-guide.md) - Architecture, modules, services
2. [API contracts](../docs/agent/apis/front-end-apis.md) - API specifications
3. [LLM integration](../docs/agent/llm-integration.md) - Multi-provider AI support

### For Frontend Changes
1. [Frontend workflow](../docs/agent/architecture/frontend-workflow.md) - User flow, components
2. [Style guide](../docs/agent/design/style-guide.md) - **REQUIRED** Swiss International Style
3. [Coding standards](../docs/agent/coding-standards.md) - Frontend conventions

### For Template/PDF Changes
1. [PDF template guide](../docs/agent/design/pdf-template-guide.md) - PDF rendering
2. [Template system](../docs/agent/design/template-system.md) - Resume templates
3. [Resume templates](../docs/agent/features/resume-templates.md) - Template types & controls

### For Features
| Feature | Documentation |
|---------|---------------|
| Custom sections | [custom-sections.md](../docs/agent/features/custom-sections.md) |
| Resume templates | [resume-templates.md](../docs/agent/features/resume-templates.md) |
| i18n | [i18n.md](../docs/agent/features/i18n.md) |
| AI enrichment | [enrichment.md](../docs/agent/features/enrichment.md) |
| JD matching | [jd-match.md](../docs/agent/features/jd-match.md) |

---

## Code Patterns

### Backend Error Handling
```python
except Exception as e:
    logger.error(f"Operation failed: {e}")
    raise HTTPException(status_code=500, detail="Operation failed. Please try again.")
```

### Frontend Textarea Fix
All textareas need Enter key handling:
```tsx
const handleKeyDown = (e: React.KeyboardEvent<HTMLTextAreaElement>) => {
  if (e.key === 'Enter') e.stopPropagation();
};
```

### Mutable Defaults (Python)
Always use `copy.deepcopy()` for mutable defaults:
```python
import copy
data = copy.deepcopy(DEFAULT_DATA)  # Correct
# data = DEFAULT_DATA  # Wrong - shared state bug
```

### Async Concurrency (Python)
Critical sections use `asyncio.Lock` — do not remove or bypass:
```python
# Used in: database.py (master resume), pdf.py (renderer init)
async with self._lock:
    # atomic operation here
```

### Personal Info & Date Preservation
The tailoring pipeline always restores `personalInfo` and dates from the original markdown. Never allow LLM to overwrite these fields — see `services/improver.py` and `services/parser.py` (`restore_dates_from_markdown`).

### Custom Section Protection
Custom sections are user-defined and must not be modified by the LLM. If adding LLM operations that touch resume data, explicitly skip `customSections`.

---

## Design System Quick Reference

| Element | Value |
|---------|-------|
| Canvas background | `#F0F0E8` |
| Ink (text) | `#000000` |
| Hyper Blue (links) | `#1D4ED8` |
| Signal Green (success) | `#15803D` |
| Alert Orange (warning) | `#F97316` |
| Alert Red (error) | `#DC2626` |
| Headers font | `font-serif` |
| Body font | `font-sans` |
| Metadata font | `font-mono` |
| Borders | `rounded-none`, 1px black, hard shadows |

---

## Definition of Done

Before completing a task:

- [ ] Code compiles without errors
- [ ] `npm run lint` passes
- [ ] UI changes follow Swiss International Style
- [ ] Python functions have type hints
- [ ] Schema/prompt changes documented

---

## Out of Scope

Do NOT modify without explicit request:
- `.github/workflows/` files
- CI/CD configuration
- Docker build behavior
- Existing tests (removal/disabling)

---

> **Full agent documentation**: [docs/agent/README.md](../docs/agent/README.md)
