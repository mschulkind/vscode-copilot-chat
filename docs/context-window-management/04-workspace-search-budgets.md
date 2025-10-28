# Workspace Search and Token Budgets

## Overview

When Copilot needs workspace context, it must search and retrieve relevant code while respecting token limits. The extension implements multiple search strategies with budget-aware algorithms.

## Search Strategies

### 1. Full Workspace Search

**Location:** `src/platform/workspaceChunkSearch/node/fullWorkspaceChunkSearch.ts`

**Strategy:** Include entire workspace if it fits within budget

**Algorithm:**

```typescript
class FullWorkspaceChunkSearch {
	private static maxFileCount = 100;  // Optimization threshold
	private _previousHitWholeWorkspaceTokenCount = 0;  // Cache for future checks

	async searchWorkspace(sizing: StrategySearchSizing, ...): Promise<StrategySearchResult | undefined> {
		const tokenBudget = sizing.fullWorkspaceTokenBudget ?? sizing.tokenBudget;

		if (!tokenBudget) {
			return undefined;  // No budget, can't search
		}

		const tokenizer = this._tokenizationProvider.acquireTokenizer(sizing.endpoint);
		const chunks: FileChunkAndScore[] = [];
		let usedTokenBudget = 0;

		// Iterate through all files in workspace
		for (const file of this._workspaceIndex.values(options.globPatterns)) {
			const text = await file.getText();
			const fileTokens = await tokenizer.tokenLength(text);

			usedTokenBudget += fileTokens;
			if (usedTokenBudget >= tokenBudget) {
				// Exceeded budget - bail out
				this._logService.debug(
					`FullWorkspaceChunkSearch: Workspace too large. ` +
					`Found at least ${usedTokenBudget} of ${tokenBudget} token limit`
				);
				return undefined;  // Return no results (will use other strategy)
			}

			chunks.push({
				chunk: { file: file.uri, range: fullRange, isFullFile: true, text },
				distance: undefined
			});
		}

		// Success - workspace fits in budget
		return { chunks };
	}
}
```

**Key Points:**

- **All or nothing**: Either returns full workspace or nothing
- **Optimization**: Skip if > 100 files (likely too large)
- **Caching**: Remembers size for future requests
- **Early exit**: Stops counting as soon as budget exceeded

### 2. TF-IDF Chunk Search

**Location:** `src/platform/workspaceChunkSearch/node/tfidfChunkSearch.ts`

**Strategy:** Use TF-IDF ranking to find most relevant code chunks

**Token Management:**

```typescript
// From workspaceChunkSearchService.ts
const MAX_CHUNK_SIZE_TOKENS = 600;  // Per-chunk limit

if (typeof sizing.tokenBudget === 'number') {
	maxResults = Math.floor(sizing.tokenBudget / MAX_CHUNK_SIZE_TOKENS);
}
```

**Flow:**

1. **Query Processing**: Tokenize and weight search terms
2. **Scoring**: Rank chunks by TF-IDF score
3. **Budget Allocation**: `maxResults = tokenBudget / 600`
4. **Return Top N**: Only return chunks that fit budget

### 3. Semantic Search

**Location:** `src/platform/workspaceChunkSearch/node/semanticChunkSearch.ts`

**Strategy:** Use embeddings to find semantically similar code

**Similar budgeting** to TF-IDF but with embedding-based ranking

## StrategySearchSizing Interface

```typescript
interface StrategySearchSizing {
	// Primary token budget for search results
	tokenBudget: number | undefined;

	// Separate budget for full workspace inclusion
	fullWorkspaceTokenBudget: number | undefined;

	// Hint for number of results to return
	maxResultCountHint?: number;

	// Endpoint for tokenization
	endpoint: TokenizationEndpoint;
}
```

## Search Strategy Selection Flow

