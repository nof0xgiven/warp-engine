# Building a coding agent with living, breathing context

**"warp engine" intuition — fresh context assembled around the agent's current position each turn — is validated by converging research from Anthropic, Chroma, and multiple academic papers proving that curated, compact context dramatically outperforms growing conversation history.** The approach is architecturally sound and aligns with the natural statelessness of LLM APIs, but no production tool fully implements it yet. The closest implementations are OpenHands' pluggable RollingCondenser, Anthropic's multi-session "Ralph Wiggum Loop" pattern, and Factory.ai's progressive context stack. The key design challenge isn't whether to do it — it's how to maintain coherence across turns while maximizing prompt cache hit rates. This report covers every dimension of the architecture space, from blackboard systems to graph-based context selection, with specific papers, tools, and code to draw from.

## The evidence against growing conversation history

Chroma Research's "Context Rot" study (July 2025) evaluated **18 LLMs including GPT-4.1, Claude 4, and Gemini 2.5** and found that performance degrades significantly and non-linearly as input length increases — even for simple tasks. Models perform best at roughly **10K tokens even when they support 200K-token windows**. The seminal "Lost in the Middle" paper (Liu et al., TACL 2024) documented a U-shaped performance curve: models recall information at the beginning and end of context but performance drops **over 30%** for information positioned in the middle. Anthropic's own engineering team describes "context rot" as a fundamental constraint — as token count increases, the model's attention budget depletes across the n² pairwise relationships in transformer attention.

These findings validate your core premise. A growing conversation history that fills to 80-90% utilization before compaction is fighting against the architecture of transformers themselves. Anthropic's context engineering guide (September 2025) states the principle directly: **"Find the smallest possible set of high-signal tokens that maximize the likelihood of some desired outcome."** Claude Code's own team found that compacting earlier — at 75% rather than 90% — paradoxically extends productive sessions because reasoning quality is maintained. Multiple practitioners confirm the sweet spot is **50-70% context utilization**, which aligns perfectly with your ~50% target.

