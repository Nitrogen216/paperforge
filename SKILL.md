---
name: paperforge
description: Analyze any academic paper with ChatGPT, Gemini, and Claude running in parallel, then synthesize into one English summary. Use when asked to read, analyze, or understand a research paper.
argument-hint: "[paper-folder]"
metadata:
  version: 0.1.0
---

# Paperforge — Tri-Model Parallel Paper Reading & Synthesis

Paperforge sends the **same questions** about the **same paper** to three models **simultaneously**, then synthesizes their answers into a single English report covering:

- **Core innovation**: What is the key new idea? What makes it fundamentally different from prior work?
- **How it works**: Method, framework, theory, or system — step by step.
- **Evidence & validation**: Experiments, proofs, case studies, or theoretical analysis — whatever the paper uses.
- **Takeaways**: What should a reader remember? What are the limitations?

Designed for use in a single Claude Code / Codex / Gemini workflow. All three panelists run **in parallel**; synthesis happens only after all three complete.

---

## Usage

Examples:
- "Use paperforge to help me understand this paper."
- "Paperforge this paper."

---

## Inputs

Required:
- `paper.pdf` — preferred; passed to ChatGPT as a browser attachment. Falls back to `paper.txt` if PDF upload times out.
- `paper.txt` — plain-text version extracted from the PDF (also piped to Gemini via stdin).

Optional:
- `questions.md` — custom question list. Default generalized questions are provided below.

---

## Operating Principle: PARALLEL Execution

```
Step 0: Extract paper.txt from PDF + build prompt
Step 1: Launch all three panelists SIMULTANEOUSLY
         - Panelist A: ChatGPT via Oracle browser (paper.pdf preferred, always browser)
         - Panelist B: Gemini (CLI, or in-session if host is Gemini)
         - Panelist C: Claude (CLI, or in-session if host is Claude Code)
Step 2: Wait for all three to complete (A+B+C required)
Step 3: Collect outputs and synthesize into final English report
```

Total wall time ≈ max(ChatGPT, Gemini, Claude) ≈ 5–10 minutes (vs. 15–30 minutes sequential).

---

## Model Choices

- **Panelist A (ChatGPT):** GPT-5.2 Thinking (NOT "Pro") via Oracle browser.
  Confirm the model picker shows **"ChatGPT 5.2 Thinking"** before running.
- **Panelist B (Gemini):** Prefer `gemini -m "gemini-3-pro"`; if model-not-found, retry without `-m` (use account default model). If the host is Gemini, run in-session.
- **Panelist C (Claude):** Claude Sonnet 4.6 via `claude` CLI, or in-session when the host is Claude Code.
- **Synthesizer:** The current host session that launched this skill (Codex / Claude Code / Gemini host).

## Host-Aware Routing (Critical)

Paperforge always requires exactly these three model families for Q1-Q5:
1. ChatGPT via Oracle browser (always browser, no exception)
2. Gemini
3. Claude

Routing rule:
- ChatGPT always runs through Oracle browser, regardless of host.
- If the host is one of the panelists (Gemini/Claude), that panelist runs in-session.
- The other missing panelists must be launched via CLI/browser in parallel.

Examples:
- In Codex: run ChatGPT (Oracle browser) + Gemini (CLI) + Claude (CLI), then synthesize in Codex.
- In Claude Code: run ChatGPT (Oracle browser) + Gemini (CLI) + Claude (in-session), then synthesize in Claude.
- In Gemini host: run ChatGPT (Oracle browser) + Claude (CLI) + Gemini (in-session), then synthesize in Gemini.

Hard requirement:
- Do not skip any of the three panelists.
- If any panelist fails or is unavailable, report blocked status and do not output final synthesis as "complete".

---


## Host Runbooks (Claude Code + Gemini)

These are mandatory run patterns to ensure the skill succeeds in both hosts.

### Claude Code host
1. Start Panelist A (ChatGPT) via Oracle browser in background.
2. Start Panelist B (Gemini CLI) in background.
3. Run Panelist C (Claude) in-session for Q1-Q5.
4. Wait for A+B to finish, then synthesize.

