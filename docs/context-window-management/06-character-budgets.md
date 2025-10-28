# Character Budgets

## Overview

In some scenarios, precise token counting is too expensive or impractical. The extension uses **character-based budgets** as a faster approximation for managing context size, particularly in TypeScript language context gathering.

## Character-to-Token Ratio

The fundamental assumption:

```typescript
// Approximate ratio: 1 token ≈ 4 characters
const charLimit = (endpoint.modelMaxPromptTokens * 4) / 3;
```

This is based on empirical observations across many programming languages.

## CharacterBudget Class

### Location
- TypeScript context: `src/extension/typescriptContext/vscode-node/languageContextService.ts`
- Server plugin: `src/extension/typescriptContext/serverPlugin/src/common/contextProvider.ts`

### Implementation

```typescript
class CharacterBudget {
	public readonly overall: number;
	private mandatory: number;
	private optional: number;
	private start: { mandatory: number; optional: number };

	constructor(mandatory: number, optional: number) {
		this.overall = mandatory;
		this.mandatory = mandatory;
		this.optional = optional;
		this.start = { mandatory, optional };
	}

	spend(chars: number): void {
		this.mandatory -= chars;
		this.optional -= chars;
	}

	isExhausted(): boolean {
		return this.mandatory <= 0;
	}

	isOptionalExhausted(): boolean {
		return this.optional <= 0;
	}

	public fresh(): CharacterBudget {
		return new CharacterBudget(this.start.mandatory, this.start.optional);
	}
}
```

### Two-Tier Budget System

**Mandatory Budget:**
- MUST be included
- Core context that's essential
- Exhaustion means stop processing

**Optional Budget:**
- Nice to have
- Can be skipped if needed
- Allows flexibility in context size

## Usage Modes

```typescript
enum ContextItemUsageMode {
	minimal = 'minimal',    // Only mandatory
	double = 'double',      // Mandatory + up to document size
	fillHalf = 'fillHalf',  // Mandatory + 50%
	fill = 'fill'           // Mandatory + 100%
}
```

### Mode Details

**Minimal Mode:**
```typescript
return new CharacterBudget(chars, 0);
// No optional budget at all
```

**Double Mode:**
```typescript
return new CharacterBudget(chars, Math.min(chars, document.getText().length));
// Optional budget up to document size
```

**Fill Half Mode:**
```typescript
return new CharacterBudget(chars, Math.floor(chars / 2));
// Optional budget = 50% of mandatory
```

**Fill Mode:**
```typescript
return new CharacterBudget(chars, chars);
// Optional budget = 100% of mandatory
```

## Server Plugin Implementation

### Enhanced CharacterBudget

```typescript
export class CharacterBudget {
	private charBudget: number;
	private lowWaterMark: number;
	private itemRejected: boolean;

	constructor(budget: number, lowWaterMark: number = 256) {
		this.charBudget = budget;
		this.lowWaterMark = lowWaterMark;
		this.itemRejected = false;
	}

	public spent(chars: number): void {
		this.charBudget -= chars;
	}

	public hasRoom(chars: number): boolean {
		const result = this.charBudget - this.lowWaterMark >= chars;
		if (!result) {
			this.itemRejected = true;
		}
		return result;
	}

	public isExhausted(): boolean {
		return this.charBudget <= 0;
	}

	public wasItemRejected(): boolean {
		return this.itemRejected;
	}

	public throwIfExhausted(): void {
		if (this.charBudget <= 0) {
			throw new TokenBudgetExhaustedError();
		}
	}

	public spentAndThrowIfExhausted(chars: number): void {
		this.spent(chars);
		this.throwIfExhausted();
	}
}
```

### Low Water Mark

The `lowWaterMark` (default 256 chars) provides a safety buffer:

```typescript
public hasRoom(chars: number): boolean {
	// Don't allow budget to go below lowWaterMark
	const result = this.charBudget - this.lowWaterMark >= chars;
	if (!result) {
		this.itemRejected = true;  // Track that we rejected something
	}
	return result;
}
```

**Purpose:**
- Prevent exact budget exhaustion
- Leave room for overhead
- Better error handling

## Real-World Example: Fix Selection

### Location
`src/extension/context/node/resolvers/fixSelection.ts`

### Code

```typescript
export function generateFixContext(
	endpoint: IChatEndpoint,
	documentContext: IDocumentContext,
	range: Range,
	rangeOfInterest: Range
): { contextInfo: IFixCodeContextInfo; tracker: CodeContextTracker } {

	// Number of tokens the endpoint can handle, 4 chars per token, we consume one 3rd
	const charLimit = (endpoint.modelMaxPromptTokens * 4) / 3;
	const tracker = new CodeContextTracker(charLimit);
	const document = documentContext.document;
	const language = documentContext.language;

	const rangeInfo = new CodeContextRegion(tracker, document, language);
	const aboveInfo = new CodeContextRegion(tracker, document, language);
	const belowInfo = new CodeContextRegion(tracker, document, language);

	// Build context from selection
	const continueExecution = processFixSelection(rangeInfo, range, rangeOfInterest);
	if (!continueExecution) {
		aboveInfo.trim();
		rangeInfo.trim();
		belowInfo.trim();
		return { contextInfo: { language, above: aboveInfo, range: rangeInfo, below: belowInfo }, tracker };
	}

	// Add surrounding context
	expandContext(document, rangeInfo, aboveInfo, belowInfo, tracker);

	return { contextInfo: { language, above: aboveInfo, range: rangeInfo, below: belowInfo }, tracker };
}
```