The practical implication is stark: a focused 300-token context often outperforms an unfocused 113,000-token context (LangChain's context engineering analysis, July 2025). Your instinct to dynamically assemble minimal, high-signal context rather than accumulate everything is the correct architectural direction.

## How existing coding agents actually manage context

No production coding agent implements true rolling context where each API call is constructed entirely fresh. The industry has converged on **compaction/summarization** as the primary strategy — effectively "summarize and replace" rather than a rolling window. Here's how the major tools work:

**Claude Code** sends the full conversation history (system prompt + CLAUDE.md + all tool calls and observations) on each turn within a ~200K token window. Auto-compact triggers at 75-92% utilization, where the model summarizes the conversation while preserving architectural decisions and the 5 most recently accessed files. The summary replaces the full history. Sub-agents via `Task()` and `Explore()` run in isolated context windows and return only condensed summaries (1,000-2,000 tokens) — this is the closest thing to "fresh session per call" in the Claude ecosystem.

**Aider** comes closest to your vision in one specific dimension: its **repo map is rebuilt from scratch each turn** using tree-sitter AST parsing and a PageRank algorithm that ranks files/symbols by cross-reference importance, then uses binary search to fit the most relevant content within a configurable token budget. This per-call re-ranking is genuine dynamic context assembly. However, conversation history still grows linearly without compaction.

**SWE-agent** (Princeton) uses purpose-built "history processors" that actively prune old observations at each step — removing redundant error messages, stripping outdated file views, and limiting the observation window. Their ablation testing a "Last 5 Observations" rolling window showed this approach works. The windowed file viewer shows only ~100 lines at a time, and ablation testing proved this 100-line window optimal over both 30-line and full-file alternatives.

**OpenHands** is architecturally the most interesting. It uses an event-sourced state model with a **pluggable condenser system** offering multiple strategies: observation masking (replacing older observations with placeholders), LLM summarization, explicit rolling window (`RollingCondenser`), and truncation fallback. Context condensation reduced their average per-turn API cost by **over 50%** and changed scaling from quadratic to linear. This pluggable architecture is the most flexible foundation in the ecosystem.

**Codex CLI** (OpenAI) takes a unique approach with "native compaction" — GPT-5.2-Codex is trained at the model level to manage its own context budget. Cloud Codex tasks each run in a **fresh, isolated sandbox** preloaded with the repo, making them effectively "fresh session per call" by design.

The emerging consensus pattern across all tools is what practitioners call **"Externalize, Then Refresh"**: write state to files or git (progress docs, TODOs, commit messages), compact or clear the context window, then reload from external state. Claude Code power users explicitly practice "Document & Clear" — dump plan and progress to a markdown file, `/clear`, restart fresh.

## The blackboard architecture maps directly to your design

The blackboard pattern — where multiple knowledge sources write to shared state and the agent reads a projection of it — has been validated for LLM systems in two recent papers. Han et al. (July 2025, arXiv:2507.01701) built a blackboard system with an LLM-based control unit that dynamically selects which agents participate each round based on current blackboard content. Their system achieved the best average performance (**81.68%** across MMLU, GPQA, and MATH) while being more token-economical than both static and dynamic multi-agent baselines.

Salemi et al. (September 2025, arXiv:2510.01285) took a different approach where subordinate agents **volunteer** to respond based on capabilities rather than being assigned by a coordinator. Their critical design decision: responses are NOT written back to the blackboard, preventing one agent's output from negatively influencing others. This produced **13-57% improvement** over master-slave paradigms.

For a coding agent, the blackboard maps naturally to the "warp engine" concept. Multiple knowledge sources — static analysis, test runner, documentation retriever, error parser, git history analyzer — asynchronously write to a shared state store. Each turn, the coding agent's context is assembled from a projection of this blackboard filtered by relevance to the current task. The blackboard IS the state store; the projection IS the dynamic context assembly. Knowledge sources write to specific sections (code analysis results, test results, documentation excerpts, error logs, git context, user intent), and the context assembler selects which sections to include based on what the agent is currently doing.

The stigmergy variant (from KeepALifeUS/autonomous-agents on GitHub) goes further: agents coordinate entirely through shared files with zero direct communication, claiming **80% token reduction** versus traditional multi-agent approaches. Tasks live in `queue.json`, reviews in `pending/`, knowledge accumulates in `patterns.jsonl`. Git's conflict detection serves as free distributed locking. This is radically decentralized — adding agents requires no architectural changes.

## Graph-based context selection is the key to dynamic assembly

The most critical question for your architecture is: given a current task state, which files, functions, and documentation should be loaded into context? Graph-based approaches provide the strongest answer.

**Aider's PageRank approach** is the proven baseline. Tree-sitter parses all source files to extract symbol definitions, builds a dependency graph where files are nodes and cross-file references are edges, then applies PageRank to identify the most-referenced symbols. Binary search fits the maximum important content within a configurable token budget (default 1,024 tokens). The repo map dynamically expands when no files are in context and contracts when specific files are loaded.

**CodexGraph** (NAACL 2025, arXiv:2408.03910) stores the code graph in Neo4j with typed nodes (MODULE, CLASS, FUNCTION) and edges (CONTAINS, INHERITS, HAS_FIELD, USES). The agent writes natural language queries that get translated to Cypher graph queries. This outperforms pure similarity-based retrieval for tasks requiring **multi-hop reasoning** across codebases — understanding a function requires understanding the types it uses, which requires understanding their definitions, which requires understanding their dependencies.

**RepoGraph** (arXiv:2410.14684) goes to line-level granularity — each node is a line of code, edges are definition/reference dependencies. Sub-graph retrieval uses ego-graphs centered around task-relevant keywords. **GraphCoder** (ASE 2024, arXiv:2406.07003) uses Code Context Graphs capturing control-flow, data-dependence, and control-dependence at the statement level, with coarse-to-fine retrieval that first finds structurally similar code via graph matching, then refines with lexical similarity.

The **cAST chunking paper** (arXiv:2506.15655) demonstrates that AST-based code chunking for retrieval outperforms naive fixed-size splitting by **+5.5 points** on RepoEval and **+2.7 points** on SWE-bench. This is a drop-in improvement for any RAG pipeline.

For your architecture, the optimal approach combines structural graph traversal (for dependency chains and call graphs) with semantic embedding search (for conceptually related code). The graph answers "what is structurally connected to what I'm working on" while embeddings answer "what is conceptually similar to my current task." **Prometheus** (arXiv:2507.19942) demonstrates this hybrid approach using tree-sitter for graph construction, Neo4j for storage, and LLM-based filtering/ranking for retrieval — achieving the first appearance on SWE-bench Multilingual.

## The optimal architecture: event-sourced blackboard with dynamic projection

Synthesizing all the research, your "warp engine" is best implemented as an event-sourced blackboard system with graph-based context projection. Here is the architecture:

**The state store** (blackboard) maintains all accumulated knowledge as structured data: current task and subtask decomposition, files modified with diffs and reasoning, errors encountered and resolutions, test results (pass/fail/coverage), discovered architectural patterns and dependencies, key decisions with rationale, and a code structure graph (maintained via tree-sitter). This is append-only and event-sourced — every tool execution, every test result, every file edit generates an event.

**The context assembler** (projection engine) runs before each API call, constructing a fresh prompt within your ~50% token budget. It operates in layers with a priority hierarchy:

- **Stable prefix** (~15% of window, prompt-cached): System instructions, tool definitions, active skill definitions, project rules. This stays identical across calls to maximize Anthropic's prompt cache hits at **90% cost reduction** for cached reads.
- **Semi-stable middle** (~10%, cached with 1-hour TTL): Repository overview, architectural context generated from the code graph, current task description.
- **Dynamic context** (~25%, assembled fresh): Relevant code files selected via the graph + embedding hybrid retrieval, recent progress summary distilled from the event log, active errors and test failures, documentation retrieved just-in-time based on APIs being used.
- **Reserved for output** (~50%): Model reasoning and generation space.

**The relevance scorer** determines what enters the dynamic context portion. For each candidate piece of context (a file, a function, a documentation section, an error trace), it computes relevance based on: graph distance from the current working file(s), recency of access or modification, semantic similarity to the current subtask description, and explicit dependency relationships. Items are ranked, and the assembler greedily fills the budget with the highest-scoring items, using binary search (Aider's technique) to maximize signal within the token limit.

**The control loop** follows an OODA-inspired pattern rather than plan-then-execute. Each turn: **Observe** (read the blackboard projection — what changed since last turn?), **Orient** (assess the current state against the goal — what matters most right now?), **Decide** (choose the single highest-value action), **Act** (execute it and emit events to the state store). The agent does not maintain a rigid multi-step plan. Instead, it maintains a lightweight task description that gets updated when circumstances change — what the research calls "scoped replanning" where only the next 1-2 steps are concrete.

## Continuous replanning beats plan-then-execute, but not every turn

Research from arXiv:2509.03581 ("Learning When to Plan") provides the definitive answer on replanning frequency: agents should **plan strategically, not constantly**. Always-replan approaches cause behavioral instability analogous to frame-skipping in reinforcement learning. The optimal strategy is to replan when a tool fails, when test results are unexpected, when new information fundamentally changes the task understanding, or when a subtask completes. Otherwise, continue executing the current intent.

This finding directly informs your architecture. The "warp engine" should not recompute the entire context strategy every turn — it should recompute what context to *load* (that's cheap, just a relevance ranking) while keeping the strategic direction stable unless something changes. Anthropic's long-running agent harness paper demonstrates this at the session level: an initializer agent creates a feature list and progress file, then coding agents make incremental progress per session, each reading the progress file and choosing ONE feature to work on.

The **12-Factor Agents methodology** (18K GitHub stars, HumanLayer) explicitly warns against rigid plan-then-execute: "Most builders pushed the 'tool calling loop' idea to the side when they realized that anything more than 10-20 turns becomes a big mess that the LLM can't recover from." Their recommendation: keep agents focused on small 5-10 step workflows, use deterministic code for the broader DAG, and treat context as something you actively own and engineer rather than passively accumulate.

## State machines provide guard rails, reactive streams provide flexibility

The research clearly shows that pure state machines (XState/FSM) are too rigid for open-ended coding — they require pre-defining all states and transitions. But pure reactive/event-driven approaches lack the predictability needed for reliable agent behavior. The **StateFlow** paper (arXiv:2403.11322) demonstrated that FSM-structured LLM workflows achieve **4-6x cost reduction** with improved performance on coding tasks. The Stately Agent framework (XState + LLM) lets the model choose which transition to take from the set of valid transitions at the current state.

The optimal approach for your harness is a **hybrid**: use a lightweight state machine for high-level phase management (understanding → implementing → testing → refining) while using reactive/event-driven patterns within each phase for the actual work. LangGraph's graph-based approach with conditional edges and cycles is the closest production framework to this hybrid. BoundaryML's SageKit (November 2025) demonstrates an event-sourced agent loop where all interactions are modeled as a single append-only event stream, with different consumers receiving projections tailored to their needs — exactly the blackboard projection pattern.

The key insight from Braintrust's analysis (August 2025) is that Claude Code, OpenAI Agents SDK, and most successful production agents share the same core: `while (!done) { callLLM() → execute tools → loop }`. Tool responses make up **67.6%** of total tokens; system prompts only 3.4%. Complexity is absorbed at the edges — in tool design and context engineering — while the core loop stays simple. Your "warp engine" should embrace this simplicity at the loop level while investing complexity in the context assembly layer.

## Prompt caching makes the fresh-session pattern economically viable

The biggest potential objection to fresh context assembly per call is cost. Anthropic's prompt caching eliminates this concern with a specific constraint: the stable portion of your prompt must be identical across calls. Cached reads cost **10% of base input token price** (90% savings) with up to **85% latency reduction**. Cache TTL is 5 minutes (refreshed on each use) or 1 hour on newer models. Processing order is Tools → System → Messages, so changing tools invalidates everything after.

This directly shapes your architecture. Structure the prompt so the stable prefix (system instructions, tool definitions, skills, rules) remains identical across calls — this gets cached. The dynamic portion (current task state, relevant files, recent progress) goes at the end. With up to **4 cache breakpoints** per prompt, you can cache at multiple granularity levels: tools, system prompt, project overview, and semi-stable task description.

Moncef Abboud's DynamicContextLoading pattern demonstrates three-level progressive loading: Level 1 loads only server/module descriptions, Level 2 loads tool summaries on-demand, Level 3 loads full tool schemas only when actively using a tool. Since schemas often represent **60-80%** of token usage in static toolsets, this progressive approach delivers massive savings. Speakeasy's dynamic toolsets achieved a **96% reduction in input tokens** using progressive and semantic search for tool discovery.

## The emerging best practices point toward your architecture

Lance Martin's comprehensive synthesis (January 2026) of agent design patterns identifies seven core patterns, and nearly all of them support the warp-engine approach. **"Offload Context"** — push context from the window to the filesystem; Manus writes old tool results to files, Cursor offloads tool results and trajectories. **"Cache Context"** — prompt caching is "the most important metric" per the Manus team. **"Isolate Context"** — sub-agents with isolated context windows prevent pollution. **"Progressive Disclosure"** — don't load all tool definitions upfront; let the agent discover what it needs.

The **"Ralph Wiggum Loop"** pattern (coined by Geoffrey Huntley, endorsed by Anthropic) is structurally identical to your concept: run agents repeatedly with fresh context until a plan is satisfied, with progress communicated across sessions via git history and plan files. Context lives in files, not the context window. Each session starts fresh, reads the plan file, makes incremental progress. This has been proven to handle **300K LOC Rust codebases** and "ship a week's worth of work in a day."

Factory.ai's production system implements what they call a "context stack" that progressively distills "everything the company knows" into "exactly what the Droid needs right now." Their explicit principle: "Effective agentic systems must treat context the way operating systems treat memory and CPU cycles: as finite resources to be budgeted, compacted, and intelligently paged." Tongyi's DeepResearch uses an "IterResearch paradigm" where each round reconstructs a streamlined workspace using only essential outputs from the previous round, preventing what they call "cognitive suffocation."

## Concrete implementation blueprint

Based on all this research, here is the specific architecture for your warp engine:

**State Store** uses an append-only event log (like OpenHands' event-sourced model). Events include: `TaskStarted`, `FileRead`, `FileEdited`, `TestRun`, `ErrorEncountered`, `DecisionMade`, `SubtaskCompleted`. Reducers compute derived state: current active files, unresolved errors, progress percentage, architectural understanding. Store this in a local SQLite or similar — it needs to be fast, not distributed.

**Code Graph** is maintained by a background process using tree-sitter. On every file change, incrementally update the AST, symbol table, and dependency edges. Store in an in-memory graph (or Neo4j for large repos). Maintain PageRank scores that update incrementally. This gives you the structural relevance signal.

**Context Assembler** runs before each API call. It takes as input: the current subtask description, the code graph with PageRank scores, the event log, and the token budget. It outputs a single prompt. The assembly algorithm: (1) always include the stable prefix (cached), (2) generate a progress summary by reducing recent events, (3) query the code graph for files/functions within N hops of the current working location, weighted by PageRank, (4) query the embedding index for semantically similar code to the current subtask, (5) merge and rank all candidates, (6) greedily fill the dynamic budget, (7) place highest-priority items at the beginning and end (lost-in-the-middle mitigation).

**The Agent Loop** is a simple while loop. Each iteration: assemble context → call LLM → parse tool calls → execute tools → emit events → check if done. No framework needed. The 12-Factor Agents principle applies: "Own your control flow." Use deterministic code for routing, LLMs for reasoning within a turn.

**Skills and rules** are loaded dynamically based on relevance. Maintain a registry of available skills (file-based, with YAML frontmatter per the Anthropic Skills Standard at agentskills.io). Each skill has a description embedding. When assembling context, compute similarity between the current task and all skill descriptions, load only the top-K relevant skills.

## Conclusion

The research converges on a clear signal: **context engineering is the central challenge** of building effective coding agents, and your instinct to treat the context window as a dynamically-assembled viewport rather than a growing log is architecturally correct. The transformer attention mechanism itself penalizes long, unfocused context. Every major coding agent team has independently discovered this — Claude Code compacts at 75%, Aider rebuilds its repo map each turn, OpenHands built pluggable condensers, Factory.ai treats context like OS memory paging.

The novel contribution of your "warp engine" design would be combining several proven techniques that no single tool has unified: event-sourced state (OpenHands), graph-based code context selection (Aider/CodexGraph), blackboard-style multi-source knowledge assembly (Han et al. 2025), progressive context loading (Factory.ai/DynamicContextLoading), and prompt-cache-aware context layering (Manus). The result is an agent where every API call is surgically constructed from a rich state store rather than accumulated from conversation history — and where the **50% utilization ceiling** is a feature, not a limitation, because it keeps the model operating in its highest-quality attention regime.

The riskiest assumption is coherence without conversational continuity. Mitigate this with: (1) a structured progress summary generated from the event log each turn (not LLM-summarized — deterministically reduced), (2) key decisions stored as first-class events with reasoning, and (3) the code graph providing implicit continuity through structural relationships. The agent doesn't need to remember the conversation — it needs to understand the codebase and the current state of the task. Those are different things, and your architecture optimizes for the right one.
