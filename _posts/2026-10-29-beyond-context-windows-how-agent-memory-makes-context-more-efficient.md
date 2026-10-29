---
layout: post
title: "Beyond Context Windows: How Agent Memory Makes Context More Efficient"
date: 2026-10-29
categories: [agentic-ai, llm-memory, rag, atlas-agent-engine]
---

# Beyond Context Windows: How Agent Memory Makes Context More Efficient

*Understanding four types of long-term memory, Pete Johnson's Recap Loop, and MongoDB Atlas Agent Engine.*

---

Imagine an AI agent designed to help employees resolve internal IT issues. When an employee requests access to a database, the agent retrieves the relevant documentation, identifies the necessary approval process, and helps resolve the ticket. A week later, the same employee encounters a similar problem. Should the agent retrieve all the documentation and reconstruct the exact same solution from scratch? What if it could first check whether it had already encountered and answered a similar question before?

This is where agent memory becomes interesting. Although modern LLMs support increasingly large context windows, context remains a finite resource. Retrieving every past conversation, previous tool execution result, external document, procedure, and background fact into a prompt is neither computationally efficient nor necessarily helpful for model reasoning. The core engineering challenge is not simply to remember more information, but to retrieve the right information at the right time while using as little context as necessary.

---

## 1. Short-Term vs. Long-Term Memory

Agent memory can be divided into two broad categories: **short-term memory** and **long-term memory**. Short-term memory maintains the recent conversation history, including user messages, agent responses, and intermediate tool calls within an active session. Long-term memory retains durable knowledge across sessions, such as previous experiences, established procedures, relevant facts, and domain-specific terminology.

Rather than repeatedly placing an entire conversation history, previous tool execution outputs, and bulky RAG chunks into the LLM's context window on every single turn, an agent equipped with long-term memory can selectively retrieve concise, focused memories. MongoDB Atlas Agent Engine supports this distinction through a dedicated memory service that records conversation turns and extracts durable knowledge asynchronously in the background.

In this article, we'll focus on long-term memory, how its four categories are organized, how knowledge is extracted and reconciled, and how those memories can improve context utilization.

---

## 2. Four Types of Long-Term Memory