```
┌─────────────────────────────┐
│ Request for workspace info  │
└──────────────┬──────────────┘
               │
               v
┌─────────────────────────────┐
│ Check if full workspace     │
│ search may be available     │
│ - File count < 100?         │
│ - Previous size < budget?   │
└──────────────┬──────────────┘
               │
       ┌───────┴───────┐
       │ Yes           │ No
       v               v
┌──────────────┐  ┌──────────────────┐
│ Try Full     │  │ Use TF-IDF or    │
│ Workspace    │  │ Semantic Search  │
└──────┬───────┘  └────────┬─────────┘
       │                   │
       v                   v
┌──────────────┐  ┌──────────────────┐
│ Under budget?│  │ Return top N     │
│ Yes: Return  │  │ chunks by score  │
│ No: Try next │  │                  │
└──────────────┘  └──────────────────┘
```

## Configuration

### Enabling Full Workspace Search

```typescript
// From config
ConfigKey.Internal.WorkspaceEnableFullWorkspace

// Checked in code
private isEnabled(): boolean {
	return this._configService.getExperimentBasedConfig<boolean>(
		ConfigKey.Internal.WorkspaceEnableFullWorkspace,
		this._experimentationService
	);
}
```

## Telemetry

Search operations emit detailed telemetry:

```typescript
this._telemetryService.sendMSFTTelemetryEvent('fullWorkspaceChunkSearch.perf.searchFileChunks', {
	status,  // 'success' or 'error'
	workspaceSearchSource: telemetryInfo.callTracker.toString(),
	workspaceSearchCorrelationId: telemetryInfo.correlationId,
	failureReason: errorReason,  // e.g., 'too-large', 'error-reading-file'
}, {
	execTime  // milliseconds
});
```

## Real-World Example: Codebase Tool

**Location:** `src/extension/tools/node/codebaseTool.tsx`

```typescript
class WorkspaceContextWrapper extends PromptElement<WorkspaceContextProps> {
	render() {
		// Main limit is set via maxChunks. Set a TokenLimit just to be sure.
		return <TokenLimit max={28_000}>
			<WorkspaceContext {...this.props} />
		</TokenLimit>;
	}
}
```

**Usage Flow:**

1. User asks a question about their codebase
2. Codebase tool invoked with query
3. Search service called with:
   - `tokenBudget: 28000` (from TokenLimit)
   - `query`: user's question
   - `endpoint`: current model's endpoint
4. Search strategy selected based on workspace size
5. Results returned and included in prompt

## Budget Allocation Strategies

### Conservative (Default)
```
Total budget: 100k tokens
- System messages: 5k (fixed)
- User query: 2k (fixed)
- Workspace context: 28k (TokenLimit)
- Conversation history: 15k
- Reserve for completion: 50k
```

### Aggressive
```
Total budget: 100k tokens
- System messages: 3k (fixed)
- User query: 2k (fixed)
- Workspace context: 60k (flexible)
- Conversation history: 10k
- Reserve for completion: 25k
```

## Best Practices

1. **Early Size Checks**
   - Use file count heuristic before tokenization
   - Cache workspace size for repeated queries

2. **Graceful Degradation**
   - Try full workspace first
   - Fall back to targeted search
   - Always have a fallback strategy

3. **Reasonable Chunk Sizes**
   - 600 tokens per chunk (TF-IDF)
   - Allows ~42 chunks in 25k budget
   - Balance between context and variety

4. **Budget Monitoring**
   - Log when budgets are exceeded
   - Track success rates of strategies
   - Adjust based on telemetry

5. **User Communication**
   - Inform user when workspace is too large
   - Suggest using more specific queries
   - Explain what context was included

## Edge Cases

### Empty Workspace
```typescript
if (workspaceFiles.length === 0) {
	return { chunks: [] };
}
```

### Single Large File
```typescript
if (fileTokens > tokenBudget) {
	// Could truncate or exclude
	this._logService.warn(`File ${uri} exceeds budget: ${fileTokens} tokens`);
	return undefined;
}
```

### Concurrent Searches
```typescript
// Use CancellationTokenSource to cancel in-progress search
const cts = new CancellationTokenSource(token);
try {
	await raceCancellationError(searchOperation, cts.token);
} catch (e) {
	if (!isCancellationError(e)) {
		throw e;
	}
}
```

## Related Files

- Search service: `src/platform/workspaceChunkSearch/node/workspaceChunkSearchService.ts`
- File index: `src/platform/workspaceChunkSearch/node/workspaceFileIndex.ts`
- Chunking: `src/platform/chunking/common/chunk.ts`
