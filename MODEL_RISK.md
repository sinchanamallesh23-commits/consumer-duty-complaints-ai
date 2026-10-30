# Model Risk & AI Governance Note

*A one-page note in the style a UK lender's model risk or AI governance team would expect before an AI tool touches customer outcomes. The numbers come from this repo's evaluation.*

## 1. What the system does
| Component | Purpose | Type |
|---|---|---|
| Baseline classifier | Predicts the complaint issue | Predictive ML (TF-IDF + logistic regression) |
| LLM triage (Gemini Flash-Lite) | Issue, Consumer Duty outcome, vulnerability flag, severity, summary | Generative AI, structured JSON output |
| Routing rule | Automate only where the baseline and the LLM agree; otherwise send to human review | Decision rule |
| SQL analyst agent | Answers management questions from complaint data | Agentic AI with one read-only tool |

## 2. Intended use, and what it must NOT be used for
- ✅ **Decision support:** prioritising the complaints queue, surfacing possible vulnerability, drafting summaries, answering factual MI questions.
- ❌ **Not** for deciding complaint outcomes, redress, or anything communicated to a customer without human review.
- ❌ **Not** a replacement for a firm's own vulnerability identification process. A "not vulnerable" flag must never reduce the support a customer receives.

## 3. Performance and known weaknesses
| Area | Evidence | Risk |
|---|---|---|
| Issue labelling | LLM 59.4% vs baseline 70.6% accuracy | The LLM shouldn't be the sole labeller |
| Confidence | Stated 0.92 vs actual 0.59 | **Self-reported confidence is unreliable.** Don't route on it |
| Routing on agreement | 65% automated at 79.1% accuracy | About 1 in 5 automated labels are still wrong. Acceptable for **triage**, not for outcomes |
| Vulnerability | 12/12 recall, 60% precision (gold set of 100) | Small sample: recall's 95% lower bound is about 76%. False alarms add review work |
| Consumer Duty outcome | 74% agreement, κ 0.45 | Moderate. Outcomes genuinely overlap |
| Agent | 10/10 single-answer questions | Not tested on open-ended or ambiguous questions |

## 4. Fairness and consumer protection
- **Under-flagging vulnerability is the main harm.** The system is deliberately tuned towards recall, and any human can override a "not vulnerable" flag.
- Complaints written by people with low literacy or limited English may be harder for the model to read, which is itself a vulnerability driver. This needs a targeted test with real firm data.
- Summaries can omit detail, so reviewers must read the original complaint before acting.

## 5. Data protection
- The data here is public and already scrubbed of personal details by the CFPB.
- **The free API tier may use inputs to improve Google's products. It must never be used with real customer data.** In production: an approved enterprise environment with a data processing agreement, no training on inputs, UK GDPR assessment (DPIA), and data minimisation (send only the complaint text, never account details).
- The API key is stored in a git-ignored `.env` file and is never committed. A key accidentally exposed during the project was rotated.

## 6. Guardrails in place
- **Fixed output schema** with enumerated values: the LLM can't invent categories.
- **Temperature 0**, and the model and prompt version are logged on every output row.
- **Agent:** read-only database connection, SELECT-only (DROP/DELETE blocked and tested), one statement per call, 50-row cap, full query audit log.
- Retries with backoff, and a resumable batch run (no silent partial results).

## 7. Monitoring (if deployed)
| Metric | How often | Action trigger |
|---|---|---|
| LLM–baseline agreement rate | Weekly | A drop of more than 10 percentage points means investigating drift or a model change |
| Accuracy on automated complaints | Monthly human sample of 100 | Below 75% means pausing automation |
| Vulnerability recall | Monthly human-labelled sample | Any sustained fall means retuning the prompt and re-validating |
| Cost per 1,000 and latency | Monthly | Budget review |
| **Model version changes** (`-latest` aliases can change underneath) | Every release | Re-run the full evaluation before accepting |

## 8. Human oversight
A named complaints lead owns the tool. Humans review **all** disagreements, **all** vulnerable or high-severity cases (the priority queue), and a random sample of automated cases each month.
