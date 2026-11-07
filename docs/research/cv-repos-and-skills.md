# CV-making repos and skills: research notes

Researched 2026-11-07. These notes cover what exists, what the good projects do, and what this repo should borrow.
I read three Claude CV skills in full (cloned locally): `rendercv/rendercv-skill`, `varunr89/resume-tailoring-skill` and `daniel-spaghetti/cv-builder-skill`.
Everything else comes from READMEs and search summaries. Star counts and "best" rankings come from vendor blogs, so check them before relying on them.

## Contents
1. Landscape: open-source CV tools
2. Claude skills for CVs: what they do and how they're built
3. Domain knowledge: content, ATS, regional rules
4. Skill-authoring rules (Anthropic best practices)
5. Recommendation for this repo

---

## 1. Landscape: open-source CV tools

Most tools fall into five families. The real choice is the **source format** (YAML, JSON, Markdown, LaTeX or Typst). Rendering engines can be swapped later; the source format is harder to change.

| Family | Project | Source → Output | Notes |
|---|---|---|---|
| CV-as-code (YAML) | **RenderCV** (`rendercv/rendercv`) | YAML → Typst → PDF/PNG/HTML/MD | v2.8. Pydantic-validated schema plus JSON Schema for editor autocomplete. 6 themes (classic, harvard, engineeringresumes, engineeringclassic, sb2nov, moderncv) and 22 locales. Has a `--watch` mode and an official Claude skill. **The best fit for an agent.** |
| Data standard (JSON) | **JSON Resume** (`jsonresume/*`) | `resume.json` → npm themes → HTML/PDF | A widely adopted schema. The original `resume-cli` is no longer maintained (`@jsonresume/cli` and `resumed` replace it). Typst bridges exist: `ferrocv` (Rust), `fantastic-cv` and `typst-jsonresume-cv`. |
| LaTeX templates | **Awesome-CV** (posquit0), **Jake's Resume** | `.tex` → PDF | Awesome-CV looks polished and has a large community. Jake's is single-column, ATS-safe and MIT-licensed, and is the de facto standard for tech/student resumes. |
| Typst templates | `modern-cv` (Awesome-CV port, v0.10.0), `basic-resume`, NNJR (Jake-like), `imprecv`, `typst-neat-cv` | `.typ` → PDF | Compiles fast and is easier than LaTeX. `modern-cv` needs the Roboto and Source Sans Pro fonts. |
| GUI builders | **Reactive Resume** (~44k★, v5 2026), **OpenResume** (`xitanggg/open-resume`) | Web app → PDF | Reactive Resume is self-hostable and full-featured. OpenResume also includes an **ATS parser tester**, which is useful for validation. |
| JD-matching / AI | **Resume-Matcher** (`srbhr/Resume-Matcher`) | Resume + JD → match score, keywords, tailored resume | Runs locally on Ollama. Older versions used textacy keyword extraction plus FastEmbed similarity. |
| Markdown-first | LapisCV, plus most Claude skills | MD → PDF (via Typora/Obsidian/VS Code or reportlab) | Simplest to edit, but layout control is weak. |

**Takeaway:** for an agent-driven workflow, use structured YAML with schema validation and Typst rendering (RenderCV). Validation errors give Claude precise feedback, and themes keep content separate from design.

---

## 2. Claude skills for CVs

### 2.1 `rendercv/rendercv-skill` (official, 805-line SKILL.md)
- **Purpose:** gives Claude the full RenderCV Pydantic schema so it can write valid YAML on the first try.
- **Install:** `npx -y skills add rendercv/rendercv-skill --skill rendercv --agent claude-code`, with `uv tool install "rendercv[full]"`.
- **Its pitfalls section is worth copying. These are the errors agents actually make:**
  - **Quote any string that contains `:`.** This is the most common cause of invalid YAML (e.g. `"Relevant coursework: ML"`).
  - `design.highlights.bullet` accepts only `● • ◦ - ◆ ★ ■ — ○`.
  - Phone numbers must be E.164 (`+15551234567`). **Never invent a phone number.** If the country is unknown, ask or leave the field out.
  - `date` and `start_date`/`end_date` are mutually exclusive. Start and end dates must be `YYYY[-MM[-DD]]` or `present`.
  - Each section must use exactly one entry type, which is auto-detected from its fields. There are 9 types (Experience, Education, Normal, Publication, OneLine, Bullet, Numbered, ReversedNumbered, Text).
  - Inline Markdown only (bold, italic, links). Block-level Markdown isn't allowed.
