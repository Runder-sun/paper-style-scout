---
name: paper-style-scout
description: >
  Analyze top papers in the field, extract writing techniques, and generate a
  project-specific writing style guide with guardrails. Use when user says
  "分析写作风格", "paper style guide", "style scout", "写作指南",
  "generate writing guide", "搜索参考论文", or wants to create a paper-specific
  writing style reference before starting paper writing.
argument-hint: "[research direction | auto]"
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, Agent, WebSearch, mcp__web_reader__webReader
---

# Paper Style Scout

Generate a project-specific writing style guide by analyzing top papers in the field.

Input: $ARGUMENTS

## Constants

- **NUM_LATEST_PAPERS = 3** — Number of latest relevant papers to search via arXiv.
- **NUM_CITED_PAPERS = 2** — Number of highest-cited accepted papers to search via Semantic Scholar.
- **CITED_YEAR_RANGE = 2024-2026** — Year range for high-citation papers.
- **CITED_MIN_CITATIONS = 30** — Minimum citation threshold (adjust per field density).
- **OUTPUT_FILE = WRITING_STYLE_GUIDE.md** — Generated writing guide filename.
- **PAPER_DIR = papers/reference_papers/** — Directory to store downloaded PDFs.

> Overrides (append to arguments):
> - `/paper-style-scout "topic" - latest: 5` — search 5 latest papers instead of 3
> - `/paper-style-scout "topic" - cited: 3` — search 3 high-cited papers instead of 2
> - `/paper-style-scout "topic" - min-citations: 100` — raise citation threshold
> - `/paper-style-scout "topic" - year: 2023-2026` — widen year range

## Inputs

The skill expects one or more of the following (best effort — use what exists):

1. **$ARGUMENTS** — User-provided research direction, idea, and claims.
2. **NARRATIVE_REPORT.md** — Experiment narrative report (if exists).
3. **IDEA_REPORT.md** — Idea discovery report (if exists).
4. **CLAIMS_FROM_RESULTS.md** — Claims extracted from experiment results (if exists).
5. **Repository code** — Python scripts, configs, results directories.

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

Based on REPO_SUMMARY, construct a focused arXiv search query.

```bash
SCRIPT=$(python3 -c "
import pathlib
candidates = [
    pathlib.Path('tools/arxiv_fetch.py'),
    pathlib.Path.home() / '.claude' / 'skills' / 'arxiv' / 'arxiv_fetch.py',
]
for p in candidates:
    if p.exists():
        print(p)
        break
" 2>/dev/null)

if [ -n "$SCRIPT" ]; then
    python3 "$SCRIPT" search "QUERY" --max 10
else
    # Fallback inline search
    python3 - <<'PYEOF'
import json, urllib.parse, urllib.request, xml.etree.ElementTree as ET
NS = "http://www.w3.org/2005/Atom"
query = urllib.parse.quote("QUERY")
url = f"http://export.arxiv.org/api/query?search_query={query}&start=0&max_results=10&sortBy=submittedDate&sortOrder=descending"
with urllib.request.urlopen(url, timeout=30) as r:
    root = ET.fromstring(r.read())
papers = []
for entry in root.findall(f"{{{NS}}}entry"):
    aid = entry.findtext(f"{{{NS}}}id", "").split("/abs/")[-1].split("v")[0]
    title = (entry.findtext(f"{{{NS}}}title", "") or "").strip().replace("\n", " ")
    abstract = (entry.findtext(f"{{{NS}}}summary", "") or "").strip().replace("\n", " ")
    published = entry.findtext(f"{{{NS}}}published", "")[:10]
    papers.append({"id": aid, "title": title, "abstract": abstract, "published": published})
print(json.dumps(papers, ensure_ascii=False, indent=2))
PYEOF
fi
```

Select the top NUM_LATEST_PAPERS most recent and relevant papers. Relevance criteria:

- Topic overlap with REPO_SUMMARY (method, problem, or application domain)
- Recency (prefer papers from the last 12 months)
- Quality signals (detailed abstract, clear contribution statement)

Download PDFs:

```bash
mkdir -p PAPER_DIR
for id in SELECTED_IDS; do
    if [ -n "$SCRIPT" ]; then
        python3 "$SCRIPT" download "$id" --dir PAPER_DIR
    else
        curl -sL "https://arxiv.org/pdf/${id}.pdf" -o "PAPER_DIR/${id}.pdf"
    fi
done
```

### Step 3: Search Highest-Cited Accepted Papers (Semantic Scholar)

Construct a Semantic Scholar query targeting formally published, highly-cited papers.

```bash
S2_SCRIPT=$(find tools/ -name "semantic_scholar_fetch.py" 2>/dev/null | head -1)
[ -z "$S2_SCRIPT" ] && S2_SCRIPT=$(find ~/.claude/skills/semantic-scholar/ -name "semantic_scholar_fetch.py" 2>/dev/null | head -1)

if [ -n "$S2_SCRIPT" ]; then
    python3 "$S2_SCRIPT" search-bulk "QUERY" --max 20 \
        --sort citationCount:desc \
        --year "CITED_YEAR_RANGE" \
        --min-citations CITED_MIN_CITATIONS \
        --publication-types JournalArticle,Conference
else
    # Fallback inline search
    python3 - <<'PYEOF'
import json, urllib.request, urllib.parse
query = urllib.parse.quote("QUERY")
fields = "title,authors,year,citationCount,venue,publicationTypes,abstract,externalIds,openAccessPdf"
url = f"https://api.semanticscholar.org/graph/v1/paper/search?query={query}&limit=20&fields={fields}&year=2024-2026&minCitationCount=30"
req = urllib.request.Request(url, headers={"User-Agent": "paper-style-scout/1.0"})
with urllib.request.urlopen(req, timeout=30) as r:
    data = json.loads(r.read())
papers = sorted(data.get("data", []), key=lambda p: p.get("citationCount", 0), reverse=True)
for p in papers[:5]:
    print(f"{p.get('citationCount',0):>5} cites | {p.get('year','')} | {p.get('title','')[:80]}")
PYEOF
fi
```

Select the top NUM_CITED_PAPERS by citation count that are relevant.

Download open-access PDFs where available. For closed-access papers, use `mcp__web_reader__webReader` to read the abstract page for analysis.

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

The output of this step is NOT a plain markdown document. It is a **project-level SKILL.md** — a proper skill file with YAML frontmatter that both Claude Code and Codex can load and follow as a skill.

Generate the skill file at: `$PROJECT_DIR/skills/paper-writing-guide/SKILL.md`

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

The body of the generated SKILL.md must contain these sections:

#### Section 1: Reference Papers Summary

For each of the 5 papers, a concise block:

```markdown
### [Short Title]
- **Venue**: [venue + year]
- **Citations**: [N]
- **Story type**: [e.g., "problem-driven empirical" / "theory-first with validation"]
- **Abstract pattern**: [e.g., "contribution-first, 5 sentences"]
- **Notable technique**: [one specific thing worth copying]
```

#### Section 2: Recommended Story Arc

Based on the most effective patterns observed, recommend a concrete story structure:

```markdown
## Recommended Narrative Structure

1. **Hook** (1 paragraph): [specific advice from analysis]
2. **Gap** (1-2 paragraphs): [specific advice]
3. **Approach preview** (1 paragraph): [specific advice]
4. **Contributions** (3-4 bullets): [format recommendation]
5. **Evidence preview** (1 paragraph): [specific advice]
```

#### Section 3: Section-by-Section Guide

For each section (Abstract, Introduction, Related Work, Method, Experiments, Conclusion), provide:

- **Recommended structure** (paragraph count, content per paragraph)
- **Techniques observed** in reference papers
- **Common mistakes to avoid**
- **Template** (a skeletal paragraph-by-paragraph outline)

#### Section 4: Writing Techniques

Concrete techniques extracted from the reference papers:

- How to write hooks that work
- How to present contributions without overclaiming
- How to design Figure 1
- How to structure experiment tables
- How to write transitions between sections
- How to handle limitations honestly

#### Section 5: Guardrails (MUST Follow)

This section contains hard rules that MUST be respected during paper writing.

### Step 6: Write Guardrails into the Skill

The guardrails section MUST include the following rules. Write them into the generated SKILL.md under a clear heading.

#### 5.1 Anti-AI-Speak

The following words and phrases are banned. Do NOT use them anywhere in the paper:

**Banned words (single tokens):**
delve, pivotal, landscape, leveraging, comprehensive, facilitate, underscore,
testament, tapestry, myriad, paramount, intricate, nuanced (when used as filler),
multifaceted, noteworthy, quintessential, harnessing, elucidate

**Banned phrases:**
- "it is worth noting that"
- "plays a crucial role"
- "in this paper, we propose" (too generic — be specific about what you propose)
- "has attracted significant attention"
- "in recent years" / "in recent studies"
- "has shown great promise"
- "to the best of our knowledge" (unless you genuinely searched)
- "opens up new avenues"
- "paves the way for"
- "bridges the gap between"

**Banned transition chains:**
Do not string together three or more filler transitions:
- "Furthermore ... Moreover ... Additionally ..." (pick one or restructure)
- "However ... Nevertheless ... Nonetheless ..." (one reversal per paragraph max)

**Detection rule:** After drafting any section, scan for these patterns. Replace with concrete, specific language.

#### 5.2 Anti-Overclaim

- Do NOT use "first", "novel", "first-ever", "pioneering" unless you have done an exhaustive literature search and can cite evidence.
- Do NOT use "state-of-the-art" or "SOTA" unless your method outperforms ALL published results on the SAME benchmark under the SAME setting.
- "Significantly" requires a statistical test (p < 0.05 or similar). Otherwise use "substantially", "noticeably", or just state the raw number.
- Every claim must be scoped: "on benchmark X", "in the Y setting", "for models up to Z parameters".
- Do NOT claim generalization beyond what the experiments test. If you test on English only, do NOT imply multilingual capability.
- Performance improvement must use exact numbers from experiment outputs. Do NOT round up (e.g., 14.7% → "15%" is acceptable, 14.2% → "15%" is not).

#### 5.3 Data Grounding

- Every number in the paper MUST trace to an actual experiment output file in the repository.
- Tables MUST match the raw results files. If a number is not in the results, do NOT put it in the table.
- If data is missing, mark it with `[TODO: need experiment for X]` — never fabricate or estimate.
- Ablation studies MUST correspond to actual runs. Do NOT invent ablation trends.
- Do NOT "beautify" numbers. Report exactly what the experiment produced.
- If a baseline number comes from another paper's reported result, cite the paper and note "as reported in [cite]". Do NOT rerun their method and report your number as theirs unless you explicitly say so.

#### 5.4 Consistency Rules

- Pick ONE term per concept and use it throughout (e.g., don't alternate "model" / "network" / "architecture").
- All mathematical notation must be defined in `math_commands.tex` and used consistently.
- Dataset names, benchmark names, and metric names must be identical every time they appear.
- Figure/table references must use consistent labels (e.g., always "Figure 1", not "Fig. 1" in some places and "Figure 1" in others).

#### 5.5 Honesty Requirements

- Include a Limitations section. Be specific about what the method cannot do.
- If an experiment fails or a hypothesis is not confirmed, report it honestly rather than omitting it.
- Do NOT cherry-pick results. If a baseline outperforms your method on some metric, report it and explain.
- Acknowledge compute resources and data dependencies.
- If using LLM assistance in writing, follow venue disclosure policies.

### Step 7: Write the Skill File

Create directory and write the complete SKILL.md:

```bash
mkdir -p "$PROJECT_DIR/skills/paper-writing-guide"
```

Write the complete skill file to `$PROJECT_DIR/skills/paper-writing-guide/SKILL.md`.

The file must be a valid SKILL.md:
- Start with YAML frontmatter (name, description, argument-hint, allowed-tools)
- Body contains all 5 sections from Steps 5-6
- Self-contained: reading only this skill, an agent can write a well-structured paper in the target field
- Total length: aim for 800-1200 lines (comprehensive but scannable)

### Step 8: Register the Skill in Project Agent Config

The user's project may already have `CLAUDE.md` and/or `AGENTS.md` (these are standard config files for Claude Code and Codex projects). After generating the project-level skill, register it in both files so the agents will load and follow the skill when writing the paper.

**For each file (`CLAUDE.md`, `AGENTS.md`) in the current project root:**

1. Check if the file already exists.
2. If it exists: read it, check if a `## Paper Writing` section is already present. If yes, skip. If no, **append** the section below. Never overwrite existing content.
3. If it does not exist: create the file with the section below.

Content to append to **`CLAUDE.md`**:

```markdown
## Paper Writing

This project has a generated writing guide skill at `skills/paper-writing-guide/SKILL.md`.
Before writing any paper content, load and follow this skill.
All guardrails in its Section 5 are mandatory. Violations must be fixed before proceeding.

When using paper writing skills (/paper-plan, /paper-write, /paper-compile, /auto-paper-improvement-loop),
always check the writing guide skill first and follow its section-by-section recommendations.
```

Content to append to **`AGENTS.md`**:

```markdown
## Paper Writing

This project has a generated writing guide skill at `skills/paper-writing-guide/SKILL.md`.
Before drafting any paper section, load and follow this skill.
All guardrails in its Section 5 are mandatory — do not skip or soften them.

When writing paper content (plan, draft, compile, improve), always follow
the recommendations and constraints in the writing guide skill.
```

## Key Rules

- The output is a **project-level SKILL.md** (not a plain document). It must have YAML frontmatter and be a valid skill that Claude Code and Codex can load.
- The generated skill is project-specific (different papers get different guides). It lives in `$PROJECT_DIR/skills/paper-writing-guide/SKILL.md`, NOT in global skill directories.
- When selecting reference papers, prioritize relevance over recency. A highly relevant 2023 paper is better than a tangentially related 2025 paper.
- All 5 papers must be from the same general field as the project. Do NOT mix unrelated fields.
- The guardrails (Section 5) are non-negotiable. They go into every generated skill without exception.
- If arXiv or Semantic Scholar APIs are unreachable, use WebSearch as fallback. Report what sources were used.
- If fewer than 5 papers are found (rare/niche field), analyze what is available and note the gap.

## Output

1. `skills/paper-writing-guide/SKILL.md` — The complete project-level writing skill (800-1200 lines)
2. Updated `CLAUDE.md` — With paper writing section referencing the skill
3. Updated `AGENTS.md` — With paper writing section referencing the skill
4. Downloaded PDFs in `papers/reference_papers/`

## Composing with Other Skills

```
/paper-style-scout "direction"  →  generates skills/paper-writing-guide/SKILL.md
/paper-plan                     →  reads the generated skill when planning
/paper-write                    →  follows the generated skill during drafting
/paper-compile                  →  compiles to PDF
/auto-paper-improvement-loop    →  review loop respects skill guardrails
```
