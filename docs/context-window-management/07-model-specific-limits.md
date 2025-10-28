# Model-Specific Token Limits

## Overview

Different language models have different context window sizes and token limits. The extension must adapt to each model's capabilities while providing a consistent user experience.

## Token Limit Configuration

### Primary Source: CAPI (Copilot API)

**Location:** `src/platform/endpoint/node/chatEndpoint.ts`

```typescript
export function getMaxPromptTokens(
	configService: IConfigurationService,
	expService: IExperimentationService,
	chatModelInfo: IChatModelInformation
): number {
	// Check debug override (internal users only)
	const chatMaxTokenNumOverride = configService.getConfig(ConfigKey.Internal.DebugOverrideChatMaxTokenNum);

	// Base: -3 tokens for each OpenAI completion
	let modelLimit = -3;

	// Priority 1: Debug override
	if (chatMaxTokenNumOverride > 0) {
		modelLimit += chatMaxTokenNumOverride;
		return modelLimit;
	}

	// Priority 2: Experimental overrides
	let experimentalOverrides: Record<string, number> = {};
	try {
		const expValue = expService.getTreatmentVariable<string>('copilotchat.contextWindows');
		experimentalOverrides = JSON.parse(expValue ?? '{}');
	} catch {
		// If experiment service fails, ignore overrides
	}

	if (experimentalOverrides[chatModelInfo.id]) {
		modelLimit += experimentalOverrides[chatModelInfo.id];
		return modelLimit;
	}

	// Priority 3: CAPI max_prompt_tokens
	if (chatModelInfo.capabilities?.limits?.max_prompt_tokens) {
		modelLimit += chatModelInfo.capabilities.limits.max_prompt_tokens;
		return modelLimit;
	}

	// Priority 4: CAPI max_context_window_tokens (fallback)
	if (chatModelInfo.capabilities.limits?.max_context_window_tokens) {
		modelLimit += chatModelInfo.capabilities.limits.max_context_window_tokens;
		return modelLimit;
	}

	return modelLimit;
}
```

### Model Metadata Structure

```typescript
interface IChatModelInformation {
	id: string;  // e.g., 'gpt-4', 'claude-3-opus'
	name: string;
	version: string;
	capabilities: {
		family: string;  // 'gpt-4', 'claude', 'gemini', etc.
		tokenizer: TokenizerType;  // CL100K, O200K
		limits: {
			max_prompt_tokens?: number;  // Preferred
			max_output_tokens?: number;
			max_context_window_tokens?: number;  // Fallback
		};
		supports: {
			tool_calls: boolean;
			vision: boolean;
			streaming: boolean;
			prediction: boolean;
		};
	};
	// ... other metadata
}
```

## Common Models and Their Limits

### OpenAI Models

| Model | Context Window | Typical Prompt Limit | Tokenizer |
|-------|---------------|---------------------|-----------|
| GPT-3.5-turbo | 16k tokens | ~12k tokens | CL100K |
| GPT-4 | 128k tokens | ~120k tokens | CL100K |
| GPT-4o | 128k tokens | ~120k tokens | O200K |
| GPT-4o-mini | 128k tokens | ~120k tokens | O200K |
| o1 | 200k tokens | ~190k tokens | O200K |
| o1-mini | 128k tokens | ~120k tokens | O200K |

### Anthropic Models

| Model | Context Window | Typical Prompt Limit | Tokenizer |
|-------|---------------|---------------------|-----------|
| Claude 3 Opus | 200k tokens | ~190k tokens | O200K |
| Claude 3 Sonnet | 200k tokens | ~190k tokens | O200K |
| Claude 3 Haiku | 200k tokens | ~190k tokens | O200K |
| Claude 3.5 Sonnet | 200k tokens | ~190k tokens | O200K |

### Google Models

| Model | Context Window | Typical Prompt Limit | Tokenizer |
|-------|---------------|---------------------|-----------|
| Gemini 1.5 Pro | 1M tokens | ~950k tokens | O200K |
| Gemini 1.5 Flash | 1M tokens | ~950k tokens | O200K |

## Reserved Tokens

Not all tokens in the context window are available for the prompt:

```typescript
// Base tokens per completion (special tokens)
export const BaseTokensPerCompletion = 3;

// Additional overhead per message
export const BaseTokensPerMessage = 3;

// If message has a name field
export const BaseTokensPerName = 1;
```

**Example calculation:**

```
Model: GPT-4 (128k tokens)
Base completion overhead: -3
Available for prompt: 127,997 tokens

With 10 messages:
Message overhead: 10 * 3 = 30 tokens
Available: 127,967 tokens

Reserve for completion (conservative: 30%): ~38k tokens
Usable for prompt: ~90k tokens
```

## Model-Specific Adaptations

### 1. o1 Family (Reasoning Models)

**Special handling in ChatEndpoint:**

```typescript
if (body?.messages && (this.family.startsWith('o1') || this.model === CHAT_MODEL.O1)) {
	// o1 models don't support system messages
	const newMessages: CAPIChatMessage[] = body.messages.map((message): CAPIChatMessage => {
		if (message.role === OpenAI.ChatRole.System) {
			// Convert system messages to user messages
			return {
				role: OpenAI.ChatRole.User,
				content: message.content,
			};
		}
		return message;
	});
	body['messages'] = newMessages;
}
```

**Why?** o1 models use internal reasoning and don't support system messages.

### 2. Models Without Tool Support

```typescript
interceptBody(body: IEndpointBody | undefined): void {
	if (body && !this.supportsToolCalls) {
		delete body['tools'];  // Remove tools from request
	}
}
```

### 3. Models Without Streaming

```typescript
if (body && !this._supportsStreaming) {
	body.stream = false;  // Force non-streaming mode
}
```

