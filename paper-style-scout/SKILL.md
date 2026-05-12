---
name: paper-style-scout
description: >
  Analyze top papers in the field, extract writing techniques, and generate a
  project-specific writing style guide with guardrails. Use when user says
  "分析写作风格", "paper style guide", "style scout", "写作指南",
  "generate writing guide", "搜索参考论文", or wants to create a paper-specific
  writing style reference before starting paper writing.
argument-hint: "[research direction | auto]"
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, WebSearch, mcp__web_reader__webReader
---

# Paper Style Scout

A fully standalone skill. No external scripts, no other skills required.

Given your research direction, idea, and claims, this skill searches reference papers, analyzes their writing, and generates a project-level writing skill with guardrails.

Input: $ARGUMENTS

## Constants

- **NUM_LATEST_PAPERS = 3** — Latest papers to search via arXiv API.
- **NUM_CITED_PAPERS = 2** — Highest-cited papers to search via Semantic Scholar API.
- **CITED_YEAR_RANGE = 2024-2026** — Year range for high-citation papers.
- **CITED_MIN_CITATIONS = 30** — Minimum citation threshold (adjust per field density).
- **PAPER_DIR = papers/reference_papers/** — Directory to store downloaded PDFs.

> Overrides (append to arguments):
> - `paper-style-scout "topic" - latest: 5` — search 5 latest papers instead of 3
> - `paper-style-scout "topic" - cited: 3` — search 3 high-cited papers instead of 2
> - `paper-style-scout "topic" - min-citations: 100` — raise citation threshold
> - `paper-style-scout "topic" - year: 2023-2026` — widen year range

## Workflow

### Step 1: Understand the Project

Parse $ARGUMENTS for research direction, idea, and claims.

Then scan the repository to understand what was actually done:

```bash
# Check for existing reports
for f in NARRATIVE_REPORT.md IDEA_REPORT.md CLAIMS_FROM_RESULTS.md EXPERIMENT_PLAN.md; do
  [ -f "$f" ] && echo "FOUND: $f"
done

# Understand codebase
find . -name "*.py" -not -path "*/venv/*" -not -path "*/__pycache__/*" | head -30
find . -name "*.yaml" -o -name "*.yml" -o -name "*.json" | grep -v node_modules | head -20
find . \( -name "results" -o -name "outputs" -o -name "logs" -o -name "checkpoints" \) -type d | head -10
find . -name "*.md" | head -20
```

From the scan, extract:

- **Research direction**: what field / problem area
- **Core idea**: what method / approach / insight
- **Claims**: what the project claims to achieve
- **Experiment scope**: datasets, metrics, baselines, scale

Output: a concise `REPO_SUMMARY` (keep in context, don't write to file).

### Step 2: Search Latest Relevant Papers (arXiv)

Based on REPO_SUMMARY, construct a focused arXiv search query. Use WebSearch first to find recent papers, then use the arXiv API directly:

```bash
python3 - <<'PYEOF'
import json, urllib.parse, urllib.request, xml.etree.ElementTree as ET

NS = "http://www.w3.org/2005/Atom"
query = urllib.parse.quote("QUERY")
url = (f"http://export.arxiv.org/api/query"
       f"?search_query={query}&start=0&max_results=10"
       f"&sortBy=submittedDate&sortOrder=descending")
with urllib.request.urlopen(url, timeout=30) as r:
    root = ET.fromstring(r.read())
papers = []
for entry in root.findall(f"{{{NS}}}entry"):
    aid = entry.findtext(f"{{{NS}}}id", "").split("/abs/")[-1].split("v")[0]
    title = (entry.findtext(f"{{{NS}}}title", "") or "").strip().replace("\n", " ")
    abstract = (entry.findtext(f"{{{NS}}}summary", "") or "").strip().replace("\n", " ")
    authors = [a.findtext(f"{{{NS}}}name", "") for a in entry.findall(f"{{{NS}}}author")]
    published = entry.findtext(f"{{{NS}}}published", "")[:10]
    papers.append({
        "id": aid, "title": title, "abstract": abstract,
        "authors": authors, "published": published,
        "pdf_url": f"https://arxiv.org/pdf/{aid}.pdf",
        "abs_url": f"https://arxiv.org/abs/{aid}"
    })
print(json.dumps(papers, ensure_ascii=False, indent=2))
PYEOF
```

Select the top NUM_LATEST_PAPERS most recent and relevant papers. Relevance criteria:

- Topic overlap with REPO_SUMMARY (method, problem, or application domain)
- Recency (prefer papers from the last 12 months)
- Quality signals (detailed abstract, clear contribution statement)

Download PDFs:

```bash
mkdir -p PAPER_DIR
for id in SELECTED_IDS; do
    out="PAPER_DIR/${id}.pdf"
    if [ -f "$out" ]; then
        echo "Already exists: $out"
    else
        python3 -c "
import pathlib, urllib.request, sys
out = pathlib.Path('$out')
req = urllib.request.Request('https://arxiv.org/pdf/${id}.pdf', headers={'User-Agent': 'paper-style-scout/1.0'})
with urllib.request.urlopen(req, timeout=60) as r:
    out.write_bytes(r.read())
size = out.stat().st_size
if size < 10240:
    out.unlink()
    print(f'ERROR: file too small ({size} bytes), likely error page')
    sys.exit(1)
print(f'Downloaded: {out} ({size // 1024} KB)')
"
    fi
    sleep 1
done
```

### Step 3: Search Highest-Cited Accepted Papers (Semantic Scholar)

Find the most relevant AND highest-cited papers. Strategy: search by relevance first (Semantic Scholar default ranking), then sort results by citation count. This avoids pulling in loosely-related mega-papers (e.g., "DeepSeek-R1" with 5000+ cites but irrelevant to your specific topic).

```bash
python3 - <<'PYEOF'
import json, urllib.request, urllib.parse

# Step A: Search by relevance (Semantic Scholar default), limit to recent + cited papers
query = urllib.parse.quote("QUERY")
fields = "title,authors,year,citationCount,venue,publicationTypes,abstract,externalIds,openAccessPdf"
url = (f"https://api.semanticscholar.org/graph/v1/paper/search"
       f"?query={query}&limit=50&fields={fields}"
       f"&year=CITED_YEAR_RANGE&minCitationCount=CITED_MIN_CITATIONS")
req = urllib.request.Request(url, headers={"User-Agent": "paper-style-scout/1.0"})
try:
    with urllib.request.urlopen(req, timeout=30) as r:
        data = json.loads(r.read())

    # Step B: Sort the relevance-filtered results by citation count (desc)
    papers = sorted(data.get("data", []), key=lambda p: p.get("citationCount", 0), reverse=True)

    print(f"Found {len(papers)} relevant papers with >={CITED_MIN_CITATIONS} citations")
    print("Top candidates (relevance-filtered, sorted by citations):")
    for p in papers[:5]:
        pdf = p.get("openAccessPdf", {})
        pdf_url = pdf.get("url", "N/A") if pdf else "N/A"
        print(f"  {p.get('citationCount',0):>5} cites | {p.get('year','')} | {p.get('venue','N/A')[:40]} | {p.get('title','')[:60]}")
        print(f"    PDF: {pdf_url}")
except urllib.error.HTTPError as e:
    print(f"Semantic Scholar API error: {e.code} {e.reason}")
    print("Falling back to web search.")
except Exception as e:
    print(f"Error: {e}")
PYEOF
```

```bash
python3 - <<'PYEOF'
import json, urllib.request, urllib.parse

query = urllib.parse.quote("QUERY")
fields = "title,authors,year,citationCount,venue,publicationTypes,abstract,externalIds,openAccessPdf"
url = (f"https://api.semanticscholar.org/graph/v1/paper/search"
       f"?query={query}&limit=20&fields={fields}"
       f"&year=CITED_YEAR_RANGE&minCitationCount=CITED_MIN_CITATIONS")
req = urllib.request.Request(url, headers={"User-Agent": "paper-style-scout/1.0"})
try:
    with urllib.request.urlopen(req, timeout=30) as r:
        data = json.loads(r.read())
    papers = sorted(data.get("data", []), key=lambda p: p.get("citationCount", 0), reverse=True)
    for p in papers[:5]:
        pdf = p.get("openAccessPdf", {})
        pdf_url = pdf.get("url", "N/A") if pdf else "N/A"
        print(f"{p.get('citationCount',0):>5} cites | {p.get('year','')} | {p.get('title','')[:80]}")
        print(f"  PDF: {pdf_url}")
except urllib.error.HTTPError as e:
    print(f"Semantic Scholar API error: {e.code} {e.reason}")
    print("Falling back to web search.")
except Exception as e:
    print(f"Error: {e}")
PYEOF
```

Select the top NUM_CITED_PAPERS by citation count that are relevant.

Download open-access PDFs where available. For closed-access papers, use `mcp__web_reader__webReader` to read the abstract/DOI page for analysis.

If Semantic Scholar API is unreachable, use WebSearch to find highly-cited published papers instead.

### Step 4: Read and Analyze Each Paper

For each of the 5 selected papers, read the content:

- If PDF is downloaded: use `Read` tool on the PDF file (reads first pages)
- If only abstract available: use `mcp__web_reader__webReader` on the arXiv/DOI page

For each paper, extract analysis across these dimensions:

**A. Story Arc**

```
Hook (opening statement)
  → Gap / Problem (what's missing)
    → Insight / Approach (the key idea)
      → Evidence (experimental validation)
        → Takeaway (what to remember)
```

Document: How many paragraphs for each phase? Where does the method first appear?

**B. Abstract Pattern**

- Sentence count and function of each sentence
- Does it follow the 5-sentence formula? Variations?
- Where does the first number/quantitative result appear?
- Opening strategy (contribution-first vs background-first)

**C. Introduction Structure**

- Total paragraph count
- Where is Figure 1 placed and what does it show?
- How many contribution bullets? How specific are they?
- How does the motivation build (data → problem → insight)?

**D. Method Presentation**

- Notation density (light / moderate / heavy)
- Does it use pseudocode, algorithm blocks, or inline equations?
- How does it connect back to the introduction's claims?
- Is there a running example?

**E. Experiment Design**

- How are baselines selected and grouped?
- Ablation strategy (one-at-a-time vs additive)
- Table vs figure ratio
- Statistical reporting (error bars, significance tests)
- Main result presentation order

**F. Language and Style**

- Active vs passive voice ratio
- Sentence length (short and punchy vs flowing academic)
- Paragraph length
- Terminology consistency
- Transition style (implicit topic flow vs explicit connectors)

### Step 5: Generate a Project-Level Writing Skill

The output is a **project-level SKILL.md** — a proper skill file with YAML frontmatter that both Claude Code and Codex can load and follow as a skill.

Generate the skill file in the correct platform-specific directory:

- **Claude Code**: `.claude/skills/paper-writing-guide/SKILL.md`
- **Codex**: `.agents/skills/paper-writing-guide/SKILL.md` (preferred cross-platform path) or `.codex/skills/paper-writing-guide/SKILL.md`

Create the directory and write the skill file to the path that matches the current agent. If unsure, create in both locations:

```bash
mkdir -p .claude/skills/paper-writing-guide
mkdir -p .agents/skills/paper-writing-guide
```

The generated SKILL.md must have this structure:

```yaml
---
name: paper-writing-guide
description: >
  Project-specific paper writing guide generated by paper-style-scout.
  Contains field-specific writing techniques, section-by-section templates,
  and mandatory guardrails. Load this skill before writing any paper content.
argument-hint: "[section name or 'full']"
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob
---
```

The body must contain these sections:

**Section 1: Reference Papers Summary** — For each of the 5 papers: venue, citations, story type, abstract pattern, notable technique.

**Section 2: Recommended Story Arc** — Concrete narrative structure based on the most effective patterns observed (hook, gap, approach, contributions, evidence).

**Section 3: Section-by-Section Guide** — For each section (Abstract, Introduction, Related Work, Method, Experiments, Conclusion): recommended structure, techniques observed, common mistakes, paragraph-by-paragraph template.

**Section 4: Writing Techniques** — Concrete techniques: hooks, contributions, Figure 1, experiment tables, transitions, limitations.

**Section 5: Guardrails (MUST Follow)** — Hard rules, see Step 6.

### Step 6: Write Guardrails into the Skill

The guardrails MUST be written into the generated SKILL.md under a clear heading. These rules are non-negotiable and go into every generated skill.

#### 6.1 Anti-AI-Speak

The following words and phrases are banned. Do NOT use them anywhere in the paper:

**Banned words:** delve, pivotal, landscape, leveraging, comprehensive, facilitate, underscore, testament, tapestry, myriad, paramount, intricate, nuanced (as filler), multifaceted, noteworthy, quintessential, harnessing, elucidate

**Banned phrases:**
- "it is worth noting that"
- "plays a crucial role"
- "in this paper, we propose" (too generic — be specific)
- "has attracted significant attention"
- "in recent years" / "in recent studies"
- "has shown great promise"
- "to the best of our knowledge" (unless genuinely searched)
- "opens up new avenues"
- "paves the way for"
- "bridges the gap between"

**Banned transition chains:**
- "Furthermore ... Moreover ... Additionally ..." (pick one or restructure)
- "However ... Nevertheless ... Nonetheless ..." (one reversal per paragraph max)

**Detection rule:** After drafting any section, scan for these patterns. Replace with concrete, specific language.

#### 6.2 Anti-Overclaim

- Do NOT use "first", "novel", "first-ever", "pioneering" without exhaustive evidence.
- Do NOT use "state-of-the-art" or "SOTA" unless outperforming ALL published results on the SAME benchmark under the SAME setting.
- "Significantly" requires a statistical test. Otherwise use "substantially" or just state the raw number.
- Every claim must be scoped: "on benchmark X", "in the Y setting".
- Do NOT claim generalization beyond what experiments test.
- Do NOT round up improvements (14.2% → "15%" is not acceptable; 14.7% → "15%" is).

#### 6.3 Data Grounding

- Every number MUST trace to an actual experiment output file.
- Tables MUST match raw results files.
- Missing data → `[TODO: need experiment for X]`, never fabricate.
- Ablations MUST correspond to actual runs. Do NOT invent trends.
- Do NOT "beautify" numbers.
- Baseline numbers from other papers → cite as "as reported in [cite]".

#### 6.4 Consistency Rules

- ONE term per concept throughout (no alternating "model"/"network"/"architecture").
- Math notation defined in `math_commands.tex`, used consistently.
- Dataset/benchmark/metric names identical every time.
- Figure/table references use consistent labels.

#### 6.5 Honesty Requirements

- Include a Limitations section. Be specific.
- Report failed experiments honestly, don't omit.
- Do NOT cherry-pick results.
- Acknowledge compute and data dependencies.
- Follow venue LLM disclosure policies.

### Step 7: Write the Skill File

```bash
mkdir -p .claude/skills/paper-writing-guide
mkdir -p .agents/skills/paper-writing-guide
```

Write the same skill file to both paths (so both Claude Code and Codex can discover it).

Requirements:
- Valid YAML frontmatter (name, description, argument-hint, allowed-tools)
- All 5 sections from Steps 5-6
- Self-contained: reading only this skill, an agent can write a well-structured paper
- Include concrete examples (good vs bad) from the reference papers
- Total length: 800-1200 lines

### Step 8: Register the Skill in Project Config

The user's project may already have `CLAUDE.md` and/or `AGENTS.md`. After generating the skill, register it so both agents load and follow it.

**For each file (`CLAUDE.md`, `AGENTS.md`) in the current project root:**

1. Check if the file already exists.
2. If it exists: read it, check if a `## Paper Writing` section is already present. If yes, skip. If no, **append** the section below. Never overwrite existing content.
3. If it does not exist: create the file with the section below.

Content to append to **`CLAUDE.md`**:

```markdown
## Paper Writing

This project has a generated writing guide skill at `.claude/skills/paper-writing-guide/SKILL.md`.
Before writing any paper content, load and follow this skill.
All guardrails in its Section 5 are mandatory. Violations must be fixed before proceeding.
```

Content to append to **`AGENTS.md`**:

```markdown
## Paper Writing

This project has a generated writing guide skill at `.agents/skills/paper-writing-guide/SKILL.md`.
Before drafting any paper section, load and follow this skill.
All guardrails in its Section 5 are mandatory — do not skip or soften them.
```

## Key Rules

- This skill is **fully standalone**. All API calls are inline Python using only the standard library. No external scripts or other skills are required.
- The output is a **project-level SKILL.md** (not a plain document). It must have YAML frontmatter and be loadable by Claude Code and Codex.
- Claude Code discovers project skills in `.claude/skills/<name>/SKILL.md`.
- Codex discovers project skills in `.agents/skills/<name>/SKILL.md` or `.codex/skills/<name>/SKILL.md`.
- The generated skill should be written to both `.claude/skills/paper-writing-guide/` and `.agents/skills/paper-writing-guide/` so both agents can discover it.
- When selecting reference papers, prioritize relevance over recency.
- All 5 papers must be from the same general field as the project.
- The guardrails are non-negotiable. They go into every generated skill.
- If arXiv or Semantic Scholar APIs are unreachable, use WebSearch as fallback.
- If fewer than 5 papers are found, analyze what is available and note the gap.

## Output

1. `.claude/skills/paper-writing-guide/SKILL.md` — Writing skill for Claude Code (800-1200 lines)
2. `.agents/skills/paper-writing-guide/SKILL.md` — Same writing skill for Codex
3. Updated `CLAUDE.md` — With paper writing section referencing the skill
4. Updated `AGENTS.md` — With paper writing section referencing the skill
5. Downloaded PDFs in `papers/reference_papers/`