In [O'Reilly's Expert Showcase: Agent Memory](https://www.oreilly.com/live-events/expert-showcase-agent-memory/0642572440619/), MongoDB's Pete Johnson introduces four memory types: **episodic**, **procedural**, **semantic**, and **taxonomic**. Each represents a distinct kind of knowledge an agent needs to operate effectively.

### Episodic Memory — What Happened Before?

Episodic memory stores previous interactions and experiences as summarized episodes rather than raw, verbatim transcripts. Consider an employee who previously encountered a database access problem. Instead of preserving the entire 20-turn conversation in the active context window, the agent can retain a concise summary:

> *An employee couldn't access the analytics database because their group membership had expired. Following approval, the membership was renewed and access was restored.*

If the employee later encounters a similar issue, this past experience provides immediate, relevant context without reintroducing entire conversation histories. Episodic memories are private by default, associated with the user identity within a project, and serve to retrieve prior experiences cleanly and efficiently.

### Procedural Memory — How Should a Task Be Performed?

Procedural memory stores reusable workflows, runbooks, and established execution plans. For example, consider a standard database access procedure:

1. Verify employee identity.
2. Determine the requested permission level.
3. Check existing group memberships.
4. Obtain the necessary approval.
5. Provision access.
6. Verify completion.

Without procedural memory, the LLM may need to reconstruct a plan each time it encounters the same type of request, which risks hallucinating missing steps or omitting verification checks. Instead, the agent retrieves an existing, proven procedure and supplies it directly into the task context.

Procedural memory is private by default within the associated user's project, supplying established plans when a task requires them and reducing repeated planning overhead. It is also important to note that procedures extracted dynamically from user conversations should be distinguished from official standard operating procedures (SOPs); a workflow observed in an ad-hoc debugging session is not necessarily an approved policy for all users.

### Semantic Memory — What Does the Agent Know?

Semantic memory stores relevant, durable facts that remain valid long after the interaction where they were first mentioned. In a software engineering or developer assistant, these facts capture environment configuration, architecture stacks, and team preferences:

- The developer's primary service is built with Python 3.12 and FastAPI, using strict type annotations and async handlers.
- The service connects to a PostgreSQL 16 cluster with `pgvector` enabled for semantic embeddings.
- All background task queues run via Celery on Redis 7.2 in an AWS ECS `us-east-1` deployment.

Unlike episodic memory, which recalls the specific session where an issue was diagnosed, semantic memory strips away the conversational narrative and focuses strictly on the factual technical reality. These facts are private by default, allowing the agent to tailor its code generation and architectural answers to the user's specific tech stack without needing to re-parse older conversation logs.

### Taxonomic Memory — What Does a Term Mean?

Taxonomic memory stores domain-specific terminology, internal project code names, abbreviations, and definitions. Consider an organization where an internal infrastructure service is officially indexed as `F-12` in ticketing databases but referred to colloquially as `Falcon` across engineering channels. A user might ask:

> *\"What is the current deployment status of Falcon?\"*

Elsewhere, project documentation and monitoring metrics may refer exclusively to `F-12`. Without knowledge of this naming relationship, an agent might fail to recognize that both identifiers denote the exact same service.

Taxonomic memory bridges this gap by maintaining alias graphs and domain definitions, allowing the agent to interpret queries within the correct organizational context. Unlike personal episodes or user preferences, taxonomic memory is organization-wide within the project by default, enabling all team members in the project to benefit from shared vocabulary.

---

### How Agent Memories are Extracted and Reconciled

Neither episodic summaries, procedural workflows, semantic facts, nor taxonomic terms need to be manually curated by developers. MongoDB Atlas Agent Engine implements an asynchronous background pipeline that extracts and reconciles all four memory types automatically across conversation sessions.

#### Snapshot-Based Extraction

As conversations progress, the platform records individual turns, tool calls, and execution responses in short-term memory. When a session reaches a configured message-count threshold or remains idle for a specified duration, a background process groups the eligible turns into a discrete session snapshot.

Independent LLM extraction handlers then evaluate that snapshot in parallel for each enabled memory type:
- An episodic handler synthesizes key incident narratives and outcomes.
- A procedural handler identifies and formats reusable multi-step runbooks.
- A semantic handler extracts technical facts, user constraints, and environment details.
- A taxonomic handler identifies domain-specific terminology, code names, and aliases.

A long, evolving conversation can produce multiple snapshots over time. Because this extraction runs asynchronously in the background, new long-term memories become available for future retrieval without blocking active conversation turns or adding latency to user requests.

#### Embedding and Knowledge Reconciliation

Once new candidate memories are extracted, the platform generates vector embeddings to compare them against existing records in the memory store. To prevent knowledge drift, conflicting information, or duplicate entries, the extraction pipeline reconciles the incoming memories through one of three deterministic actions:

- **ADD:** Creates a new memory record when incoming information represents novel knowledge not currently present in the store.
- **UPDATE:** Creates a new version when incoming information supersedes or updates existing knowledge, marking the prior version as deprecated. For example, if a developer switches their service database from PostgreSQL 15 to PostgreSQL 16, the system updates the semantic fact rather than maintaining conflicting versions.
- **REINFORCE:** Strengthens an existing memory when new interactions repeat or support it, incrementing a reinforcement counter instead of creating redundant duplicates.

This continuous reconciliation loop ensures that the agent's long-term memory remains clean, accurate, and aligned with the current state of knowledge as teams and codebases evolve.

---

## 3. Memory Scope and Authorization

When agent memory is deployed across an enterprise, retrieval mechanisms must respect strict organizational boundaries. Not every memory should be visible to every user.

MongoDB Atlas Agent Engine implements project-level isolation combined with two baseline visibility scopes: `private` and `org`.

| Memory Type | Default Scope | Who Can Read It? |
| :--- | :--- | :--- |
| **Episodic** | `private` | Associated user only |
| **Procedural** | `private` | Associated user only |
| **Semantic** | `private` | Associated user only |
| **Taxonomic** | `org` | All users within the same project |

The platform automatically filters all memory reads based on the caller's authenticated user ID and project token. This separation is crucial: a semantically relevant memory from vector search must not leak across user boundaries simply because another employee's access ticket resembles a previous incident.

In his showcase presentation, Pete Johnson discusses how organizational memory sharing can be governed through Role-Based Access Control (RBAC). MongoDB Atlas Agent Engine currently establishes this foundation through `private` and `org` scopes, giving applications the core primitives needed to structure access. Systems requiring finer department-level or role-specific access controls can enforce additional authorization rules in the application layer before information reaches the model's context window.

---

## 4. The Recap Loop: Efficiently Using the Context Window

A major challenge in building memory-enabled agents is determining how much information should enter the model's context window. Simply retrieving every available memory defeats much of the purpose of having structured long-term memory.

In [O'Reilly's Expert Showcase: Agent Memory](https://www.oreilly.com/live-events/expert-showcase-agent-memory/0642572440619/), MongoDB's Pete Johnson presents an architecture called the **Recap Loop**. The central principle is to evaluate whether the agent already possesses sufficient information to answer a user's question before retrieving additional context from external tools, documents, or databases.

<figure>
    <img src="/assets/images/MongoDB Agent Memory Recap Loop .png" alt="MongoDB Agent Memory Recap Loop Architecture" style="max-width:100%;height:auto;display:block;margin:1.5rem auto;border-radius:8px;">
    <figcaption>Pete Johnson's Recap Loop Architecture — Evaluating prior answers before retrieving additional context.<br><small style="color: var(--text-muted);">Diagram adapted from Pete Johnson's talk at <a href="https://learning.oreilly.com/videos/expert-showcase-agent/0642572428136/?utm_campaign=what-should-an-agent-remember&utm_medium=referral&utm_source=www.turingpost.com" target="_blank" rel="noopener">O’Reilly’s Expert Showcase: Agent Memory</a>.</small></figcaption>
</figure>

### How the Recap Loop Works

Consider an employee asking:
> *\"How did we resolve my previous analytics database access issue?\"*

The Recap Loop executes through a structured, multi-stage retrieval strategy:

1. **Episodic Retrieval:** The system first searches episodic memory for potentially relevant prior answers and case summaries.
2. **Sufficiency Evaluation:** An **Evaluative LLM** acts as an upstream context gatekeeper. It examines the retrieved episodic memories against the incoming prompt to determine whether the existing record is sufficient to fully answer the query.
3. **Selective Execution Paths:**
   - **Path A (Prior answer is sufficient):** If the Evaluative LLM determines that a complete answer already exists in memory, the system uses that episodic context directly. The generative model answers immediately without loading extra procedures, facts, external documents, or tool results.
   - **Path B (Prior answer is insufficient):** If the prior memories do not adequately address the query—for example, if the employee asks *\"My database access expired again, how do I renew it under the current policy?\"*—the Evaluative LLM triggers context expansion. Only then does the agent pull in the necessary procedural workflows, relevant semantic facts, taxonomic aliases, and live policy documentation via RAG and external tools.
4. **Final Generation:** The synthesized context is supplied to the generative LLM, which produces a well-grounded response.

The power of this architecture lies in selectively constructing context rather than forcing the generative LLM to parse every potentially relevant piece of information on every turn.

### Why This Matters for Context Efficiency

LLM context windows are limited, and processing larger token volumes directly increases computational cost, introduces latency, and raises the risk of "needle-in-a-haystack" retrieval failure. 

The Recap Loop provides an intentional mechanism to rein in context expansion. Rather than eagerly dumping all knowledge into the prompt upfront, the agent reuses past solutions whenever possible and expands context only when necessary. In this sense, memory isn't just about retaining knowledge—it is a tool for context optimization.

---

## 5. MongoDB Atlas Agent Engine: Bringing Agent Memory Into Applications

MongoDB Atlas Agent Engine provides managed infrastructure for building and operating AI agents, combining runtime integration, persistent memory, semantic indexing, and observability into a single platform. Its memory service allows agents to retain durable knowledge across sessions, enabling developers to integrate persistent memory without building extraction, reconciliation, and vector indexing pipelines from scratch.

### Connecting to the Memory Service

MongoDB provides a Python SDK for interacting with its memory service. A standalone application initializes the client using a project identifier and access token, then binds it to a specific user and session:

```python
from agent_engine_sdk_memory import (
    Memory,
    MemoryRequestContext,
)

memory = Memory(
    api_key="<your-access-token>",
    project_id="<your-project-id>",
)

chat = memory.bind(
    MemoryRequestContext(
        user_id="employee_123",
        session_id="support_session_456",
    )
)
```

Once bound, all memory interactions automatically inherit the user and project context boundaries.

### Retrieving Only the Required Memory Types

MongoDB provides a `build_context` method that retrieves matching memories, deduplicates results, and assembles a prompt-ready context:

```python
context = chat.build_context(
    query="Help the user renew database access",
    enabled_sources={
        "episodic",
        "semantic",
        "procedural",
    },
)
```

The context builder ranks results and trims assembled context according to a configured token budget. By default, context assembly queries episodic and semantic memory; procedural and taxonomic memory can be selectively enabled based on the agent's task. This selective retrieval directly supports the principles of the Recap Loop. However, note that while `build_context` provides prompt assembly, the upstream sufficiency check by the Evaluative LLM remains an application-level orchestration decision.

### Automatic Background Extraction

In addition to explicit API calls, Atlas Agent Engine supports automatic memory extraction from short-term conversation logs:

```yaml
memory:
  extraction:
    enabled:
      - semantic
      - episodic
      - procedural
      - taxonomic
```

Extraction runs asynchronously, ensuring that conversational turns remain low-latency while long-term memory is continuously indexed and reconciled in the background.

### Choosing an Integration Point

Pete Johnson's presentation highlights three integration layers where developers can adopt Atlas Agent Engine:

- **Agentic runtime:** The framework and execution environment (such as LangGraph, CrewAI, or custom orchestration) driving the agent's logic.
- **Agentic memory:** The persistent memory infrastructure capturing turns, extracting episodes, reconciling facts, and enforcing isolation.
- **Retrieval:** The application's mechanisms for fetching contextual information through memory search or RAG.

Applications can adopt these layers independently. For example, an existing agent running on LangGraph can retain its current LLM runtime while offloading cross-session memory and the Recap Loop's episodic evaluation to Atlas Agent Engine.

> **Note:** Atlas Agent Engine is currently in Public Preview, recommended for prototyping and architectural evaluation ahead of production enterprise rollout.

---

## Closing Thoughts

Building effective agent memory isn't just about expanding context windows or retaining larger chat histories. An agent needs to distinguish previous experiences from reusable procedures, persistent facts, and domain terminology. It also needs a principled method to decide which memories to bring into the prompt.

Pete Johnson's Recap Loop demonstrates how structured memory can make context windows significantly more efficient: start with episodic memory, check whether a prior answer is sufficient, and expand context into procedural memory, semantic facts, tools, and RAG only when necessary. Combined with purpose-built infrastructure like MongoDB Atlas Agent Engine, developers can build agents that don't just process queries in isolation, but intelligently learn and retain knowledge over time.

As I reflect on this architecture, one open question I keep coming back to is: **How do we reliably gauge the performance of the Evaluative LLM to know whether it really reduces cost, or are there false positives where it prematurely accepts an outdated or mismatched past answer?** It's a nuance I'm eager to explore and benchmark further as persistent memory patterns evolve.

---

## References

1. Pete Johnson, MongoDB — [Expert Showcase: Agent Memory, O'Reilly](https://www.oreilly.com/live-events/expert-showcase-agent-memory/0642572440619/).
2. Ksenia Se, Turing Post — [What Should an Agent Remember?](https://www.turingpost.com/p/agent-memory-experience-organizational-belief?gift_content=79cdbae5-993e-4ad0-a82a-d374cbd47d00), October 17, 2026.
3. MongoDB Documentation — [Atlas Agent Engine](https://www.mongodb.com/docs/agentengine/).
4. MongoDB Documentation — [Agent Memory Types](https://www.mongodb.com/docs/agentengine/add-features/memory-types/).
5. MongoDB Documentation — [Memory Extraction & Reconciliation](https://www.mongodb.com/docs/agentengine/add-features/memory-extraction/).

*Note: The Recap Loop diagram is an original presentation slide courtesy of Pete Johnson (MongoDB) from the [O’Reilly Expert Showcase](https://learning.oreilly.com/videos/expert-showcase-agent/0642572428136/?utm_campaign=what-should-an-agent-remember&utm_medium=referral&utm_source=www.turingpost.com).*