### Gemini host
1. Start Panelist A (ChatGPT) via Oracle browser in background.
2. Start Panelist C (Claude CLI) in background.
3. Run Panelist B (Gemini) in-session for Q1-Q5.
4. Wait for A+C to finish, then synthesize.

Host-specific hard rules:
- ChatGPT must always be Oracle browser (no CLI/API substitution).
- Final synthesis is valid only when outputs from A+B+C all exist.
- If one panelist fails, report blocked and return partial status.

---

## Step 0 — Prepare Paper Files (run once per paper)

### Extract paper.txt from PDF

```python
# run once: python extract_paper.py
import pypdf, sys
sys.stdout.reconfigure(encoding='utf-8')
reader = pypdf.PdfReader('paper/paper.pdf')
with open('paper/paper.txt', 'w', encoding='utf-8') as f:
    for i, page in enumerate(reader.pages):
        f.write(f'--- PAGE {i+1} ---\n')
        f.write(page.extract_text() or '')
        f.write('\n\n')
print(f"Extracted {len(reader.pages)} pages → paper/paper.txt")
```

### CRITICAL on Windows: never use `$(cat file)` for prompts

`$(cat file)` injects CRLF (`\r\n`) line endings that corrupt the `-p` argument.
Oracle then receives only ~23 tokens (the system prompt) and silently ignores the paper.

**Always write the prompt as a single inline string.** See the launch commands below.

**Diagnostic:** check Oracle's launch log token count:
- `~19 k+ tokens` ✓ — file delivered successfully
- `~23 tokens` ✗ — prompt or file failed; use inline prompt

### PDF vs. TXT for ChatGPT upload

- **Preferred:** the actual PDF file — ChatGPT reads PDFs natively; figures and layout are preserved. Pass the real filename (e.g., `--file "paper/My_Paper_Title.pdf"`).
- **CRITICAL:** `paper/paper.pdf` is a **convention shorthand only** — always verify the actual filename before running. If the file does not exist, Oracle exits immediately with code 1 and an empty log.
- **Fallback:** `paper.txt` — use only if Oracle prints `"Attachment upload timed out"` in verbose mode. PDF size is NOT a limiting factor; large PDFs upload successfully.
- Diagnostic: run with `--verbose`; look for `"state":"ready"` (success) vs. `"state":"disabled"` repeating until timeout (rare network issue → switch to `paper.txt`).
- **Token count note:** In browser attachment mode, Oracle's `→ tokens` reflects only the prompt text, NOT the PDF content. A low token count (e.g., `→ 4`) is **normal** when using PDF attachments — it does NOT indicate failure. Confirm success by checking that `"state":"ready"` appeared and ChatGPT's answer references the paper.

---

## Step 1 — Canonical Generalized Prompt

Use this prompt **identically** for all three panelists.
Write it as **one inline string** (no newlines in the `-p` argument on Windows).

The questions are intentionally generic so they apply to any paper type:
empirical ML, theory, systems, survey, human-computer interaction, etc.
Each question instructs the model to **skip sub-parts that do not apply**.

```
You are an expert researcher helping a colleague quickly understand an academic paper.
Read the paper carefully. Answer the following 5 questions with clear section headers Q1 through Q5.
Be specific and cite concrete details from the paper. Do not invent information.
For each question, skip any sub-parts that clearly do not apply to this paper's type.
All answers must be in English.

Q1. PROBLEM AND MOTIVATION:
What problem, challenge, or open question does this paper address?
Why is it important or non-trivial?
What gap, limitation, or assumption in prior work does it identify and target?

Q2. CORE INNOVATION:
What is the single most important new idea, insight, method, or framework?
What makes it fundamentally different from prior work?
Is it a new algorithm, a new theoretical result, a new system design, a new perspective, or a new empirical finding?

Q3. HOW IT WORKS:
Describe the proposed approach step by step.
What are the key components, formulations, or theoretical constructs?
If applicable: what are the inputs and outputs, how is it optimized or implemented, and what are the key equations or algorithms?
If the paper is theoretical, describe the proof strategy and main lemmas instead.

Q4. EVIDENCE AND VALIDATION:
How does the paper support its claims?
This may include: experiments (datasets, metrics, comparisons to baselines, ablations),
theoretical proofs (theorems, bounds), case studies, simulations, or user studies.
Report the key quantitative or qualitative findings.
What does the evidence reveal about which components or assumptions matter most?
Skip sub-questions that do not apply (e.g., if there are no experiments, skip dataset/metric details).

Q5. TAKEAWAYS AND LIMITATIONS:
What is the single most important thing a reader should take away from this paper?
What are the key limitations, assumptions, or failure modes?
What future directions or open problems does the paper suggest?
```