## Token Limit Overrides

### Debug Override (Internal Only)

```typescript
const override = configService.getConfig(ConfigKey.Internal.DebugOverrideChatMaxTokenNum);
if (override > 0) {
	return override - 3;  // Subtract base tokens
}
```

**Use case:** Testing behavior with different limits

### Experimental Override

```typescript
// Experiments can override limits per model
const expValue = expService.getTreatmentVariable<string>('copilotchat.contextWindows');
// expValue might be: '{"gpt-4": 100000, "claude-3-opus": 150000}'
```

**Use case:** A/B testing optimal context sizes

### Cloning with Override

```typescript
public cloneWithTokenOverride(modelMaxPromptTokens: number): IChatEndpoint {
	return this._instantiationService.createInstance(
		ChatEndpoint,
		mixin(deepClone(this._modelMetadata), {
			capabilities: {
				limits: {
					max_prompt_tokens: modelMaxPromptTokens
				}
			}
		})
	);
}
```

**Use case:** Temporarily reduce context for specific requests

## Model Metadata Examples

### GPT-4 Model

```typescript
{
	id: 'gpt-4',
	name: 'GPT-4',
	version: 'turbo-2024-04-09',
	capabilities: {
		family: 'gpt-4',
		tokenizer: TokenizerType.CL100K,
		limits: {
			max_prompt_tokens: 128000,
			max_output_tokens: 4096,
			max_context_window_tokens: 128000
		},
		supports: {
			tool_calls: true,
			vision: false,
			streaming: true,
			prediction: false
		}
	}
}
```

### Claude 3 Opus

```typescript
{
	id: 'claude-3-opus',
	name: 'Claude 3 Opus',
	version: '20240229',
	capabilities: {
		family: 'claude',
		tokenizer: TokenizerType.O200K,
		limits: {
			max_prompt_tokens: 200000,
			max_output_tokens: 4096,
			max_context_window_tokens: 200000
		},
		supports: {
			tool_calls: true,
			vision: true,
			streaming: true,
			prediction: false
		}
	}
}
```

### Gemini 1.5 Pro

```typescript
{
	id: 'gemini-1.5-pro',
	name: 'Gemini 1.5 Pro',
	version: 'latest',
	capabilities: {
		family: 'gemini',
		tokenizer: TokenizerType.O200K,
		limits: {
			max_prompt_tokens: 1000000,
			max_output_tokens: 8192,
			max_context_window_tokens: 1048576
		},
		supports: {
			tool_calls: true,
			vision: true,
			streaming: true,
			prediction: false
		}
	}
}
```

## Handling Context Window Exhaustion

### Detection

```typescript
const usedTokens = await tokenizer.countMessagesTokens(messages);
const availableTokens = endpoint.modelMaxPromptTokens;

if (usedTokens > availableTokens) {
	// Context window exceeded!
}
```

### Strategies

**1. Priority-Based Pruning (automatic via prompt-tsx)**

```typescript
// Lower priority content automatically removed
<UserMessage priority={500}>
	Less important context that can be dropped
</UserMessage>
```

**2. Summarization**

```typescript
if (contextSize > threshold) {
	const summary = await summarizeContext(context, endpoint);
	return summary;
}
```

**3. Chunking**

```typescript
// Split large context into multiple requests
const chunks = splitIntoChunks(context, maxChunkSize);
for (const chunk of chunks) {
	await processChunk(chunk);
}
```

**4. Fallback to Smaller Model**

```typescript
if (usedTokens > endpoint.modelMaxPromptTokens) {
	const smallerEndpoint = await endpointProvider.getFallbackEndpoint();
	return smallerEndpoint;
}
```

## Billing Considerations

### Premium Models

```typescript
interface IChatEndpoint {
	isPremium?: boolean;  // Costs more
	multiplier?: number;  // Cost multiplier
	restrictedToSkus?: string[];  // Who can use it
}
```

**Example:** GPT-4 with 128k context costs more than GPT-3.5 with 16k

### Token Tracking

```typescript
// Track actual usage for billing
const usage = {
	prompt_tokens: actualPromptTokens,
	completion_tokens: actualCompletionTokens,
	total_tokens: actualPromptTokens + actualCompletionTokens
};

// Report to telemetry
telemetryService.sendGHTelemetryEvent('token.usage', {
	model: endpoint.model,
	isPremium: endpoint.isPremium,
	multiplier: endpoint.multiplier
}, {
	promptTokens: usage.prompt_tokens,
	completionTokens: usage.completion_tokens
});
```

## Testing Different Limits

### Mock Endpoint

```typescript
class MockEndpoint implements IChatEndpoint {
	modelMaxPromptTokens: number = 50000;  // Configurable

	// ... other properties
}
```

### Integration Tests

```typescript
test('handles context window limit', async () => {
	const endpoint = createMockEndpoint({ modelMaxPromptTokens: 1000 });
	const largeContext = generateContext(2000);  // Exceeds limit

	const result = await buildPrompt(endpoint, largeContext);

	// Should prune to fit
	expect(result.tokenCount).toBeLessThanOrEqual(1000);
});
```

## Future Considerations

### Adaptive Context Windows

Some future models may have dynamic context windows based on:
- User tier (free vs. paid)
- Current load
- Request complexity
- Time of day

The extension's architecture supports this via the experimentation framework.

### Cost Optimization

Automatically selecting optimal model/context based on:
- Task complexity
- Available budget
- User preferences
- Historical success rates

## Related Files

- Endpoint implementation: `src/platform/endpoint/node/chatEndpoint.ts`
- Endpoint provider: `src/platform/endpoint/common/endpointProvider.ts`
- Model metadata: Fetched from CAPI at runtime
- Token overrides: `src/platform/endpoint/node/proxyExperimentEndpoint.ts`