### Key Points

1. **Conservative Allocation**: Uses 1/3 of available tokens
2. **Three Regions**: Above, selection, below
3. **Shared Tracker**: All regions share same budget
4. **Early Exit**: Stops if budget exhausted

## Real-World Example: Inline Chat Selection

### Location
`src/extension/context/node/resolvers/inlineChatSelection.ts`

### Pattern

```typescript
/**
 * Limits the total char count to 1/3rd of the max tokens size.
 */
export async function getInlineChatSelection(
	endpoint: IChatEndpoint,
	...
): Promise<...> {
	const charLimit = (endpoint.modelMaxPromptTokens * 4) / 3;

	// Use character limit for fast estimation
	if (selectionText.length > charLimit) {
		// Truncate or reject
		return truncateSelection(selectionText, charLimit);
	}

	return selectionText;
}
```

## Advantages of Character Budgets

1. **Speed**: No tokenization required
2. **Simplicity**: Easy to reason about
3. **Real-time**: Works in interactive scenarios
4. **Conservative**: Usually underestimates (safer)

## Disadvantages

1. **Inaccuracy**: Can be off by 20-30%
2. **Language Dependent**: Varies by programming language
3. **No Message Overhead**: Doesn't account for base tokens
4. **Optimization Missed**: May leave budget unused

## When to Use Character Budgets

### Good Use Cases ✅
- Language server context gathering
- Real-time inline chat
- Quick budget checks
- Incremental context building

### Bad Use Cases ❌
- Final prompt composition (use tokenizer)
- API requests (need exact count)
- Billing/quota management
- Detailed telemetry

## Conversion Factors

From actual tokenizer code:

```typescript
const EFFECTIVE_TOKEN_LENGTH = {
	python: 3.99,
	typescript: 4.54,
	javascript: 4.76,
	csharp: 5.13,
	java: 4.86,
	cpp: 3.85,
	go: 3.93,
	// etc.
};
```

**General rule:** 3.5 - 5.5 characters per token

## Budget Calculation Example

**Scenario:** GPT-4 with 128k token limit

```
Token budget: 128,000 tokens

Character budget calculation:
= 128,000 * 4 / 3
= 512,000 / 3
= 170,666 characters

Actual vs. Character budget:
- Token counting: 128,000 tokens (precise)
- Character budget: ~170k chars → ~38k-48k tokens (approximate)
```

## Error Handling

### TokenBudgetExhaustedError

```typescript
export class TokenBudgetExhaustedError extends Error {
	constructor() {
		super('Budget exhausted');
	}
}
```

**Usage:**

```typescript
try {
	budget.spentAndThrowIfExhausted(itemSize);
	// Process item
} catch (e) {
	if (e instanceof TokenBudgetExhaustedError) {
		// Stop gathering more context
		return accumulatedContext;
	}
	throw e;
}
```

## Integration with TypeScript Language Server

The TypeScript language server uses character budgets for:

1. **Symbol Context**: Function signatures, type definitions
2. **Neighbor Files**: Related imports and exports
3. **Documentation**: JSDoc comments and README snippets
4. **Project Structure**: Package.json, tsconfig.json

**Flow:**

```
User types code
     ↓
Language server computes context
     ↓
Character budget allocated (chars = tokens * 4)
     ↓
Gather context up to budget
     ↓
Convert to actual tokens before sending to API
```

## Best Practices

1. **Use Conservative Ratios**
   - 4 chars per token is safe
   - Language-specific can be more accurate

2. **Leave Safety Margin**
   - Use 1/3 or 1/2 of available tokens
   - Account for message overhead

3. **Switch to Tokenizer Later**
   - Use chars for building
   - Use tokenizer for final check

4. **Monitor Actual Usage**
   - Log character estimate vs. actual tokens
   - Adjust ratios based on data

5. **Handle Exhaustion Gracefully**
   - Track rejected items
   - Provide feedback to user

## Related Files

- Language context service: `src/extension/typescriptContext/vscode-node/languageContextService.ts`
- Server plugin: `src/extension/typescriptContext/serverPlugin/src/common/contextProvider.ts`
- Fix selection: `src/extension/context/node/resolvers/fixSelection.ts`
- Inline chat: `src/extension/context/node/resolvers/inlineChatSelection.ts`