---

## Step 2-4 - Parallel Launch of All Three Panelists

Launch all three simultaneously. In agent hosts, use background tasks for external panelists.

### Panelist A: ChatGPT via Oracle browser (background)

Try PDF first. Fall back to `paper.txt` if Oracle reports upload timeout.

```bash
npx -y @steipete/oracle \
  --engine browser \
  --browser-chrome-profile "Profile 1" \
  --browser-model-strategy current \
  --browser-attachments always \
  --slug "paperforge chatgpt panelist a" \
  --verbose \
  -p "You are an expert researcher helping a colleague quickly understand an academic paper. Read the paper carefully. Answer the following 5 questions with clear section headers Q1 through Q5. Be specific and cite concrete details from the paper. Do not invent information. For each question, skip any sub-parts that clearly do not apply to this paper type. All answers must be in English. Q1. PROBLEM AND MOTIVATION: What problem, challenge, or open question does this paper address? Why is it important or non-trivial? What gap, limitation, or assumption in prior work does it identify and target? Q2. CORE INNOVATION: What is the single most important new idea, insight, method, or framework? What makes it fundamentally different from prior work? Is it a new algorithm, a new theoretical result, a new system design, a new perspective, or a new empirical finding? Q3. HOW IT WORKS: Describe the proposed approach step by step. What are the key components, formulations, or theoretical constructs? If applicable: what are the inputs and outputs, how is it optimized or implemented, and what are the key equations or algorithms? If the paper is theoretical, describe the proof strategy and main lemmas instead. Q4. EVIDENCE AND VALIDATION: How does the paper support its claims? This may include: experiments (datasets, metrics, comparisons to baselines, ablations), theoretical proofs (theorems, bounds), case studies, simulations, or user studies. Report the key quantitative or qualitative findings. What does the evidence reveal about which components or assumptions matter most? Skip sub-questions that do not apply. Q5. TAKEAWAYS AND LIMITATIONS: What is the single most important thing a reader should take away? What are the key limitations, assumptions, or failure modes? What future directions or open problems does the paper suggest?" \
  --file "paper/paper.pdf"
```

Fallback (PDF timeout): replace `--file "paper/paper.pdf"` with `--file "paper/paper.txt"`.

**Success signal:** Oracle log shows `~19 k+ tokens` and prints full Q1–Q5 answer.

---

### Panelist B: Gemini (background unless host is Gemini)

```bash
cat paper/paper.txt | gemini -m "gemini-3-pro" \
  -p "You are an expert researcher helping a colleague quickly understand an academic paper. The paper text is provided via stdin. Read it carefully. Answer the following 5 questions with clear section headers Q1 through Q5. Be specific and cite concrete details from the paper. Do not invent information. For each question, skip any sub-parts that clearly do not apply to this paper type. All answers must be in English. Q1. PROBLEM AND MOTIVATION: What problem, challenge, or open question does this paper address? Why is it important or non-trivial? What gap, limitation, or assumption in prior work does it identify and target? Q2. CORE INNOVATION: What is the single most important new idea, insight, method, or framework? What makes it fundamentally different from prior work? Is it a new algorithm, a new theoretical result, a new system design, a new perspective, or a new empirical finding? Q3. HOW IT WORKS: Describe the proposed approach step by step. What are the key components, formulations, or theoretical constructs? If applicable: what are the inputs and outputs, how is it optimized or implemented, and what are the key equations or algorithms? If the paper is theoretical, describe the proof strategy and main lemmas instead. Q4. EVIDENCE AND VALIDATION: How does the paper support its claims? This may include: experiments (datasets, metrics, comparisons to baselines, ablations), theoretical proofs (theorems, bounds), case studies, simulations, or user studies. Report the key quantitative or qualitative findings. What does the evidence reveal about which components or assumptions matter most? Skip sub-questions that do not apply. Q5. TAKEAWAYS AND LIMITATIONS: What is the single most important thing a reader should take away? What are the key limitations, assumptions, or failure modes? What future directions or open problems does the paper suggest?"
```

