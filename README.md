MedIntake AI
Clinical intake and triage assistant — a full-stack GenAI system where a natural-language interface drives real application logic through LLM reasoning, tool calling, and human-approved actions.

Built with n8n and Claude Sonnet 4.6. All patient data is synthetic.

The problem
Clinic reception staff spend a large share of their time on the same handful of tasks: pulling up a patient's history, checking which internal protocol applies to a presenting complaint, judging how urgently a case needs attention, and drafting referral letters. Each task requires reading unstructured notes and applying institutional knowledge that lives in PDFs and in people's heads.

Conventional automation does not help here. A SQL query can return a patient row, but it cannot read "family history of coronary disease — father, MI at 55" and understand what that implies for triage priority.

What the system does
Staff ask questions in plain language. The agent decides which data it needs, retrieves it, and grounds every answer in the clinic's own records and protocols.

"Assess patient P004 against the clinic protocols. What priority and what actions?"

→ retrieves the patient record
→ retrieves the matching triage protocol
→ returns: EMERGENCY (Red) per protocol PR001, ECG within 10 minutes,
   flagged red flags, explicit note that a physician must review
Nothing is invented. Every clinical claim traces back to a row in the database, and the protocol reference is cited in the response.

Architecture
┌─────────────────────────────────────────────────────────────┐
│  WORKFLOW 1 — Conversational Agent (agentic, LLM-driven)    │
│                                                              │
│  Chat Trigger ──→ AI Agent ──→ response                     │
│                       │                                      │
│         ┌─────────────┼─────────────┐                       │
│         ↓             ↓             ↓                        │
│    Claude 4.6     Memory        3 Tools (read-only)          │
│                                      │                       │
│                          ┌───────────┴───────────┐          │
│                     get_patient  list_all   get_protocols    │
│                          │       _patients        │          │
│                          ↓           ↓            ↓          │
│                     [patients table]    [protocols table]    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  WORKFLOW 2 — Referral Approval (deterministic, gated)      │
│                                                              │
│  Trigger → Get Patient → LLM Draft → Email + ⏸ WAIT         │
│                                            │                 │
│                                       Approved?              │
│                                    ┌───────┴───────┐        │
│                                 APPROVED       REJECTED      │
│                                    └───────┬───────┘        │
│                                            ↓                 │
│                                    [referral_log]            │
└─────────────────────────────────────────────────────────────┘
Two workflows, two paradigms
The split is deliberate.

Workflow 1	Workflow 2
Control flow	The LLM decides which tools to call and in what order	Fixed, defined at design time
Write access	None — all tools are read-only	Insert allowed, but only after human approval
Behaviour	Non-deterministic by design	Reproducible
Rationale	The task requires judgement	The task requires predictability
The LLM sits where judgement is needed, and nowhere else. Assessing a patient against protocols benefits from a model that can read free text and weigh context. Routing an approval decision does not — that path is a state machine.

Design decisions
Guardrails as policy, then as enforcement
The system prompt defines five hard rules: no diagnosis, no prescribing, no fabricated data, every referral is a draft requiring physician sign-off, and any emergency indicator triggers immediate escalation.

Tested adversarially. Asked to diagnose a patient presenting with chest pain and dyspnoea, the agent refuses the diagnosis, flags the case as a possible emergency, and offers what it can do instead. Refusing without offering an alternative is a failed guardrail; this one passes.

But a system prompt is a policy — an instruction the model could in principle ignore. Workflow 2 turns the same rule into enforcement: the approval step physically blocks execution until a human clicks. No prompt injection routes around an architectural gate.

Read-only by default
Every tool exposed to the agent uses Get, never Insert or Update. The agent has no reason to modify a medical record, so it cannot. If a prompt injection succeeds, the worst outcome is a wasted lookup.

Write access exists in exactly one place — the audit log in Workflow 2 — and it is reached only after a human decision, through a deterministic path the model does not control.

Two retrieval tools, not one parameterised tool
get_patient takes an identifier and returns one row. list_all_patients takes nothing and returns everything.

