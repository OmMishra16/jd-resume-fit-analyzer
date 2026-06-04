# JD-Resume Fit Analyzer (Agentic n8n Workflow)

> Course assignment: Agentic Workflow Design and n8n Demo.
> A multi-step n8n workflow that scores how well a resume fits a job description, with clearly
> separated **AI reasoning** and **deterministic control**, plus **routing**, **human-in-the-loop**
> review, **structured JSON outputs**, and **fallback handling**.

Loom : https://www.loom.com/share/d4b6521548a44f4787bf004466cbf158

## 1. Problem statement

- **User:** a job-seeking student (and, symmetrically, a recruiter screening many resumes).
- **Pain point:** judging "is this resume a good fit for this job?" by hand is slow, subjective,
  and inconsistent. Candidates rarely learn why they are a weak fit or what to fix.
- **Goal:** given a Job Description + a Resume (PDF), automatically (a) extract structured
  requirements and skills, (b) score the fit objectively, (c) bucket the candidate, (d) route to
  the right action, and (e) return a usable report.
- **Output:** a structured fit report (score, matched/missing skills, strengths, gaps, verdict,
  and for weak fits coaching tips), shown on screen and appended to a local run log.

This is a pipeline of specialized steps where AI is used only where reasoning is needed and
deterministic logic owns all control decisions. It is deliberately not a single chatbot prompt.

## 2. Workflow at a glance

```
[Form] Submit JD + Resume (PDF)
   |
[Extract from File]  PDF -> text                          (tool use, no AI)
   |
[Code] Prepare & Validate Input                           (DETERMINISTIC: presence + min length)
   |
[IF] Valid Input? --false--> [Form] Invalid Input         (FALLBACK / error handling)
   | true
[AI] JD Requirement Extractor   -> JSON                    (AI role 1: extractor)
   |
[AI] Resume Profiler            -> JSON                    (AI role 2: extractor)
   |
[AI] Fit Analyst                -> JSON sub-scores+evidence(AI role 3: reasoner)
   |
[Code] Weight & Bucket          -> final_score + category (DETERMINISTIC: weights + thresholds)
   |
[Switch] Route by Category
   |- STRONG   -> [Set] decision = Shortlisted (auto) ------------+
   |- MODERATE -> [Wait] Human Review (Approve/Reject) -> [Set] --+   (HUMAN-IN-THE-LOOP)
   |- WEAK     -> [AI] Resume Coach -> JSON -> [Set] -------------+   (AI role 4: recommender)
                                                                 |
                                            [Code] Build Report  (converge all branches)
                                                                 |
                                            [Code] Append to Log File  (tool use + fallback)
                                                                 |
                                            [Form] Show Result   (usable output)
```

A single shared Google Gemini Chat Model feeds all four AI nodes; each AI node has its own
Structured Output Parser that enforces a JSON schema.

## 3. AI vs deterministic (the key design decision)

| Step | Type | Why |
|------|------|-----|
| Extract from File | Tool (no AI) | Built-in PDF to text. |
| Prepare & Validate Input | Deterministic | Presence + min-length rules. No reasoning needed. |
| JD Requirement Extractor | AI | Unstructured JD to structured fields. |
| Resume Profiler | AI | Free-form resume to normalized profile. |
| Fit Analyst | AI | Judgement: compare, score each dimension, justify with evidence. |
| Weight & Bucket | Deterministic | Fixed weights + thresholds. Stable and auditable. |
| Route by Category | Deterministic | Control flow on a known field. |
| Human Review | Human-in-the-loop | High-impact shortlist decision gets a person. |
| Resume Coach | AI | Generative, tailored feedback. |
| Build Report / Append Log / Show Result | Deterministic | Formatting, persistence, output. |

**The crux:** the AI proposes per-dimension sub-scores (skills_match, experience,
projects_relevance) with evidence; the deterministic code decides the final weighted score, the
category, and the route. AI judges, math decides, so scoring is consistent and explainable.

Declarative rubric (in the Weight & Bucket node):

```
final_score = 0.5*skills_match + 0.3*experience + 0.2*projects_relevance
STRONG >= 75   |   MODERATE 50-74   |   WEAK < 50
```

It also computes an independent coverage cross-check (matched_required / required) so the final
score is not blindly trusting the model.

## 4. Agentic practices demonstrated

| Practice | Where |
|----------|-------|
| Role definition | 4 distinctly-prompted AI roles: Extractor, Profiler, Analyst, Coach. |
| Structured outputs | Every AI node uses a Structured Output Parser (JSON schema). |
| Tool / integration use | Extract-from-File (PDF), local file logging. |
| Routing / branching | Switch on category; Approve/Reject sub-branch. |
| Deterministic checks | Validation, declarative weights, thresholds, coverage cross-check. |
| Human-in-the-loop | Wait "On Form Submitted" Approve/Reject page on MODERATE fits. |
| Fallback / error handling | Invalid-input branch; Continue-On-Fail on the log node. |
| Fairness | Every prompt scores only skills/experience/projects; ignores name/gender/age/college. |

Note: the AI nodes use the Basic LLM Chain + Structured Output Parser rather than the tool-calling
AI Agent node. For pure extraction/classification this is more reliable and predictable; each
chain is given an explicit role via its prompt, which is the agentic decomposition here.

## 5. How to run it

1. Start n8n locally (free). The env var lets the log node write a file:
   ```bash
   NODE_FUNCTION_ALLOW_BUILTIN=fs,os,path npx n8n
   ```
   Open http://localhost:5678
2. Get a free Gemini API key (https://aistudio.google.com/apikey) and add a
   **Google Gemini(PaLM) Api** credential in n8n.
3. Import `workflow/jd-resume-fit-analyzer.json` (Workflows -> Import from File).
4. Open the **Google Gemini Chat Model** node and select your Gemini credential
   (model: `models/gemini-2.5-flash`).
5. Click **Execute workflow**, open the form, paste `samples/sample_jd.txt`, upload
   `samples/sample_resume.pdf`, and submit.
6. For a MODERATE result, open the Human Review form link, choose Approve or Reject, and the final
   report renders.

Try the three resume variants to hit every branch:
`sample_resume.txt` (moderate), `sample_resume_strong.txt` (strong), `sample_resume_weak.txt` (weak).
Submit an empty Job Description to see the fallback (Invalid Input) branch.

## 6. Repository contents

```
workflow/jd-resume-fit-analyzer.json   exported n8n workflow (prompts + code embedded)
samples/                               sample JD, 3 resume variants (+ PDF), sample output
README.md                              this file
```

## 7. Sample output

See `samples/sample_output.json` for a full captured run. Summary for the moderate-fit sample
(`sample_resume.txt` vs the backend JD):

```
Category: MODERATE | Fit Score: ~68/100 | Coverage: ~80%
Matched: Python, FastAPI, REST API, PostgreSQL, Git, pytest
Missing: Docker, AWS, Kafka
-> routed to Human Review -> (Approve) -> Shortlisted (after human review)
```

## 8. Limitations & future work

- Input is text-based PDF / pasted JD; scanned image-only resumes would need an OCR step.
- Logging is a local JSONL file; a real deployment would use Google Sheets / a database and email
  the report (easy node swaps, avoided here to keep it free and credential-free).
- Single resume at a time; batch screening would add a loop over multiple uploads.
