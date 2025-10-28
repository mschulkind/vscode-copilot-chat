# Anthropic Adapter - Context Window Scaling

## Problem

The agent mode is designed to work with a 200k token context window, but some models (especially smaller or cheaper ones) have smaller context windows. When running on a model with a smaller context window (e.g., 128k), the agent might:

1. Run out of context too quickly
2. Make poor decisions about what to keep
3. Fail multi-step tasks that require more context

## Solution: Token Usage Scaling

**Location:** `src/extension/agents/node/adapters/anthropicAdapter.ts`

The Anthropic adapter implements a clever solution: **lie to the agent about how much context it has used**.

### How It Works

```typescript
private adjustTokenUsageForContextWindow(context: IStreamingContext, usage?: APIUsage): APIUsage {
	// If we don't have usage, return defaults
	if (!usage) {
		return {
			prompt_tokens: 0,
			completion_tokens: 0,
			total_tokens: 0
		};
	}

	// Skip adjustment for certain models
	if (context.endpoint.modelId === 'gpt-4o-mini') {
		return usage;
	}

	const realContextLimit = context.endpoint.modelMaxPromptTokens;  // e.g., 128000
	const agentAssumedContextLimit = 200000;  // The agent thinks it has 200k tokens

	// Calculate scaling factor to make the agent think it has a larger context window
	const scalingFactor = agentAssumedContextLimit / realContextLimit;  // 200000 / 128000 = 1.5625

	const adjustedPromptTokens = Math.floor(usage.prompt_tokens * scalingFactor);
	const adjustedCompletionTokens = Math.floor(usage.completion_tokens * scalingFactor);
	const adjustedTotalTokens = adjustedPromptTokens + adjustedCompletionTokens;

	return {
		...usage,
		prompt_tokens: adjustedPromptTokens,
		completion_tokens: adjustedCompletionTokens,
		total_tokens: adjustedTotalTokens,
	};
}
```

### Example

**Scenario:** Agent running on a 128k token context model

**Real usage after 3 tool calls:**
- Prompt tokens: 50,000
- Completion tokens: 5,000
- Total: 55,000 tokens
- **Percentage used: 43% of 128k**

**Reported usage to agent:**
- Prompt tokens: 50,000 * 1.5625 = 78,125
- Completion tokens: 5,000 * 1.5625 = 7,812
- Total: 85,937 tokens
- **Percentage used: 43% of 200k** (same percentage!)

### Why This Works

1. **Proportional Awareness**: The agent sees the same percentage of context used
2. **Early Warnings**: Agent starts conserving context earlier
3. **Smoother Degradation**: Agent doesn't suddenly hit limits
4. **Consistent Behavior**: Agent logic works the same regardless of actual model

### When Scaling Happens

The adjustment is applied:

1. **During streaming** (in SSE events)
2. **At message start**
3. **At message completion**
4. **In delta events**

```typescript
override async *handleSSEMessage(chunk: CopilotStreamEvent, context: IStreamingContext): AsyncIterable<IStreamEventData> {
	if (chunk.type === 'usage') {
		// Adjust usage before yielding
		const adjusted = this.adjustTokenUsageForContextWindow(context, chunk.usage);
		yield { type: 'usage', usage: adjusted };
	}
	// ... handle other event types
}
```

## Model-Specific Behavior

### GPT-4o-mini Exception

```typescript
if (context.endpoint.modelId === 'gpt-4o-mini') {
	return usage;  // No adjustment
}
```

**Why?** GPT-4o-mini is optimized for cost and speed. The agent should be aware of its actual smaller context to make appropriate decisions.

## Trade-offs

### Advantages
- ✅ Agent can run on any model
- ✅ Consistent behavior across models
- ✅ Graceful degradation
- ✅ No agent code changes needed

### Disadvantages
- ❌ Slight inaccuracy in token reporting
- ❌ May be conservative on larger models
- ❌ Telemetry shows adjusted (not real) usage

## Alternative Approaches Considered

### 1. Dynamic Prompt Sizing
- Adjust agent prompts based on model
- **Rejected:** Too complex, many edge cases

### 2. Model-Specific Agents
- Different agent implementations per model
- **Rejected:** Maintenance burden

### 3. Aggressive Context Management
- More aggressive pruning of history
- **Rejected:** May lose important context

## Implementation Details

### Integration Points

1. **Stream Processor**: Adjusts usage in real-time
2. **Message Events**: Adjusts in message_start, message_delta, message_stop
3. **Content Blocks**: Adjusts in content_block_delta
4. **Final Response**: Adjusts in final completion

### Token Usage Structure

```typescript
interface APIUsage {
	prompt_tokens: number;     // Input tokens
	completion_tokens: number; // Output tokens
	total_tokens: number;      // Sum of above
}
```

### Anthropic-Specific Events

Anthropic models emit these event types:
- `message_start`: Beginning of response
- `content_block_start`: Start of content block
- `content_block_delta`: Incremental content
- `content_block_stop`: End of content block
- `message_delta`: Message metadata updates
- `message_stop`: End of response
- `ping`: Keep-alive

Each event that includes usage gets adjusted.

## Monitoring and Telemetry

```typescript
// Real usage is logged internally for debugging
this._logService.debug(`Real usage: ${usage.prompt_tokens} prompt, ${usage.completion_tokens} completion`);
this._logService.debug(`Adjusted usage: ${adjustedPromptTokens} prompt, ${adjustedCompletionTokens} completion`);

// Telemetry gets adjusted values
this._telemetryService.sendGHTelemetryEvent('agent.token.usage', {
	model: context.endpoint.modelId,
	promptTokens: adjustedPromptTokens,
	completionTokens: adjustedCompletionTokens,
});
```

## Future Improvements

1. **Per-Model Scaling Factors**
   - Different factors for different model families
   - Learn optimal factors from usage data

2. **Dynamic Scaling**
   - Adjust factor based on task complexity
   - More aggressive for simple tasks, conservative for complex

3. **User Configuration**
   - Allow users to tune scaling behavior
   - Power users can optimize for their use cases

## Related Files

- Anthropic adapter: `src/extension/agents/node/adapters/anthropicAdapter.ts`
- Protocol adapter interface: `src/extension/agents/common/protocolAdapter.ts`
- Agent prompt logic: `src/extension/agents/node/agentPrompt.tsx`
