---
name: process-analysis
description: Research and document a CLFL business process from wiki, Jira, and GitHub sources. Produces a structured process description with AS-IS/TO-BE diagrams, prerequisites, and ABAP implementation map, then saves the result as a Word document.
license: private
---

# Process Analysis Skill

You are performing a structured process analysis for the CLFL Fulfillment Solutions team. Your output is a Word document (`.docx`) in `docs/` and an inline markdown summary.

## Authentication

All SAP systems require SSO as **diana.dekret@sap.com** (C-user: **C5283308**). Follow these procedures per system — never prompt the user for credentials.

### Wiki (`wiki.one.int.sap`)

The Playwright MCP browser (msedge) maintains an SSO session automatically.

**Check first:** Try navigating directly. If the page loads (not the Microsoft login page), the session is active — proceed.

**Session expired or first run:**
1. Navigate to `https://wiki.one.int.sap/`
2. If redirected to `login.microsoftonline.com`, look for a "Pick an account" prompt — click the `diana.dekret@sap.com` entry
3. If an MFA prompt appears (showing a number like "94"), inform the user: *"MFA approval needed — please approve request number XX in your Authenticator app"* and wait with `browser_wait_for`
4. After login succeeds, proceed with the research — the session persists for the remainder of the conversation

**Session detection:** If `browser_snapshot` shows `Page Title: Sign in to your account`, the session has expired. Re-authenticate as above.

---

### GitHub (`github.tools.sap`)

Uses a **Personal Access Token (PAT)** — no browser auth needed.

**Read the PAT at runtime:**
```bash
# Extract token from auth store (never hardcode)
PAT=$(node -e "const d=require('C:/Users/C5283308/.sap-mcp/auth.json'); console.log(d.providers['github.tools.sap'].token)")
```
Or read the file directly and parse the `providers["github.tools.sap"].token` field.

**Use in every curl request:**
```bash
curl -s -H "Authorization: token <PAT>" -H "Accept: application/vnd.github.v3.raw" "<URL>"
```

**If the PAT is expired or invalid** (HTTP 401 response): inform the user — *"GitHub PAT for github.tools.sap appears expired. Please regenerate it at https://github.tools.sap/settings/tokens and store it via the sap-auth-mcp tool."* Continue with wiki + Jira data only.

---

### Jira (`jira.tools.sap`)

**Verified (2026-06-14):**
- `sap-jira` MCP — **authenticated and working**. Returns HTTP 429 on rapid calls (rate limit), not auth errors. Always use this.
- Playwright SSO via Edge — **does not work**. Conditional Access detects the embedded browser, drops the session mid-MFA flow (page closes), and leaves a 401. Do not attempt.

**Rate limiting:** If `sap-jira` returns HTTP 429, wait a few seconds and retry. Never treat 429 as an auth failure.

**Strategy:**

1. **Use `sap-jira` MCP directly** (primary) — call `mcp__sap-jira__get_issue`, `mcp__sap-jira__search_issues`, etc.
2. **Extract ticket keys from wiki** (supplementary) — wiki pages embed ITSCLFL-XXXXX / ARTFS-XXX inline; use these to seed Jira lookups
3. **Never attempt Playwright for Jira** — session never establishes due to Conditional Access
4. **Never block** on Jira — if MCP returns persistent errors, proceed with wiki + GitHub data and note the gap in Sources

---

### SAP Help Portal (`help.sap.com`) and SAP Community (`community.sap.com`)

Public sites — no authentication required. Access directly via Playwright or `mcp__sap-wiki__general_search`.

---

### MS Teams (`teams.cloud.microsoft`)

Uses an OAuth token stored in `C:/Users/C5283308/.sap-mcp/auth.json` under `providers["teams.cloud.microsoft"]`. Access via `mcp__sap-msteams__*` tools. If the token is expired, inform the user and skip Teams as a source.

---

## Default Sources

Always search these sources unless the user explicitly excludes them:

| # | Source | How to access |
|---|--------|---------------|
| 1 | **CLFL Wiki** (`wiki.one.int.sap`, space `CLFL`) | Playwright browser (msedge) — already authenticated via SSO. Navigate to `https://wiki.one.int.sap/wiki/dosearchsite.action?queryString=<TOPIC>&spaceKey=CLFL`, then use `browser_evaluate` to extract `#main-content`. |
| 2 | **ITSCLFL Jira** (`jira.tools.sap`) | Use `sap-jira` MCP directly — authenticated and working (verified 2026-06-14). Call `mcp__sap-jira__get_issue` or `mcp__sap-jira__search_issues`. If HTTP 429, wait and retry. Use wiki-embedded ticket keys as supplementary seeds for Jira lookups. |
| 3 | **ABAP Documentor GitHub** (`github.tools.sap/C5403764/abap-documentor`) | Use `curl` with the PAT from `C:/Users/C5283308/.sap-mcp/auth.json` key `github.tools.sap`. Raw content: `curl -s -H "Authorization: token <PAT>" -H "Accept: application/vnd.github.v3.raw" "https://github.tools.sap/api/v3/repos/C5403764/abap-documentor/contents/<path>"`. File tree: `https://github.tools.sap/api/v3/repos/C5403764/abap-documentor/git/trees/main?recursive=1`. |

