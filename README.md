# AI Customer Support Copilot

**Recruitbots AI — LLM Application Engineer · Challenge 1 (Beginner)**

One implementation file: [`support_copilot.py`](support_copilot.py).

**Helpline** is a production-style support assistant for Northline. It retrieves answers from a product knowledge base, keeps conversation memory, returns structured JSON with citations, and opens tickets when issues cannot be resolved from the docs.

No fine-tuned models. Customer-facing replies carry no machine-origin labels.

---

## Quick start

```bash
python support_copilot.py --check

python support_copilot.py -m "How do I download my invoice PDF?"

# Continue a conversation
python support_copilot.py -m "What about refunds on monthly plans?" --conversation c1

# Optional HTTP API
pip install fastapi uvicorn
python support_copilot.py --serve
# POST /chat  GET /health  GET /tickets/{id}
```

Optional Llama polish (OpenAI-compatible endpoint such as Ollama):

```bash
python support_copilot.py -m "..." --remote
```

---

## Architecture

```text
Customer message
      │
      ▼
 Conversation memory (per conversation_id)
      │
      ▼
 Hybrid retrieval (BM25 + dense hash + RRF)
      │
      ├─ grounded answer + citations
      ├─ optional Llama polish (grounding guard)
      └─ escalate → support ticket when coverage is low or user asks for a human
      │
      ▼
 Structured JSON response
```

---

## Success criteria

| Criterion | Implementation |
| --- | --- |
| Uses LLM APIs | Optional OpenAI-compatible chat completions (`llama3.1:8b` → `llama3.2:3b`) |
| Conversation context | `ConversationStore` keeps turns; retrieval uses prior user messages |
| Source-backed answers | Citations with `doc_id`, title, source, excerpt |
| Structured JSON | `answer`, `citations`, `structured`, `ticket`, `confidence`, timings |
| Support-ticket escalation | Ticket id, reason, history, cited articles |

---

## Constraints

| Constraint | How it is met |
| --- | --- |
| Must use retrieval | `KnowledgeIndex.search` over 8 product articles |
| Must support chat memory | Per-`conversation_id` history with follow-ups |
| No fine-tuned models | Retrieval + templates; optional base Llama only |

---

## Response shape

```json
{
  "id": "msg_…",
  "conversation_id": "conv_…",
  "answer": "You can download invoices from Settings > Billing > Invoices. [1]",
  "confidence": 0.82,
  "citations": [
    {
      "id": 1,
      "doc_id": "billing_invoice",
      "title": "Download an invoice",
      "source": "kb://billing/invoice",
      "excerpt": "…"
    }
  ],
  "escalated": false,
  "ticket": null,
  "structured": {
    "intent": "billing",
    "category": "billing",
    "action": "resolve",
    "citation_ids": [1]
  }
}
```

When escalated, `ticket` includes history and any articles that were considered.

---

## Escalation rules

- Customer asks for a human / agent / manager  
- Knowledge coverage below threshold  
- Explicit policy paths (fraud, legal, locked account language)  

Tickets are stored in-process and readable via `GET /tickets/{id}`.

---

## Knowledge base topics

Billing (invoice, refunds), account (password reset, 2FA), developers (rate limits, API keys), orders (shipping), and escalation policy.

---

## Unmarked replies

`strip_machine_labels` removes “AI-generated”, “powered by AI”, and similar phrases from answers and ticket reasons. Llama rewrites that invent new numbers are discarded.

---

## Self-check

`python support_copilot.py --check` asserts:

- Invoice question returns citations and resolves  
- Follow-up in the same conversation uses memory  
- “Talk to a human” opens a ticket  
- Out-of-domain questions escalate  
- Structured keys present; no origin labels; under SLA  

---

## File layout

```text
support_copilot.py          # full application
README_support_copilot.md   # this document
```
