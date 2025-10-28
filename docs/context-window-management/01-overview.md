# Context Window Management - Overview

## Introduction

The GitHub Copilot Chat extension implements a sophisticated context window management system to ensure that prompts sent to language models (LLMs) stay within token limits. This is critical as different models have different maximum context window sizes (e.g., 128k tokens for GPT-4, 200k tokens for Claude).

## Key Concepts

### Token Budget
The **token budget** is the maximum number of tokens that can be included in a prompt sent to an LLM. This is determined by:

1. **Model Capabilities**: Each model reports its maximum prompt tokens via CAPI (Copilot API)
2. **Experimental Overrides**: Configuration and experiments can override token limits
3. **Reserved Tokens**: Some tokens are reserved for message overhead and completion

### Context Window Components

A typical prompt includes multiple components:
- **System Messages**: Instructions, safety rules, and model behavior guidelines
- **User Messages**: User queries and selected context
- **Assistant Messages**: Previous responses (conversation history)
- **Tool Results**: Output from tool calls (file searches, code analysis, etc.)
- **Context Snippets**: Code from files, workspace context, related files

### The 128k Limit Problem

Models like GPT-4 with 128k token context windows present challenges:
1. **Large Workspaces**: Can easily exceed 128k tokens if all files are included
2. **Long Conversations**: History can accumulate many tokens
3. **Tool Results**: Search results and file contents can be large
4. **Priority Management**: Need to decide what to keep when limits are reached

## Architecture Overview

The extension uses multiple layers of context management:

### 1. Model-Level Token Limits
- Defined in `IChatEndpoint` interface
- Property: `modelMaxPromptTokens`
- Source: CAPI model metadata or configuration overrides

### 2. TSX Prompt Rendering with Priorities
- Framework: `@vscode/prompt-tsx`
- Priority-based pruning system
- Higher priority content kept when budget is exceeded

### 3. Component-Level Budgets
- Individual prompt components can have sub-budgets
- Examples: `TokenLimit`, `flexGrow`, character budgets
- Prevents any single component from dominating

### 4. Workspace Search Strategies
- Full workspace search (only when under budget)
- TF-IDF search with result limits
- Semantic search with chunk prioritization

## Key Files and Locations

| Component | Location | Purpose |
|-----------|----------|---------|
| Endpoint Token Config | `src/platform/endpoint/node/chatEndpoint.ts` | Determines max tokens per model |
| Tokenizer | `src/platform/tokenizer/node/tokenizer.ts` | Counts tokens using TikToken |
| Prompt Renderer | `src/extension/prompts/node/base/promptRenderer.ts` | Renders prompts with budget management |
| Workspace Search | `src/platform/workspaceChunkSearch/` | Budget-aware workspace searching |
| Character Budget | `src/extension/typescriptContext/vscode-node/languageContextService.ts` | Character-based budget tracking |

## How Context is Managed When Approaching Limits

### Phase 1: Prevention
- Pre-calculate token costs before rendering
- Set conservative budgets for sub-components
- Use sampling/summarization for large content

### Phase 2: Prioritization
- Assign priorities to all prompt elements (0-1000+ scale)
- Higher numbers = higher priority
- System messages typically have highest priority (900-1001)

### Phase 3: Pruning
- When budget is exceeded, remove lowest priority elements first
- Preserve message order (can't reorder messages)
- Keep conversation coherence (don't split related content)

### Phase 4: Fallback
- If workspace is too large, skip full workspace inclusion
- Use targeted search instead
- Report size limits to telemetry

## Next Topics

- [02-token-counting.md](./02-token-counting.md) - How tokens are counted
- [03-prompt-tsx-priorities.md](./03-prompt-tsx-priorities.md) - Priority-based rendering system
- [04-workspace-search-budgets.md](./04-workspace-search-budgets.md) - Workspace search and token limits
- [05-anthropic-adapter.md](./05-anthropic-adapter.md) - Special handling for different models
- [06-character-budgets.md](./06-character-budgets.md) - Character-based budget management
