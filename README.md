# JobHunt AI — System Design Document
*AI-Powered Job Search, Application & Referral Agent for Software Developers*

---

## Executive Summary

JobHunt AI is an agentic system that collapses three painful, time-consuming loops in the modern job search:
1. **Discovery** — scanning 100+ company portals and job boards
2. **Application** — filling repetitive forms across different systems
3. **Networking** — finding and reaching warm referrals at target companies

The system runs a local or API-backed LLM as its reasoning engine, exposes a clean dashboard, and uses browser automation + structured data pipelines to do the grunt work for you.

---

## 1. Problem Statement

| Pain Point | Current Time Cost | With JobHunt AI |
|---|---|---|
| Searching across 100+ company portals | 3–5 hrs/day | 5–10 min review |
| Filling out long job application forms | 20–40 min/application | 1-click autofill |
| Finding referrals at target companies | 1–2 hrs of LinkedIn digging | Automated, ranked list |
| Tailoring resume/cover letter per role | 30–60 min | AI-generated in seconds |
| Tracking what you applied to | Spreadsheet chaos | Unified dashboard |

---

## 2. High-Level Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                        JobHunt AI System                        │
│                                                                  │
│  ┌──────────────┐    ┌───────────────────────────────────────┐  │
│  │   Dashboard  │    │            Agent Core (LLM)           │  │
│  │  (React SPA) │◄───│  claude-sonnet-4-6 / local Ollama    │  │
│  └──────┬───────┘    │  (Orchestrates all sub-agents)        │  │
│         │            └───────────────┬───────────────────────┘  │
│         │                            │                          │
│  ┌──────▼───────────────────────────▼────────────────────────┐  │
│  │                    Tool Layer (MCP / REST)                 │  │
│  │  ┌──────────────┐  ┌────────────────┐  ┌───────────────┐  │  │
│  │  │ Job Scraper  │  │  Form Filler   │  │ Referral Finder│  │  │
│  │  │  Sub-Agent   │  │  Sub-Agent     │  │   Sub-Agent   │  │  │
│  │  └──────┬───────┘  └───────┬────────┘  └──────┬────────┘  │  │
│  └─────────┼──────────────────┼──────────────────┼───────────┘  │
│            │                  │                  │              │
│  ┌─────────▼──┐   ┌───────────▼──┐   ┌───────────▼──────────┐  │
│  │  Job Board │   │  Browser     │   │  LinkedIn / GitHub   │  │
│  │  APIs +    │   │  Automation  │   │  / Twitter Scraper   │  │
│  │  Scrapers  │   │  (Playwright)│   │  + Email Finder      │  │
│  └────────────┘   └──────────────┘   └──────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Local Storage (SQLite + ChromaDB)           │   │
│  │   Jobs | Applications | Referrals | Your Profile        │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────┘
```

---

## 3. Core Modules

### 3.1 Job Discovery Sub-Agent

**What it does:** Continuously crawls job boards and company career portals. Matches listings to your profile using semantic similarity (embeddings), not just keyword matching.

**Sources (in priority order):**
- LinkedIn Jobs API / scraper
- Greenhouse.io, Lever, Workday, Ashby (ATS portals — used by 80%+ of tech cos)
- Indeed, Glassdoor, Naukri (for India), Cutshort
- Company-specific careers pages (via sitemap crawling + Playwright)
- Wellfound (AngelList) for startups
- Y Combinator Jobs board

**Matching logic:**
```
score = 0.4 × semantic_similarity(job_desc, your_profile)
      + 0.2 × keyword_overlap(required_skills, your_skills)
      + 0.2 × title_match(job_title, target_roles)
      + 0.1 × exp_range_fit(required_yoe, your_yoe)
      + 0.1 × company_score(company_culture_fit, your_prefs)