If `gemini-3-pro` is unavailable (model not found), retry without `-m` to use the default account model.
PowerShell form (Windows): `Get-Content paper/paper.txt | gemini -p "[same inline prompt]"`.
If the host is Gemini, answer Q1-Q5 in-session instead of launching CLI.

---

### Panelist C: Claude (background unless host is Claude Code)

If the host is Claude Code, Claude reads and analyzes `paper/paper.txt` directly in-session while A and B run in background. Answer Q1-Q5 as a careful paper reader and do not invent information.

If the host is not Claude Code, launch Claude as CLI panelist:
```bash
cat paper/paper.txt | claude -p "[same inline prompt as above]"
```

---

### Bash Parallel Template (terminal / Codex sessions)

```bash
npx -y @steipete/oracle ... --slug "paperforge chatgpt panelist a" \
  --file "paper/paper.pdf" > paper/panelist_a_chatgpt.md 2>&1 &
ORACLE_PID=$!

cat paper/paper.txt | gemini -m "gemini-3-pro" -p "..." \
  > paper/panelist_b_gemini.md 2>&1 &
# If model not found, rerun without -m:
# cat paper/paper.txt | gemini -p "..." > paper/panelist_b_gemini.md 2>&1 &
GEMINI_PID=$!

cat paper/paper.txt | claude -p "..." \
  > paper/panelist_c_claude.md 2>&1 &
CLAUDE_PID=$!

wait $ORACLE_PID && echo "✓ ChatGPT done"
wait $GEMINI_PID && echo "Gemini done"
wait $CLAUDE_PID && echo "Claude done"
echo "All three panelists complete - proceed to synthesis."
```

---

## Step 5 — Synthesis: Merge Three Outputs into Final Report

Once all three panelists have responded, synthesize into a single reader-friendly **English** report.

### Synthesis Prompt

