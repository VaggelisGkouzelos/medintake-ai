# MedIntake AI
 
Clinical intake and triage assistant. Natural language drives real application logic through LLM reasoning, tool calling, and human-approved actions.
 
n8n · Claude Sonnet 4.6 · synthetic data only.
 
 
## The problem
 
A SQL query can return a patient row. It cannot read "family history of coronary disease — father, MI at 55" and understand what that implies for triage priority.
 
Staff ask in plain language; the agent decides what data it needs, retrieves it, and cites the protocol behind every answer.
 
> "Assess patient P004 against the clinic protocols. What priority and what actions?"
> → EMERGENCY (Red) per PR001, ECG within 10 minutes, red flags listed, physician review required.
 
## Architecture
 
Two workflows, deliberately split.
 
**Workflow 1 — agentic.** Chat trigger → AI agent with memory and three read-only tools over two data tables. The LLM decides which tools to call and in what order. Non-deterministic by design, because the task requires judgement.
 
**Workflow 2 — deterministic.** Drafts a referral, emails it, pauses for human approval, logs APPROVED or REJECTED. Fixed control flow, because the task requires predictability.
 
 
The LLM sits where judgement is needed, and nowhere else.
 
## Design decisions
 
**Guardrails as policy, then as enforcement.** Five hard rules in the system prompt: no diagnosis, no prescribing, no fabricated data, every referral is a draft, emergencies escalate. Tested adversarially. But a prompt is a policy the model could ignore — so Workflow 2 blocks execution until a human clicks. No prompt injection routes around an architectural gate.
 
**Read-only by default.** Every agent tool uses Get, never Insert. Write access exists in one place — the audit log — reached only after a human decision.
 
**Two retrieval tools, not one with filters.** The model picks tools by reading descriptions; ambiguity produces unreliable selection. Each tool has one job.
 
**Knowledge in data, not in prompts.** Protocols live in a table. Updating one means editing a row.
 
**Table retrieval instead of vector search.** At twelve protocols, embeddings would add a vector store and a threshold to tune for a problem that doesn't exist yet. Breaks past ~50 entries; then it's pgvector, same tool interface.
 
## Measured trade-offs
 
| Configuration | Tokens | Latency |
|---|---|---|
| Bare model | 61 | 2.3s |
| + guardrails | 1,445 | 13.1s |
| + memory + tool schemas | 4,154 | 10.9s |
| Multi-step lookup | 13,435 | 29.1s |
 
Safety is not free. Twenty-nine seconds is fine for a referral assessment and wrong for an autocomplete.
 
## Running it
 
1. Import both `.json` files into n8n (Workflows → Import from File)
2. Create three data tables — `patients`, `protocols`, `referral_log` — from the CSVs
3. Add Anthropic and Gmail credentials
4. Open the agent workflow and use the built-in chat
No API keys are committed.
 
## Limitations
 
- **Synthetic data only.** Real records need GDPR compliance, encryption at rest, access controls. Not implemented.
- **Retrieval doesn't scale.** Fine at 12 protocols, wrong at 500.
- **No evaluation suite.** Guardrails tested manually; production needs a regression suite on every prompt change.
- **In-memory state.** Lost on restart.
- **Not a medical device.** Portfolio project, not clinically validated, not for patient care.
---
 
Built by Evangelos Gkouzelos.