```

**Tech Stack:**
- `Playwright` — headless browser for JS-heavy portals
- `BeautifulSoup` + `httpx` — fast static page scraping
- `sentence-transformers` (all-MiniLM-L6-v2) — local embeddings for similarity
- `ChromaDB` — vector store for job descriptions
- `SQLite` — structured job metadata storage

---

### 3.2 Autofill Sub-Agent

**What it does:** When you click "Apply" on any job in the dashboard, a browser automation session opens the portal and fills all form fields using your stored profile.

**Your profile includes:**
- Personal: name, phone, email, LinkedIn URL, GitHub, portfolio
- Resume: parsed sections (experience, education, skills, projects)
- Dynamic: AI-generated cover letter tailored to each job, answers to common screening questions

**Field detection strategy:**
1. Scan form for field `name`, `id`, `aria-label`, `placeholder` attributes
2. Map to profile fields using LLM (handles "Current CTC", "Notice Period", "Why us?" etc.)
3. For free-text questions (e.g., "Describe a challenging project") — call LLM to generate contextual answer from your experience
4. For file uploads (resume, cover letter) — auto-generate tailored PDF and upload

**Supported portals:** Greenhouse, Lever, Workday, SAP SuccessFactors, iCIMS, Ashby, Rippling, and custom forms

**Human-in-the-loop checkpoints:**
- You review and confirm before final submit
- Flags any question it isn't confident about
- Shows diff vs. your last application so you catch inconsistencies

---

### 3.3 Referral Finder Sub-Agent

**What it does:** Once you've identified a company you want to apply to, this agent finds people in your network (1st, 2nd degree) and likely advocates who work there.

**Data sources:**
- LinkedIn (profile scraper — your own session cookie)
- GitHub org members
- Twitter/X profile bios
- Alumni databases (via college LinkedIn groups)
- Hunter.io / Apollo.io API — for verified work email addresses

**Scoring referrals:**
```
referral_score = 0.3 × connection_degree(1st > 2nd > 3rd)
               + 0.25 × role_relevance(engineering_manager > SWE > others)
               + 0.2 × common_background(same_college, city, prev_company)
               + 0.15 × engagement_signals(mutual_connections, shared_posts)
               + 0.1 × tenure_at_company(longer = more influence)
```

**Output for each referral:**
- Name, title, LinkedIn URL
- Connection path ("Friend of X who you know from Y")
- Suggested intro message (personalized by LLM)
- Work email (when found via Hunter.io)
- Common ground summary

---

### 3.4 Dashboard (React SPA)

**Core views:**
1. **Job Feed** — Filterable table of matched jobs (company, role, match %, salary, deadline)
2. **Company Tracker** — Kanban: Saved → Applied → Interviewing → Offer/Rejected
3. **Application Detail** — Job description, your tailored resume, referrals, notes
4. **Profile Setup** — Your resume, preferences, autofill data
5. **Referral Outreach** — Draft messages, track who you've contacted

**Filters available:**
- Company name / industry / size
- Job title / role type (SWE, backend, fullstack, etc.)
- Years of experience required
- Salary range
- Remote / hybrid / onsite
- Match score threshold
- Application deadline
- Tech stack keywords

---

## 4. LLM Strategy

### Option A: API-based (Recommended to start)
- **Model:** `claude-sonnet-4-6` via Anthropic API
- **Cost:** ~$0.003/1K tokens. A full application cycle (job parsing + autofill + cover letter + referral messages) costs roughly ₹2–5 per application
- **Pros:** Best quality, no setup, handles complex reasoning
- **Cons:** Requires internet, API cost at scale

### Option B: Local LLM (Privacy-first)
- **Model:** `Mistral-7B-Instruct` or `Llama-3-8B` via `Ollama`
- **Use for:** Form field mapping, basic text generation, embedding
- **Pros:** Fully offline, free, private
- **Cons:** Slower, less coherent for creative tasks (cover letters)

### Hybrid Recommended:
```
Local (Ollama):     field detection, form mapping, keyword extraction
Anthropic API:      cover letter writing, "Why us?" answers, referral messages
Embeddings:         sentence-transformers (local, fast, free)
```

---

## 5. Tech Stack (Full)

| Layer | Technology |
|---|---|
| Frontend | React 18, TailwindCSS, Zustand, React Query |
| Backend API | FastAPI (Python) |
| Agent Orchestration | LangGraph or CrewAI |
| Browser Automation | Playwright (Python) |
| LLM (cloud) | Claude Sonnet via Anthropic SDK |
| LLM (local) | Ollama + Mistral/Llama3 |
| Embeddings | sentence-transformers (MiniLM-L6-v2) |
| Vector DB | ChromaDB (local) |
| Structured DB | SQLite (dev) → PostgreSQL (prod) |
| Job Data | Scrapy + Playwright + Apify (optional) |
| Referral Data | LinkedIn scraper, Hunter.io API, GitHub API |
| Resume Parsing | PyMuPDF + LLM extraction |
| PDF Generation | ReportLab / WeasyPrint |
| Auth | Local auth (single user, no cloud needed) |
| Deployment | Docker Compose (runs on your laptop) |

---

## 6. Data Model (SQLite Schema)

```sql
-- Core entities
jobs(id, company, title, url, jd_text, jd_embedding, salary_min, salary_max,
     yoe_required, location, remote_ok, tech_stack[], match_score,
     source, scraped_at, deadline)

applications(id, job_id, status, applied_at, resume_version, cover_letter,
             notes, form_data_json, referral_id)

companies(id, name, size, industry, culture_tags[], careers_url, ats_type)

referrals(id, job_id, company_id, name, title, linkedin_url, email,
          connection_degree, score, outreach_status, message_sent)

your_profile(id, full_name, email, phone, linkedin, github, portfolio,
             resume_pdf_path, resume_parsed_json, preferences_json)
