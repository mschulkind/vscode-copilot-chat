# Token Counting

## Overview

Token counting is fundamental to context window management. The extension uses **TikToken** (OpenAI's tokenizer) to accurately count tokens before sending prompts to language models.

## Tokenizer Implementation

### Location
`src/platform/tokenizer/node/tokenizer.ts`

### Supported Tokenizers

1. **CL100K** - Used by GPT-3.5 and GPT-4 family
2. **O200K** - Used by newer models (GPT-4o, o1)

### Token Constants

```typescript
// Base tokens per completion request
export const BaseTokensPerCompletion = 3;

// Each message has overhead (special tokens)
export const BaseTokensPerMessage = 3;

// Each message with a name field costs 1 additional token
export const BaseTokensPerName = 1;
```

## TokenizerProvider Class

### Initialization
```typescript
class TokenizerProvider implements ITokenizerProvider {
	// Lazy-loaded tokenizers
	private readonly _cl100kTokenizer: Lazy<BPETokenizer>;
	private readonly _o200kTokenizer: Lazy<BPETokenizer>;

	constructor(useWorker: boolean, telemetryService: ITelemetryService) {
		// Tokenizer dictionaries loaded from disk
		this._cl100kTokenizer = new Lazy(() =>
			new BPETokenizer(useWorker, './cl100k_base.tiktoken', 'cl100k_base', telemetryService)
		);
		this._o200kTokenizer = new Lazy(() =>
			new BPETokenizer(useWorker, './o200k_base.tiktoken', 'o200k_base', telemetryService)
		);
	}
}
```

## Token Counting Methods

### 1. Text Token Length
```typescript
async tokenLength(text: string): Promise<number> {
	// Uses LRU cache for performance
	let cacheValue = this._cache.get(text);
	if (!cacheValue) {
		cacheValue = (await this.tokenize(text)).length;
		this._cache.put(text, cacheValue);
	}
	return cacheValue;
}
```

### 2. Message Token Counting
```typescript
async countMessageTokens(message: Raw.ChatMessage): Promise<number> {
	// Each message has base overhead
	return this.baseTokensPerMessage +
		(await this.countMessageObjectTokens(toMode(OutputMode.OpenAI, message)));
}
```

### 3. Tool Token Counting
```typescript
async countToolTokens(tools: LanguageModelChatTool[]): Promise<number> {
	const baseToolTokens = 16; // Overhead for tools array
	let numTokens = tools.length ? baseToolTokens : 0;

	const baseTokensPerTool = 8; // Per-tool overhead
	for (const tool of tools) {
		numTokens += baseTokensPerTool;
		numTokens += await this.countObjectTokens({
			name: tool.name,
			description: tool.description,
			parameters: tool.inputSchema
		});
	}

	// Add 10% safety margin
	return Math.floor(numTokens * 1.1);
}
```

### 4. Image Token Counting
```typescript
function calculateImageTokenCost(imageUrl: string, detail: 'low' | 'high' | undefined): number {
	let { width, height } = getImageDimensions(imageUrl);

	if (detail === 'low') {
		return 85; // Fixed cost for low detail
	}

	// Scale to fit 2048x2048 if needed
	if (width > 2048 || height > 2048) {
		const scaleFactor = 2048 / Math.max(width, height);
		width = Math.round(width * scaleFactor);
		height = Math.round(height * scaleFactor);
	}

	// Scale smallest dimension to 768
	const scaleFactor = 768 / Math.min(width, height);
	width = Math.round(width * scaleFactor);
	height = Math.round(height * scaleFactor);

	// Calculate tiles (512x512 each)
	const tiles = Math.ceil(width / 512) * Math.ceil(height / 512);

	return tiles * 170 + 85; // 170 per tile + base cost
}
```

## Performance Optimizations

### 1. LRU Cache
- Caches token counts for strings
- Size: 5000 entries
- Dramatically reduces repeated tokenization

### 2. Worker Thread Option
- Can run tokenization in a Web Worker
- Prevents blocking the main thread
- Auto-terminates after 15s of inactivity

### 3. Telemetry Sampling
- 1 in 1000 tokenizations sends stats
- Tracks: call count, encoding duration, text length

## Integration with Endpoints

Each endpoint specifies its tokenizer type:

```typescript
export interface IChatEndpoint {
	readonly tokenizer: TokenizerType; // CL100K or O200K
	readonly modelMaxPromptTokens: number; // e.g., 128000
	acquireTokenizer(): ITokenizer;
}
```

## Character-to-Token Approximation

When precise tokenization is too expensive, character-based approximations are used:

### Approximation Factor
```typescript
// In many places: tokens ≈ characters / 4
const charLimit = (endpoint.modelMaxPromptTokens * 4) / 3;
```

This is based on the empirical observation that:
- Average token length ≈ 4 characters
- Used in TypeScript context, fix selection, and inline chat

### Language-Specific Approximations
From `src/extension/completions-core/vscode-node/prompt/src/tokenization/tokenizer.ts`:

```typescript
const EFFECTIVE_TOKEN_LENGTH = {
	[TokenizerName.cl100k]: {
		python: 3.99,
		typescript: 4.54,
		typescriptreact: 4.58,
		javascript: 4.76,
		csharp: 5.13,
		java: 4.86,
		cpp: 3.85,
		php: 4.1,
		html: 4.57,
		// ... more languages
	}
};
```

## Token Overhead Examples

### Single Message
```
Base tokens per message: 3
Content: "Hello, world!" → ~3 tokens
Total: ~6 tokens
```

### Message with Name
```
Base tokens per message: 3
Name token: 1
Content tokens: variable
Total: 4 + content
```

### Tool Call
```
Base tool overhead: 16 (if tools present)
Per tool: 8 + tokens for name/description/schema
Safety margin: +10%
```

## Best Practices

1. **Cache Aggressively**: The LRU cache is essential for performance
2. **Batch Counting**: Count tokens for multiple items together when possible
3. **Use Approximations Early**: Character-based estimates for quick checks
4. **Precise Counting Later**: Use actual tokenization when making final decisions
5. **Account for Overhead**: Always add base tokens for messages and tools

## Related Files

- Token counting interface: `src/util/common/tokenizer.ts`
- Worker implementation: `src/platform/tokenizer/node/tikTokenizerWorker.js`
- WASM tokenizer: `src/platform/tokenizer/node/tikTokenizerImpl.ts`
