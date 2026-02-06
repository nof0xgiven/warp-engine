# Warp Engine 🚀

A coding agent architecture that treats context as a viewport, not a backpack.

## The Problem

Traditional coding agents stuff the context window until it overflows, then summarize. Research shows this is backwards—LLMs perform best at ~50% context utilization, with performance degrading significantly as input length increases ("Lost in the Middle" effect).

## The Solution

**Warp Engine** dynamically assembles fresh, focused context for every API call:

- **Event-sourced state store** — All actions, decisions, and discoveries logged as structured events
- **Graph-based code selection** — Tree-sitter AST + PageRank identifies the *right* code, not just *more* code  
- **Prompt-cache-aware layering** — Stable prefix cached at 90% cost reduction, dynamic context assembled fresh
- **50% utilization ceiling** — A feature, not a limitation. Keeps the model in its highest-quality attention regime

## Architecture
```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   State Store   │────▶│ Context Assembler │────▶│    LLM Call     │
│  (Event Log)    │     │   (Projection)    │     │  (Fresh Each)   │
└─────────────────┘     └──────────────────┘     └─────────────────┘
        ▲                                                 │
        └─────────────────────────────────────────────────┘
                         Tool Execution → Events
```

## Inspired By

- Claude Code's sub-agent isolation & auto-compaction
- Aider's PageRank repo mapping
- OpenHands' event-sourced condenser system
- Blackboard architecture patterns (Han et al. 2025)
- The "Ralph Wiggum Loop" — context lives in files, not the window

## Status

🔬 Research validated, implementation in progress.