- **Useful trick:** render PNG only (`--dont-generate-pdf --dont-generate-html --dont-generate-markdown`) to check page count, or to let Claude *look* at the layout.

### 2.2 `varunr89/resume-tailoring-skill` (MIT, the most sophisticated)
Its mission statement: *"A person's ability to get a job should be based on their experiences and capabilities, not on their resume writing skills."* Core principle: **truth-preserving optimization.**

The pipeline has checkpoints where the user must confirm before it continues:
0. **Library init.** It indexes the user's existing markdown resumes into a bullet database.
1. **Research.** It parses the JD (requirements, keywords, implicit preferences, red flags, role archetype), then uses WebSearch to research the company (mission, eng blog, news). It benchmarks the role by fetching LinkedIn profiles of people who hold it, then builds a **"success profile"** that includes a *terminology map* from the user's vocabulary to the company's. *Checkpoint.*
2. **Template generation.** It picks section order and slot count per role. *Checkpoint.*
2.5. **Branching experience discovery** (optional). It asks adaptive questions to surface undocumented experience (see 2.4).
3. **Assembly.** Each slot gets candidate bullets with a weighted score: **Direct 40% + Transferable 30% + Adjacent 20% + Impact 10%**, banded as 90+ DIRECT, 75–89 TRANSFERABLE, 60–74 ADJACENT, <60 GAP. It shows the top 3 with ✓/⚠ reasoning and its source resume.
4. **Generation.** Outputs MD, DOCX and PDF, plus a **generation report** (mapping, reframing applied, gaps before and after discovery, interview prep).
5. **Library update.** Newly discovered bullets go back into the library, so it improves over time.
- **Multi-job mode:** shared gaps are ranked by *leverage* (HIGH = appears in 3+ JDs). Discovery runs once and is reused across applications.
- Weakness: the 1,319-line SKILL.md is far over Anthropic's 500-line guidance.

### 2.3 `daniel-spaghetti/cv-builder-skill` (MIT, the most cleanly decomposed)
Three small skills (~100 lines each), each with an explicit "Do NOT use for…" list that routes to its sibling:
- **cv-create:** gathers info, picks a format (chronological, functional or hybrid), writes XYZ/STAR bullets, checks ATS, and reviews CVs in **Must fix / Should fix / Nice to have** tiers. Includes career-stage rules (entry-level: education first, 1 page; senior: 1–2 pages, strategic impact).
- **cv-tailor:** has a **"Critical Rule: No Fabrication"** block (no invented projects, metrics or skills, and no moving accomplishments between roles). Flow: JD parse → map each requirement as Direct, Adjacent or Gap → discovery questions for gaps ("if the user says no after 2–3 probes, move on") → tailored CV saved as `cv-tailored-[company]-[role].md` → gap report with interview talking points. Has a "quick tailor" mode that skips discovery.
- **cv-pdf:** markdown → reportlab PDF with a fixed type scale (Helvetica 16/10/8.5pt) and a **one-page fit ladder**: tighten spacing, then cut 0.5pt from fonts, then trim margins, and only then cut content.
- Its discovery files are adapted from varunr89.

### 2.4 Branching-discovery pattern (from varunr89, reused by daniel-spaghetti)
The core idea is that discovery is not a questionnaire: each answer picks the next branch.
```
Probe: "The job needs {SKILL}. Have you worked with {SKILL} or {RELATED}?"
  YES       → what for, what scale, prod or dev, what challenges, what metrics → build a bullet
  INDIRECT  → your role relative to it → frame as a supporting/enabling contribution
  ADJACENT  → explore related tech → frame as related expertise
  LEARNING  → courses or side projects, how recent → include only if recent and substantive
  NO        → try the broader category once, then move on
```