```text
You are the Synthesizer for Paperforge (tri-model parallel paper reading).

You have three independent analyses of the same paper from three AI models:
  Panelist A — ChatGPT (GPT-5.2 Thinking)
  Panelist B — Gemini (preferred: gemini-3-pro; fallback: default model)
  Panelist C — Claude (Sonnet 4.6)

Each panelist answered Q1–Q5 independently.
Your job: produce a single, clear, reader-friendly English report.
The report must work for any paper type (empirical ML, theory, systems, HCI, survey, etc.).
Adapt section headings as needed — omit sections that are not applicable to this paper.

PASS 1 — TRIANGULATE
For each question Q1–Q5:
  A) Find what all three models agree on (consensus = 3/3). Tag with [A][B][C].
  B) Note additions or unique insights from individual models. Tag with [A], [B], or [C].
  C) If models disagree, flag it explicitly and identify which answer is better supported by the paper.
  D) Keep all claims grounded in the paper. Do not invent.

PASS 2 — SYNTHESIZER'S INDEPENDENT JUDGMENT
  E) Form your own understanding of the paper.
     In 2–3 sentences: What is this paper really about?
     What is the single most important thing to understand?

FINAL — PRODUCE THE REPORT

Output in GitHub-compatible Markdown.
For mathematical expressions: use $...$ for inline math and $$...$$ for display math.
Use standard LaTeX notation inside math delimiters (e.g., \frac, \sum, \|, \mathbb).

Required sections (adapt names to fit the paper; omit sections that truly do not apply):

# Paper Summary: [Full Paper Title]

## 0. Paper at a Glance
| Field | Value |
|-------|-------|
| Research area | ... |
| Core problem | ... (one sentence) |
| Proposed approach | ... (one sentence) |
| Key result | ... (one sentence) |

## 1. Problem & Motivation [triangulated]
- The problem being solved: [consensus] [A][B][C]
- Why it matters / why it is non-trivial: ...
- The gap in prior work: ...

## 2. Core Innovation [triangulated]
- The key new idea: [consensus] [A][B][C]
- What makes it fundamentally different from prior work: ...
- Innovation type (algorithm / theory / system / insight / empirical finding): ...
- Synthesizer's judgment on true novelty vs. incremental: ...

## 3. How It Works [triangulated]
- Approach overview (step by step): ...
- Key components or theoretical constructs: ...
- Key equations or algorithms (if applicable): use $$...$$ for display math
- Optimization / implementation (if applicable): ...

## 4. Evidence & Validation [triangulated]
(Adapt to the paper: experiments, proofs, case studies, etc.)
- Validation method(s) used: ...
- Key quantitative or qualitative results: ...
- Comparison to prior work (with numbers, if available): ...
- What the evidence reveals about design choices: ...

## 5. Key Takeaways [triangulated]
- The single most important insight: [consensus] [A][B][C]
- When this approach applies / when it does not: ...
- Key limitations or assumptions to be aware of: ...
- Most promising future direction: ...

## 6. Synthesizer's Assessment
- What this paper really contributes (2–3 sentences):
- Confidence in understanding: High / Medium / Low — reason:
- One thing a reader must not miss:

## 7. Consensus Scorecard
| Claim | A | B | C | Consensus |
|-------|---|---|---|-----------|
| [key claim 1] | ✓/✗ | ✓/✗ | ✓/✗ | 3/3 |
| ... | | | | |

Rules:
- Write entirely in English.
- Write for a researcher who has not read the paper.
- Be concrete: cite numbers, theorem names, dataset names, equation labels where relevant.
- Tag consensus [A][B][C]; unique insights with single tag [A], [B], or [C].
- Use bullet points; keep each section tight.
- Do not invent paper content.
- Omit entire sections (e.g., "Evidence & Validation") only if they genuinely do not apply.
```

---

## Default Question Set (Reference)

```
Q1. PROBLEM AND MOTIVATION: Problem addressed. Why important. Gap in prior work.
Q2. CORE INNOVATION: Key new idea. What makes it different. Innovation type.
Q3. HOW IT WORKS: Step-by-step approach. Key components, formulations, equations.
     (For theory papers: proof strategy and main lemmas instead of pipeline.)
Q4. EVIDENCE AND VALIDATION: How claims are supported. Key results. Design insights.
     (Experiments, proofs, case studies, simulations — whatever applies.)
Q5. TAKEAWAYS AND LIMITATIONS: Most important insight. Limitations. Future directions.
```

---

## Platform Notes (Windows + Mac)

| Issue | Symptom | Fix |
|---|---|---|
| CRLF in `$(cat file)` | Oracle shows `~23 tokens` | Write prompt as inline string (no `$(cat ...)`) |
| PDF file not found (exit 1, empty log) | `--file` path wrong or file doesn't exist | Verify actual filename; pass exact path |
| PDF upload timeout (rare) | Oracle prints `"Attachment upload timed out"` | Switch to `--file "paper/paper.txt"` (PDF size is NOT the cause) |
| Gemini model unavailable | `ERROR: model not found` | Retry without `-m` to use default account model |
| Oracle shows `~23 tokens` | Prompt or file not delivered | Use inline prompt; verify file path |
| Two "remove file" buttons | Stale attachment from prior Oracle run | Normal if `"state":"ready"` appears in log |
| `--slug` invalid (exit 1) | `✓ Custom slug must include between 3 and 5 words` | Use space-separated words: `"paperforge chatgpt panelist a"` (hyphenated = 1 word) |

---

## Expected Deliverable

A single **Paper Summary** (in English) that:
- Works for any paper type — empirical, theoretical, systems, survey
- Clearly explains the core innovation to a reader unfamiliar with the paper
- Describes how the approach works, with key equations rendered in `$$...$$`
- Reports validation results concretely (numbers, theorem names, dataset names)
- Highlights what to remember and what to watch out for
- Is based on triangulated consensus across all three required AI readers (ChatGPT + Gemini + Claude)
