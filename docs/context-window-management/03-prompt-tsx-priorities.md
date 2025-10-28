# Prompt TSX Priority System

## Overview

The extension uses `@vscode/prompt-tsx`, a React-like TSX framework for building prompts. The key innovation is a **priority-based rendering system** that automatically prunes low-priority content when the token budget is exceeded.

## Core Concept

From `CONTRIBUTING.md`:
> Each node in the tree has a `priority` that is conceptually similar to a `zIndex` (higher number == higher priority). If an intent declares more messages than can fit into the token budget, the prompt renderer prunes messages with the lowest priority from the `ChatMessage` array that is eventually sent to the Copilot API, preserving the order in which they were declared.

## Priority Scale

Priorities range from 0 to 1000+, with typical values:

| Priority Range | Usage | Examples |
|----------------|-------|----------|
| 1000-1100 | Critical system instructions | Safety rules, required context |
| 900-999 | High-priority user content | Current query, tool results |
| 700-899 | Important context | Selected code, current file |
| 400-699 | Supporting context | Conversation history, related files |
| 1-399 | Nice-to-have context | Extra workspace info |
| 0 | Lowest priority | Optional hints |

## Prompt Element Structure

### Basic PromptElement

```typescript
import { BasePromptElementProps, PromptElement, PromptSizing } from '@vscode/prompt-tsx';

interface MyPromptProps extends BasePromptElementProps {
	query: string;
}

class MyPrompt extends PromptElement<MyPromptProps> {
	override render(state: void, sizing: PromptSizing) {
		return (
			<>
				<SystemMessage priority={1000}>
					Critical instructions that must always be included.
				</SystemMessage>
				<UserMessage priority={900}>
					{this.props.query}
				</UserMessage>
				<UserMessage priority={500}>
					Additional context that can be pruned if needed.
				</UserMessage>
			</>
		);
	}
}
```

### PromptSizing Object

The `sizing` parameter provides budget information:

```typescript
interface PromptSizing {
	// Token budget available for this prompt
	tokenBudget: number;

	// The endpoint being used (includes modelMaxPromptTokens)
	endpoint: IChatEndpoint;

	// Method to count tokens
	countTokens(text: string): Promise<number>;
}
```

## Priority Examples from Codebase

### Terminal Quick Fix (High Priority)
```typescript
// src/extension/prompts/node/panel/terminalQuickFix.tsx
<SystemMessage priority={1000}>
	You are an expert at explaining terminal output and fixing errors.
</SystemMessage>

<UserMessage priority={1100}>
	{query}
</UserMessage>

<UserMessage priority={700}>
	Additional context from workspace...
</UserMessage>
```

### Agent Prompt (Mixed Priorities)
```typescript
// src/extension/prompts/node/agent/agentPrompt.tsx
<TokenLimit max={sizing.tokenBudget / 6} flexGrow={3} priority={898}>
	<PlanContent plan={plan} />
</TokenLimit>
```

### Conversation History (Lower Priority)
```typescript
// src/extension/prompts/node/panel/conversationHistory.tsx
<TokenLimit max={32768}>
	<PrioritizedList priority={this.props.priority} descending={false}>
		{history}
	</PrioritizedList>
</TokenLimit>
```

## Advanced Priority Features

### 1. FlexGrow

The `flexGrow` property allows elements to expand and consume unused budget:

```typescript
<UserMessage priority={900} flexGrow={2}>
	This can grow to use 2x share of available budget
</UserMessage>

<UserMessage priority={800} flexGrow={1}>
	This can grow to use 1x share of available budget
</UserMessage>
```

**How it works:**
- If budget remains after rendering all required content
- Elements with `flexGrow` proportionally consume the surplus
- Still respects priority (higher priority grows first)

### 2. TokenLimit Component

Sets a hard cap on tokens for a subtree:

```typescript
<TokenLimit max={28_000}>
	<WorkspaceContext {...props} />
</TokenLimit>
```

**Use cases:**
- Prevent any component from dominating the budget
- Reserve budget for other components
- Implement hard limits on variable-size content

### 3. PrioritizedList

Manages a list of items with priorities:

```typescript
<PrioritizedList priority={700} descending={false}>
	{conversationHistory.map((msg, i) => (
		<HistoryMessage key={i} message={msg} />
	))}
</PrioritizedList>
```

**Features:**
- `descending={false}`: Keep oldest messages (prune newest)
- `descending={true}`: Keep newest messages (prune oldest)
- Each child can have its own sub-priority

## Rendering Process

### Phase 1: Build Tree
```typescript
const renderer = PromptRenderer.create(
	instantiationService,
	endpoint,
	MyPrompt,
	{ query: 'user question' }
);
```