### 2.5 Other skills seen (not read in full)
- `amanattar/resume-tailoring-skill` (~113★): a single SKILL.md with research, fit scoring and a truthful-generation flow.
- `claude-office-skills/cv-builder`: wraps RenderCV with YAML themes.
- `majiayu000/claude-skill-registry` → `cv-resume-builder`: ATS, STAR/XYZ, and chronological, hybrid, functional and academic templates.
- `erichowens/some_claude_skills` → `cv-creator`: structured data → ATS resume, with a 2-page cap for senior roles.
- `paramchoudhary/resume-bullet-writer` (Smithery): bullets only.

### 2.6 Patterns shared by the good skills
1. A **hard no-fabrication rule**, stated as a rule rather than a suggestion, with gaps reported honestly.
2. **Structured source of truth**, edited by hand or by agent, plus a deterministic renderer.
3. **JD → requirement → evidence mapping** with explicit match strength.
4. **Interactive discovery** to close gaps, with a probe limit.
5. **User checkpoints** before expensive or irreversible steps.
6. **Gap/generation report** as a second deliverable that feeds cover letters and interview prep.
7. **Validation loop**: schema errors, page count and an ATS text-extraction check.

---

## 3. Domain knowledge

### 3.1 Bullet writing
- **XYZ** (commonly attributed to Google recruiting; I found no primary Google source): *"Accomplished [X] as measured by [Y], by doing [Z]."*
- **STAR:** Situation → Task → Action → Result. Use it for richer bullets and interview prep.
- Rules: start with a strong action verb, one accomplishment per bullet, past tense for past roles and present for the current one. Quantify with scale, speed, frequency or money, not just percentages. Ranges or estimates are fine **if the user supplies them**. Don't force a number into every line. **Never** use fake skill meters ("Excel 95%").
- Relevance beats completeness: fewer strong bullets beat many weak ones.

### 3.2 ATS (2026 consensus; mostly from vendor blogs)
- **Safe:** single column, standard headings (Experience, Education, Skills), common fonts, round bullets, contact details in the body.
- **Breaks parsing:** tables, text boxes, multi-column layouts, graphics or icons, info in headers or footers, decorative fonts, unusual bullet glyphs.
- **Files:** a text-based PDF or DOCX both work in modern ATS. Keep both if possible.
- **Test:** extract the text (copy-paste, or `pdftotext`) and check that headings survive and dates stay attached to the right roles. Many recruiters see the ATS's parsed profile, not your PDF.
- Modern ATS use NLP and skills scoring, so **mirror JD terminology honestly; don't keyword-stuff.** Spell out abbreviations once.

### 3.3 Regional conventions
| | US résumé | UK CV | EU (often Europass-style) |
|---|---|---|---|
| Length | 1 page (≤2 for senior) | ≤2 A4 pages | Can be longer |
| Photo | No | No | Often expected in some countries |
| Personal details (age, nationality, marital status) | Omit | Omit (equality law) | Often included |
| Paper / spelling | Letter, US spelling | A4, British spelling | A4 |
| Extras | — | Personal statement at the top; interests are acceptable | Language levels (CEFR) |
"CV" in the US usually means a long **academic** CV.

---

## 4. Skill-authoring rules (from Anthropic's best-practices doc)
- Frontmatter: `name` is at most 64 characters, lowercase, digits and hyphens, and must not contain "claude" or "anthropic". `description` is at most 1,024 characters, **third person**, and says *what* the skill does and *when* to use it. The description is all Claude sees when choosing a skill.
- SKILL.md body **under 500 lines**. Put detail in reference files linked **one level deep**, and give long files a table of contents.
- Assume Claude is smart and only add what it doesn't already know. Pick one default tool and don't list alternatives.
- Prefer bundled **scripts** for deterministic steps (validation, rendering, ATS text check), and make it explicit whether to *run* or *read* each one.
- Use **feedback loops**: validate, fix, repeat. Use copyable progress checklists for multi-step workflows.
- **Evals first:** write about 3 realistic scenarios, measure a baseline without the skill, then write the minimum needed to pass.
- Gerund names are recommended (`tailoring-cvs`), or noun phrases (`cv-tailoring`).