```

---

## 7. Privacy & Ethics Guidelines

- LinkedIn scraping uses YOUR login session — no credential sharing
- Never submit applications without your explicit confirm click
- Autofill data stored locally only (not in any cloud)
- Referral cold outreach messages are always shown to you before sending
- Rate limiting on scrapers to avoid account bans (max 100 req/hr per source)
- Option to mark companies as "never apply" (competitors, ethical concerns)

---

## 8. What Actually Gets You Hired (HR + AI Expert Perspective)

### The Brutal Truth About Application Volume vs. Quality

Mass-applying blindly (100 apps with generic resumes) has a < 1% response rate. Targeted applications with warm referrals have a 5–15× higher response rate. This system optimizes for quality + referrals, not just volume.

### What Actually Moves the Needle:

**1. Referrals > Everything Else**
- 35–40% of hires at top tech companies come via referrals
- A referral submission gets reviewed — a cold apply may be filtered by ATS before a human sees it
- Even a 2nd-degree connection messaging you "happy to forward your resume" can 10× your chances

**2. Resume ATS Optimization**
- Use the exact skill keywords from the JD (verbatim, not synonyms)
- Quantify impact: "Reduced API latency by 40%" beats "improved performance"
- One-page for < 5 YOE; two pages max for senior roles
- No tables, columns, or graphics in the resume file (ATS parsers break on them)

**3. Tailored Cover Letters Work (When Done Right)**
- Generic: "I am excited to apply for the role at [Company]..." → deleted
- Good: References a specific product, engineering blog post, or problem the company is solving
- This system auto-generates tailored letters — your job is to add one personal sentence

**4. Application Timing**
- Apply within 48 hours of posting — many roles fill or close screening in days
- Tuesday–Thursday, 9–11 AM local time = highest recruiter activity
- This system alerts you to fresh postings and sorts by age

**5. The "Dream List" Strategy**
- Identify 20–30 target companies you genuinely want to work at
- Build referral network there before roles open
- Set alerts for their job pages — apply the moment relevant roles appear
- This beats frantically applying everywhere when you "need a job now"

**6. Skills That Currently Get Interviews (2025–2026 India/Global Market)**
- LLM integration / AI features (even basic: RAG, embeddings, agents)
- System design fluency (can explain distributed systems clearly)
- Any of: Rust, Go, or deep TypeScript
- Cloud-native: Kubernetes, Terraform, observability (not just "I've used AWS")
- Data engineering pipelines if targeting fintech/analytics-heavy companies

**7. The Interview Prep Flywheel**
- Use the job descriptions collected by this system to extract recurring interview topics
- LLM can generate mock interview questions from JDs + company engineering blogs
- Track which topics come up most across your target companies

---

## 9. Implementation Phases

### Phase 1 — Core (Weeks 1–3)
- [ ] Profile setup (resume upload + parsing)
- [ ] Manual job URL → parse + save to DB
- [ ] Basic dashboard with job table + filters
- [ ] Autofill for Greenhouse / Lever forms

### Phase 2 — Discovery (Weeks 4–6)
- [ ] Scrapers for LinkedIn, Indeed, Naukri, Wellfound
- [ ] Semantic matching + match score display
- [ ] Daily digest email with new matching jobs

### Phase 3 — Referrals (Weeks 7–9)
- [ ] LinkedIn 1st/2nd degree search for company
- [ ] Referral scoring + ranking
- [ ] AI-generated outreach message drafts

### Phase 4 — Intelligence (Weeks 10–12)
- [ ] AI cover letter generation (tailored per role)
- [ ] Interview question extraction from JDs
- [ ] Application analytics (response rates by company/role/channel)

---

## 10. Running It Locally

```bash
# Prerequisites: Python 3.11+, Node 20+, Ollama (optional)

# 1. Clone and install
git clone https://github.com/you/jobhunt-ai
cd jobhunt-ai
pip install -r requirements.txt
cd frontend && npm install

# 2. Set your API key (only if using cloud LLM)
export ANTHROPIC_API_KEY=sk-ant-...
export HUNTER_API_KEY=...  # optional, for email finding

# 3. Upload your resume (one-time setup)
python setup.py --resume path/to/resume.pdf

# 4. Start all services
docker-compose up  # or: make dev

# 5. Open dashboard
# http://localhost:3000
```

---

## 11. Cost Estimate (Monthly, Solo Developer)

| Service | Usage | Cost |
|---|---|---|
| Anthropic API | ~500 applications/month | ~₹800–1,200 |
| Hunter.io | 25 email lookups/month | Free tier |
| Apify (optional scraping) | — | Free tier (100 runs) |
| LinkedIn | Your existing account | ₹0 |
| Hosting | Runs on your laptop | ₹0 |
| **Total** | | **~₹800–1,200/month** |

---

*Document version 1.0 — Designed for a software developer targeting SWE/backend/fullstack roles*