### Phase 2: Prepare State (Optional)
```typescript
class MyPrompt extends PromptElement<Props, State> {
	override async prepare(sizing: PromptSizing): Promise<State> {
		// Async operations: API calls, file reads, etc.
		const fileContent = await readFile(uri);
		const tokens = await sizing.countTokens(fileContent);

		return { fileContent, tokens };
	}

	override render(state: State, sizing: PromptSizing) {
		// Use prepared state
		if (state.tokens > sizing.tokenBudget) {
			return <p>File too large</p>;
		}
		return <p>{state.fileContent}</p>;
	}
}
```

### Phase 3: Render with Budget
```typescript
const result = await renderer.render(progress, token);
// result.messages: Raw.ChatMessage[]
// result.tokenCount: number
// result.references: PromptReference[]
```

### Phase 4: Prune Low Priority

**Algorithm (simplified):**
1. Calculate total token cost of all messages
2. If cost ≤ budget, done
3. Otherwise, identify lowest priority message
4. Remove it
5. Repeat until cost ≤ budget

**Important constraint:**
> For now, if two prompt messages _with the same priority_ are up for eviction due to exceeding the token budget, it is not possible for a subtree of the prompt message declared before to evict a subtree of the prompt message declared later.

## Line-Based Priority

For code blocks, can assign descending priority per line:

```typescript
<CodeBlock
	uri={file.uri}
	code={content}
	lineBasedPriority={true}
	priority={800}
>
```

**Effect:**
- Line 1: priority 800
- Line 2: priority 799
- Line 3: priority 798
- etc.

**Benefit:** Can trim from bottom if budget is tight

## Real-World Example: Test Generation

From `src/extension/testing/node/setupTestsFileManager.tsx`:

```typescript
override async render(state: void, sizing: PromptSizing) {
	const testFrameworkInfo = await this.getTestFrameworkInfo();
	const workspaceContext = await this.getWorkspaceContext();

	return (
		<>
			{/* Highest priority: System instructions */}
			<SystemMessage priority={1000}>
				You are an expert at setting up test frameworks.
			</SystemMessage>

			{/* High priority: User's request */}
			<UserMessage priority={900}>
				{this.props.query}
			</UserMessage>

			{/* Medium priority: Test framework details */}
			<UserMessage priority={700} flexGrow={1}>
				<TestFrameworkDetails info={testFrameworkInfo} />
			</UserMessage>

			{/* Lower priority: Workspace context (can be pruned) */}
			<UserMessage priority={500} flexGrow={2}>
				{workspaceContext && (
					<WorkspaceContext
						budget={sizing.tokenBudget * (2/3)}
						context={workspaceContext}
					/>
				)}
			</UserMessage>
		</>
	);
}
```

## Token Budget Distribution

### Strategy 1: Fixed Fractions
```typescript
const historyBudget = sizing.tokenBudget * 0.3;  // 30% for history
const contextBudget = sizing.tokenBudget * 0.5;  // 50% for context
const queryBudget = sizing.tokenBudget * 0.2;    // 20% for query
```

### Strategy 2: Waterfall
```typescript
// Give everyone a minimum, then distribute surplus via flexGrow
<SystemMessage priority={1000}>...</SystemMessage>  {/* Gets minimum */}
<UserMessage priority={900} flexGrow={1}>...</UserMessage>  {/* Gets 1x surplus */}
<UserMessage priority={800} flexGrow={2}>...</UserMessage>  {/* Gets 2x surplus */}
```

### Strategy 3: Conditional Rendering
```typescript
override render(state: State, sizing: PromptSizing) {
	if (state.fileTokens > sizing.tokenBudget * 0.8) {
		// File is too large, show error instead
		return <UserMessage priority={900}>
			File exceeds 80% of token budget.
		</UserMessage>;
	}

	// Normal rendering
	return <UserMessage priority={900}>
		<FileContent content={state.fileContent} />
	</UserMessage>;
}
```

## Best Practices

1. **Use High Priorities for Critical Content**
	- System instructions: 1000+
	- User query: 900+
	- Required context: 800+

2. **Reserve Budget for Completion**
	- Don't use all `modelMaxPromptTokens`
	- Leave room for model's response
	- Typical: use 70-80% of budget

3. **Test Budget Scenarios**
	- Small budget: Does critical content remain?
	- Large budget: Does flexGrow distribute well?
	- Edge cases: Empty context, huge files

4. **Avoid Same Priority Conflicts**
	- Give unique priorities when order matters
	- Use priority ranges (700-799) for related content

5. **Monitor Token Usage**
	- Log actual vs. budgeted tokens
	- Track eviction frequency
	- Adjust priorities based on data

## Related Files

- Prompt renderer: `src/extension/prompts/node/base/promptRenderer.ts`
- Base elements: `@vscode/prompt-tsx` (external package)
- Contributing docs: `CONTRIBUTING.md` (lines 130-186)