---

## 5. Recommendation for this repo

**Use RenderCV YAML as the single source of truth, and build a small family of skills around it.** This combines daniel-spaghetti's clean split with varunr89's rigour and RenderCV's validation.

```
cv/
  master.yaml              # every fact, every bullet ever written (the "library")
  tailored/<company>-<role>.yaml
  design/default.yaml      # shared theme/design, reused via --design
.claude/skills/
  cv-writing/SKILL.md      # create/review: format choice, XYZ/STAR, ATS, Must/Should/Nice review
  cv-tailoring/SKILL.md    # JD → success profile → evidence map → discovery → tailored YAML + gap report
    discovery-questions.md
    scoring.md
  cv-rendering/SKILL.md    # rendercv render, PNG page-count check, ATS text-extraction check
    scripts/ats_check.sh   # pdftotext + heading/date sanity checks
evals/                     # 3+ scenarios per skill
```

Rules to bake in:
- A no-fabrication rule at the top of every content skill. Gaps are reported, never filled.
- Never invent contact details (phone, email, URLs).
- Ask for target region (US, UK, EU) and seniority early, since they drive length, photo, personal details and spelling.
- Run a validation loop on every render: schema OK, page count within target, ATS text extraction OK.
- Discovered experience gets written back to `master.yaml`, so the library compounds.

**Next step:** build these skills and a starter `master.yaml` from your current CV. That needs your CV (any format) and, optionally, a target job description.

---

## Sources
- RenderCV: https://pypi.org/project/rendercv · skill: https://claudemarketplaces.com/skills/rendercv/rendercv-skill/rendercv · https://github.com/rendercv/rendercv-skill
- https://github.com/varunr89/resume-tailoring-skill
- https://github.com/daniel-spaghetti/cv-builder-skill
- https://github.com/amanattar/resume-tailoring-skill
- https://openskillindex.com/skills/majiayu000-claude-skill-registry-cv-resume-builder · https://openskillindex.com/skills/erichowens-some-claude-skills-cv-creator · https://skills.browseract.com/skills/claude-office-skills_skills_cv-builder · https://smithery.ai/skills/paramchoudhary/resume-bullet-writer
- Anthropic skill best practices: https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices
- Open-source builders: https://dev.to/srbhr/5-open-source-resume-builders-thatll-help-get-you-hired-in-2026-1b92 · https://resufit.com/blog/5-best-open-source-cv-builders-for-professional-resumes-in-2024/ · https://best-of-web.builder.io/library/AmruthPillai/Reactive-Resume · https://github.com/srbhr/Resume-Matcher
- JSON Resume / Typst: https://github.com/jsonresume/resume-cli · https://docs.rs/crate/ferrocv/0.6.0/source/README.md · https://typst.app/universe/package/modern-cv
- LaTeX: https://best-of-web.builder.io/library/posquit0/Awesome-CV · https://best-of-web.builder.io/library/jakegut/resume · https://www.github.com/epidrome/awesome-cv-templates
- ATS: https://atsverification.com/blog/ats-friendly-resume-format-guide-2026/ · https://jobshinobi.com/blog/ats-resume-formatting-rules-2026 · https://www.toptal.com/resume/career-center/ats-friendly-resumes-formatting-examples-and-common-mistakes
- XYZ: https://www.tealhq.com/post/xyz-resume · https://simplify.jobs/blog/how-to-use-the-xyz-resume-format
- Regional: https://standout-cv.com/cv-vs-resume · https://enhancv.com/blog/international-resume · https://blog.openl.io/resume-vs-cv-by-country
