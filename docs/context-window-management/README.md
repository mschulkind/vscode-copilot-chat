# Context Window Management - Complete Reference

This directory contains comprehensive documentation on how the GitHub Copilot Chat extension manages context windows and token budgets.

## Quick Navigation

### Core Concepts
1. **[Overview](./01-overview.md)** - Start here for an introduction to context window management, the 128k limit problem, and the overall architecture.

### Technical Deep Dives
2. **[Token Counting](./02-token-counting.md)** - How tokens are counted using TikToken, caching strategies, and approximation techniques.

3. **[Prompt TSX Priorities](./03-prompt-tsx-priorities.md)** - The priority-based rendering system that automatically prunes low-priority content when budgets are exceeded.

4. **[Workspace Search Budgets](./04-workspace-search-budgets.md)** - How workspace search strategies respect token limits, including full workspace vs. TF-IDF vs. semantic search.

5. **[Anthropic Adapter](./05-anthropic-adapter.md)** - Special token usage scaling to make agents work consistently across models with different context windows.

6. **[Character Budgets](./06-character-budgets.md)** - Fast character-based approximations used in TypeScript language context and inline chat.

7. **[Model-Specific Limits](./07-model-specific-limits.md)** - How different models (GPT-4, Claude, Gemini) have different limits and how the extension adapts.

## Key Takeaways

### The 128k Problem
Most modern models have 128k-200k token context windows, but:
- Workspaces can easily exceed this
- Conversation history accumulates
- Tool results can be large
- Must prioritize what matters most

### Solution Architecture
The extension uses a multi-layered approach:

1. **Model-Level Limits**: Each model declares its `modelMaxPromptTokens`
2. **Priority System**: Prompt elements have priorities (0-1000+)
3. **Automatic Pruning**: Low-priority content removed when over budget
4. **Component Budgets**: Sub-components get allocated portions
5. **Search Strategies**: Smart workspace search with budget awareness
6. **Character Approximations**: Fast checks before expensive tokenization

### Critical Code Locations

| Feature | File Path |
|---------|-----------|
| Token counting | `src/platform/tokenizer/node/tokenizer.ts` |
| Model limits | `src/platform/endpoint/node/chatEndpoint.ts` |
| Priority rendering | `src/extension/prompts/node/base/promptRenderer.ts` |
| Workspace search | `src/platform/workspaceChunkSearch/node/` |
| Character budgets | `src/extension/typescriptContext/vscode-node/languageContextService.ts` |
| Agent adapter | `src/extension/agents/node/adapters/anthropicAdapter.ts` |

## Common Patterns

### Pattern 1: Pre-Check with Characters

```typescript
// Fast check before expensive tokenization
const charLimit = (endpoint.modelMaxPromptTokens * 4) / 3;
if (content.length > charLimit) {
	return truncate(content, charLimit);
}
```

### Pattern 2: Priority-Based Rendering

```typescript
<SystemMessage priority={1000}>Critical instructions</SystemMessage>
<UserMessage priority={900}>{query}</UserMessage>
<UserMessage priority={500}>Extra context (can be pruned)</UserMessage>
```

### Pattern 3: Token Limit Wrapping

```typescript
<TokenLimit max={28_000}>
	<LargeComponent {...props} />
</TokenLimit>
```

### Pattern 4: Conditional Rendering

```typescript
if (estimatedTokens > sizing.tokenBudget * 0.8) {
	return <ErrorMessage>Content too large</ErrorMessage>;
}
return <FullContent />;
```

### Pattern 5: FlexGrow for Dynamic Allocation

```typescript
<UserMessage priority={900} flexGrow={2}>
	Can expand to use surplus budget
</UserMessage>
```

## Token Budget Guidelines

### Conservative (Recommended)
- System messages: 5% (fixed high priority)
- User query: 3% (fixed high priority)
- Workspace context: 30-40% (flexible)
- Conversation history: 15-20% (flexible)
- Tool results: 10-15% (flexible)
- **Reserve for completion: 30-40%**

### Aggressive (For Large Context Needs)
- System messages: 3% (fixed)
- User query: 2% (fixed)
- Workspace context: 50-60% (flexible)
- Conversation history: 10% (flexible)
- Tool results: 10% (flexible)
- **Reserve for completion: 20%**

## Best Practices

### ✅ DO:
- Use priorities consistently across prompts
- Leave 30%+ for model completion
- Pre-check sizes with characters
- Use TokenLimit for variable content
- Monitor and log actual usage
- Test with small budgets

### ❌ DON'T:
- Use the full `modelMaxPromptTokens`
- Assume all text is 4 chars/token
- Give everything the same priority
- Forget message overhead (3 tokens each)
- Skip tokenization for final checks
- Ignore telemetry about exceeded budgets

## Debugging Token Issues

### Problem: Prompt rejected as too large

**Check:**
1. Log `sizing.tokenBudget` value
2. Count actual tokens in each component
3. Check for duplicate content
4. Verify priority ordering

### Problem: Important context being pruned

**Check:**
1. Priority values (should be 800+)
2. TokenLimit maxes (may be too restrictive)
3. flexGrow allocation
4. Total budget available

### Problem: Performance issues

**Check:**
1. Are you over-tokenizing? (use character checks first)
2. Is LRU cache enabled? (should be)
3. Are you using worker threads? (should be for large ops)
4. Are you caching results?

## Telemetry and Monitoring

Key metrics tracked:
- `token.usage`: Actual prompt and completion tokens
- `fullWorkspaceChunkSearch.perf.searchFileChunks`: Workspace search performance
- `tokenizer.stats`: Tokenization performance
- Custom metrics when budgets exceeded

## Contributing

When adding new features that consume tokens:

1. **Assign appropriate priorities** (see [03-prompt-tsx-priorities.md](./03-prompt-tsx-priorities.md))
2. **Use TokenLimit** for variable-size content
3. **Test with small budgets** (e.g., 1000 tokens)
4. **Log when content is pruned** (for telemetry)
5. **Document budget assumptions** (in code comments)

## Related Documentation

- **Architecture**: `CONTRIBUTING.md` (main repo, lines 130-186)
- **Prompts**: `docs/prompts.md`
- **Tools**: `docs/tools.md`
- **Copilot Instructions**: `.github/copilot-instructions.md`

## Glossary

- **Token**: Smallest unit of text for LLMs (~4 characters)
- **Token Budget**: Maximum tokens allowed in a prompt
- **Priority**: Numeric value (0-1000+) indicating importance
- **Pruning**: Removing low-priority content to fit budget
- **flexGrow**: Property allowing element to expand with surplus budget
- **TokenLimit**: Hard cap on tokens for a component
- **Character Budget**: Character-based approximation of token budget
- **CAPI**: Copilot API (provides model metadata)
- **TikToken**: OpenAI's tokenization library
- **TSX**: TypeScript XML (React-like syntax for prompts)

## Questions?

For questions about context window management:
1. Check this documentation first
2. Search the codebase for examples
3. Look at telemetry data
4. Open an issue with reproduction steps

---

**Last Updated**: 2025-01-27
**Documentation Version**: 1.0
