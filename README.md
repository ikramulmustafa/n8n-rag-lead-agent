# IkramBot — RAG Agent with Lead Capture

A conversational agent built in n8n that answers questions about my professional background from a private vector store, and captures project inquiries as structured leads delivered by email.

Built as a working system rather than a demo: it runs in production on my portfolio site, handles real visitor traffic, and has a defined behaviour for every failure path.

---

## What it does

Two distinct jobs in one agent:

1. **Retrieval** — answers factual questions about my experience, projects and skills, grounded in a PGVector store rather than model knowledge.
2. **Lead capture** — recognises hiring or project intent, collects the fields needed to act on it, confirms with the user, and emails the structured lead.

The interesting engineering is in keeping those two behaviours separate without them interfering with each other.

---

## Architecture

```
Chat Trigger (webhook)
        │
        ▼
    AI Agent ──── Chat Model (OpenAI)
        │     └── Simple Memory (20-turn window)
        │
        ├── your_information_retriever  →  PGVector store
        │                                    ↑
        │                              OpenAI Embeddings
        │                              (text-embedding-3-small, 1536d)
        │
        └── send_email  →  Gmail (OAuth2)
```

**Stack:** n8n, LangChain nodes, OpenAI (chat + embeddings), PostgreSQL with pgvector, Gmail API.

---

## Design decisions

### Mode priority, not intent classification

The agent resolves to one of four modes in a fixed priority order: lead capture, greeting, retrieval, out of scope.

Lead capture sits at the top deliberately. A message like *"can you build me a CRM?"* is both a question about capability and a buying signal. Classifying it as retrieval loses the lead. Ordering the modes rather than classifying the message means the higher-value interpretation always wins, and ambiguity resolves in the direction that matters commercially.

Once lead capture is active it stays active until the lead is sent or declined, so a multi-turn conversation cannot drift back into Q&A halfway through collecting a name and email.

### Retrieval guardrail inside the lead flow

The retriever is called **once** at the first lead turn, to produce a single capability summary. It is then locked out for the remainder of that lead session.

Without this, the agent re-runs retrieval on every turn while collecting contact details — adding latency and token cost to answer a question nobody asked, and often repeating the same capability blurb three times in one conversation. The only escape hatch is an explicit user request to see the profile again.

This is the kind of thing that only shows up once real users are in a flow. It is invisible in testing.

### Scoped side effects

The agent has exactly one tool that changes the outside world, and its blast radius is fixed at design time:

- `sendTo` is **hardcoded**, not model-supplied. The agent decides *whether* to send, never *where*.
- The send is gated behind explicit user confirmation ("Shall I email this to Ikram now?").
- Required fields are validated before the confirmation step is reached.

With an agent that can take actions, a wrong answer is not a bad response — it is a wrong action someone has to clean up. Authorisation belongs before the tool call, not as a check on the output afterwards.

### Grounding and refusal

Retrieval answers are constrained to the retrieved passages, capped at 180 words, with no citations surfaced to the user. When nothing relevant is retrieved, the agent asks for clarification rather than filling the gap from model knowledge.

Out-of-scope questions are refused explicitly. The bot has one job and says so.

### Defined failure states

Every external dependency has a user-facing behaviour when it fails:

| Failure | Behaviour |
|---|---|
| Retriever slow | "Searching Ikram's knowledge base… please wait." |
| Retriever unavailable | Explicit apology, invitation to retry |
| Email send fails | Apologise, request an alternative contact method |
| No relevant passages | Ask for clarification rather than guess |

Silence is the worst error state for a conversational interface, because the user cannot distinguish a broken system from being ignored. Every path here produces a response.

---

## Data flow and privacy

- Conversation memory is a 20-turn rolling window, scoped per session.
- PII is retained only long enough to compose a confirmed lead email, and is not persisted beyond that.
- System instructions and credentials are never exposed in output.
- The vector store contains only my own professional information.

---

## What I would change at higher scale

Being honest about the current limits:

- **Public webhook + side-effecting tool.** The chat trigger is public and the agent can send email. The hardcoded recipient caps the damage at inbox spam rather than exfiltration, but rate limiting and abuse protection belong in front of it before traffic grows.
- **Model choice under a long instruction set.** The mode-priority rules and the retrieval lockout are exactly the sort of instruction a smaller model drops under context pressure. A larger model on the reasoning path, or splitting mode selection into its own deterministic step, is the fix.
- **Corpus size.** The store is intentionally small. Broader coverage would need chunking strategy and re-ranking work rather than simply more rows.
- **Observability.** Tool calls and mode decisions should be logged, not just final output. When an agent takes a wrong path, the answer alone does not tell you why.

---

## Files

- `workflow.json` — importable n8n workflow
- `system-prompt.md` — full agent instruction set

Credentials and instance-specific identifiers have been removed.