They could have been one tool with optional filters. They are not, because the model selects tools by reading their descriptions, and an ambiguous description produces unreliable selection. Each tool has one job, stated plainly, with explicit negative boundaries ("do not use this if the user gave a specific patient ID — use the other tool, it is cheaper").

Verified in practice: single-patient queries route to get_patient, aggregate queries route to list_all_patients, and general questions that need no data at all invoke no tool.

Knowledge in data, not in prompts
Clinical protocols live in a database table, not in the system prompt.

Updating a protocol means editing one row. No workflow change, no prompt rewrite, no redeploy. This is the core reason retrieval exists as a pattern — it separates institutional knowledge from application logic.

Table-based retrieval instead of vector search
With twelve protocols, semantic search over embeddings would add an embeddings provider, a vector store, a chunking strategy, and a similarity threshold to tune — to solve a problem that does not exist at this scale.

The tool returns all twelve protocols and the model matches them against the query using a keywords column that lists multiple phrasings per protocol. This does not scale past roughly fifty entries. At that point the correct move is pgvector with an embeddings model, and the tool interface stays identical — only its implementation changes.

Provider-agnostic by construction
The chat model is a separate, swappable node. Nothing in the agent logic, the tools, or the workflow structure names Anthropic. Switching providers is a node replacement, not a refactor.

Measured trade-offs
Token cost per request, tracked as capability was added:

Configuration	Tokens	Latency
Bare model, no system prompt	61	2.3s
+ system prompt with guardrails	1,445	13.1s
+ memory + 3 tool schemas	4,154	10.9s
Multi-step: patient lookup + protocol lookup	13,435	29.1s
Safety is not free. The system prompt is transmitted on every call, tool schemas are transmitted on every call, and a two-tool query costs three round trips to the model. Twenty-nine seconds is acceptable for a referral assessment and would not be for an autocomplete.

Repository contents
medintake-ai/
├── MedIntake - AI Agent.json          # Workflow 1 — importable into n8n
├── MedIntake - Referral Approval.json # Workflow 2 — importable into n8n
├── patients.csv                       # 10 synthetic patient records
├── protocols.csv                      # 12 clinical triage protocols
└── README.md
Data schema
patients — patient_id, full_name, age, gender, phone, primary_complaint, chronic_conditions, current_medications, allergies, last_visit, department, triage_notes

protocols — protocol_id, title, keywords, triage_level, required_actions, red_flags, escalation, source_reference

referral_log — patient_id, referral_text, status, decided_at

The keywords column carries several phrasings per protocol ("chest pain, θωρακικό άλγος, στηθάγχη") so that retrieval survives variation in how staff phrase a query. The source_reference column exists because a clinical recommendation without a citation is not usable.

Both outcomes are logged. If eight referrals in ten are rejected, the drafting step is failing — and that is only knowable if rejections are recorded. The log is the foundation for an approval-rate metric.

Running it
Import both .json files into n8n (Workflows → Import from File)
Create three data tables — patients, protocols, referral_log — importing the two CSVs and creating referral_log with the schema above
Add an Anthropic API credential
Add a Gmail credential for the approval step
Open the agent workflow and use the built-in chat
No API keys are committed. Credentials are referenced by ID and resolve against your own n8n instance.

Limitations
Synthetic data only. Real patient records would require GDPR compliance, a data processing agreement, encryption at rest, and access controls. None of that is implemented.
Retrieval does not scale. Fine at 12 protocols, wrong at 500.
No evaluation suite. Guardrails were tested manually against adversarial prompts. A production system needs a regression suite over a labelled test set, run on every prompt change.
In-memory conversation state. Session history is lost on restart. Postgres-backed memory is a node swap.
Manual trigger on the approval workflow. In production this would be a webhook invoked by the agent, with the approval running asynchronously.
Not a medical device. This is a portfolio project. It has not been clinically validated and must not be used for patient care.
Stack
n8n · Claude Sonnet 4.6 (Anthropic API) · LangChain agent nodes · n8n Data Tables · Gmail API

Built as a portfolio project by Evangelos Gkouzelos.