## Extra Sources (user-provided)

If the user passes additional sources in the skill invocation arguments, append them to the search plan before starting. Format expected from user:

```
/process-analysis <process topic> [+ <source description or URL>] [+ <source description or URL>]
```

Examples the user might provide:
- A specific wiki page URL
- A Jira filter URL or JQL query
- A different GitHub repo path
- A Teams conversation ID (`mcp__sap-msteams__teams_web_messages`)
- A SAP Help portal search term

Parse any text after the process topic that is preceded by `+` as an additional source. Include each extra source in the research phase.

## Execution Protocol

### Phase 1 — Parse Input

Extract from the invocation arguments:
1. **Process topic** — the main subject (e.g., "Move to URM: CA calculation and ES update")
2. **Extra sources** — any `+ <source>` entries appended by the user

Report the parsed topic and source list before starting research.

### Phase 2 — Research (parallel where possible)

For each source (defaults + extras), run searches in parallel:

**Wiki search** — see [Authentication → Wiki] for session handling:
1. Navigate to `https://wiki.one.int.sap/wiki/dosearchsite.action?queryString=<ENCODED_TOPIC>&spaceKey=CLFL`
2. Collect all result page URLs from the snapshot
3. Fetch the top 3–5 most relevant pages using `browser_navigate` + `browser_evaluate(() => document.querySelector('#main-content')?.innerText)`

**GitHub search** — see [Authentication → GitHub] for PAT retrieval:
1. Read PAT from `C:/Users/C5283308/.sap-mcp/auth.json` → `providers["github.tools.sap"].token`
2. Get file tree: `curl -s -H "Authorization: token <PAT>" "https://github.tools.sap/api/v3/repos/C5403764/abap-documentor/git/trees/main?recursive=1"`
3. Identify relevant `target-Combined-Analysis.md` files by package name (e.g., CA pkg, ES pkg, FR pkg)
4. Fetch with: `curl -s -H "Authorization: token <PAT>" -H "Accept: application/vnd.github.v3.raw" "https://github.tools.sap/api/v3/repos/C5403764/abap-documentor/contents/<path>"`

**Jira (direct via MCP)** — see [Authentication → Jira] for rate-limit handling:
- Call `mcp__sap-jira__get_issue` for known ticket keys (extracted from wiki or user input)
- Use `mcp__sap-jira__search_issues` with `projectKey: "ITSCLFL"` and relevant keywords for discovery
- If HTTP 429 on first attempt, wait ~3 seconds and retry once before falling back to wiki-only keys

**Extra sources:**
- Handle each according to its type (Playwright for URLs, MCP tools for structured sources)

### Phase 3 — Synthesis

Produce a structured analysis with these mandatory sections:

1. **Overview** — what the process does, which system variants exist, a comparison table
2. **Key Concepts and Terminology** — glossary table (term → meaning)
3. **Prerequisites** — checklist by category (system config, data state, contract state)
4. **Current State (AS-IS) Process Flow** — numbered steps per variant, in a styled step-lane table
5. **Entitlement / Object Status Lifecycle** — status transition table
6. **Decision Logic** — key branching rules presented as a decision table
7. **Manual / OPS Procedure** — exception handling steps (if found in sources)
8. **ABAP Implementation Map** — ABAP objects table (object, type, package, role)
9. **Planned Changes (TO-BE)** — ticket table with description
10. **Sources** — table of all source URLs/references used

For sections where source data is thin, write "Insufficient data from available sources — manual review recommended" rather than inventing content.

### Phase 4 — Output

**Inline:** Deliver the full markdown analysis in the conversation.

**Word document:** Generate a `.docx` using `docx-js` (node, already installed globally):
- Save to: `C:/Users/C5283308/Documents/CLFL/IAS/docs/<kebab-case-topic>.docx`
- Use SAP colour scheme: SAP Blue `#003B5C`, SAP Mid `#0070F2`, body grey `#3D3D3D`
- Include: cover table, Table of Contents, header/footer with page numbers
- Reuse the generation script pattern from `docs/gen-move-to-urm-doc.js` as a template — adapt sections, do not copy content verbatim

After saving, confirm the file path and size.

## Constraints

- **Always use msedge** for Playwright — Chromium is blocked by SAP Conditional Access
- **Never block on Jira** — if Jira is inaccessible, proceed with wiki + GitHub data and note the gap
- **GitHub PAT** is read from `C:/Users/C5283308/.sap-mcp/auth.json` at runtime — never hardcode it
- **No invented content** — every claim must trace to a source; mark gaps explicitly
- **Parallel fetches** — run independent wiki page fetches and GitHub reads in parallel tool calls
- **Word output is mandatory** — even if some sections are sparse

## Invocation Examples

```
/process-analysis Move to URM: CA calculation and ES update
/process-analysis Tenant Decommissioning + https://wiki.one.int.sap/wiki/spaces/CLFL/pages/1234567
/process-analysis ES Split and Merge + ITSCLFL project Jira filter https://jira.tools.sap/issues/?filter=99999
/process-analysis FR 9500970 Regular Setup + docs/ai-analysis-res/09-06-2026 (PC pkg)/target-Combined-Analysis.md in abap-documentor
```
