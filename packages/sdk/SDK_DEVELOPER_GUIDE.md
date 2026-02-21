# SDK Developer Guide

This guide focuses on practical secondary development with
`@google/gemini-cli-sdk`, based on the current implementation in this
repository.

## 1. What the SDK is (and is not)

`@google/gemini-cli-sdk` is a thin programmable layer on top of
`@google/gemini-cli-core`.

- It gives you a session-based API (`GeminiCliAgent` + `GeminiCliSession`).
- It supports custom tools (`tool(...)`) and skill loading (`skillDir(...)`).
- It streams low-level agent events from Core (`ServerGeminiStreamEvent`).

It does **not** currently expose first-class SDK APIs for
hooks/subagents/extensions/ACP server wrappers.

## 2. Runtime prerequisites

- Node.js `>=20`
- Install package:

```bash
npm install @google/gemini-cli-sdk
```

Authentication is resolved at session initialization time. Current env-based
auth priority is:

1. `GOOGLE_GENAI_USE_GCA=true` -> Google login flow
2. `GOOGLE_GENAI_USE_VERTEXAI=true` -> Vertex AI flow
3. `GEMINI_API_KEY` present -> Gemini API key flow
4. Otherwise defaults to compute/application default credentials

## 3. Quick start (minimal working call)

```ts
import { GeminiCliAgent } from '@google/gemini-cli-sdk';

async function main() {
  const agent = new GeminiCliAgent({
    instructions: 'You are a concise assistant.',
  });

  const session = agent.session();
  const controller = new AbortController();

  for await (const event of session.sendStream(
    'Summarize this repo.',
    controller.signal,
  )) {
    if (event.type === 'content') {
      process.stdout.write(event.value || '');
    }
  }
}

main().catch(console.error);
```

## 4. Core object model

### 4.1 `GeminiCliAgent`

Constructor options (`GeminiCliAgentOptions`):

- `instructions: string | (ctx) => string | Promise<string>`
- `tools?: Tool[]`
- `skills?: SkillReference[]`
- `model?: string`
- `cwd?: string`
- `debug?: boolean`
- `recordResponses?: string`
- `fakeResponses?: string`

Key methods:

- `session(options?: { sessionId?: string })` -> create a new session
- `resumeSession(sessionId: string)` -> restore a previous persisted session

### 4.2 `GeminiCliSession`

Key APIs:

- `id` -> current session id
- `initialize()` -> optional manual init (auth + core wiring)
- `sendStream(prompt: string, signal?: AbortSignal)` -> async event stream

`sendStream` internally loops through:

1. Send user prompt to model
2. Collect tool call requests from model
3. Execute tools via Core scheduler
4. Feed function responses back to model
5. End when no more tool calls

## 5. Streaming event handling

`sendStream(...)` yields Core stream events (for example):

- `content`
- `thought`
- `tool_call_request`
- `tool_call_response`
- `tool_call_confirmation`
- `finished`
- `error`
- `retry`
- `max_session_turns`
- `loop_detected`
- `agent_execution_stopped`
- `agent_execution_blocked`

Recommended handling skeleton:

```ts
for await (const event of session.sendStream(prompt, signal)) {
  switch (event.type) {
    case 'content':
      process.stdout.write(event.value || '');
      break;
    case 'tool_call_request':
      console.log('\n[tool request]', event.value.name);
      break;
    case 'tool_call_response':
      console.log('\n[tool response]', event.value.callId);
      break;
    case 'error':
      throw new Error(event.value.error.message);
    case 'finished':
      console.log('\n[done]', event.value.reason);
      break;
  }
}
```

## 6. Custom tools

### 6.1 Define tools with schema

```ts
import { GeminiCliAgent, tool, z } from '@google/gemini-cli-sdk';

const addTool = tool(
  {
    name: 'add',
    description: 'Add two numbers',
    inputSchema: z.object({
      a: z.number(),
      b: z.number(),
    }),
  },
  async ({ a, b }) => ({ result: a + b }),
);

const agent = new GeminiCliAgent({
  instructions: 'Use tools when helpful.',
  tools: [addTool],
});
```

### 6.2 Tool context (important for secondary development)

Tool action receives optional `SessionContext`:

- `sessionId`
- `transcript`
- `cwd`
- `timestamp`
- `fs.readFile/writeFile`
- `shell.exec`
- `agent`
- `session`

Example:

```ts
const readConfig = tool(
  {
    name: 'read_config',
    description: 'Read project config file',
    inputSchema: z.object({ path: z.string() }),
  },
  async ({ path }, ctx) => {
    const content = await ctx?.fs.readFile(path);
    return content ?? 'file not found';
  },
);
```

### 6.3 Error semantics in tools

By default, thrown errors are runtime errors and may fail the current loop.

If you want model-visible errors:

- Throw `new ModelVisibleError('...')`, or
- Set `sendErrorsToModel: true` on tool definition

Then the SDK wraps error as tool result text (`Error: ...`) and sends it back to
model.

## 7. Skills integration

Use `skillDir(...)` to load a skill directory or a skill root.

```ts
import { GeminiCliAgent, skillDir } from '@google/gemini-cli-sdk';

const agent = new GeminiCliAgent({
  instructions: 'You are a helpful assistant.',
  skills: [skillDir('./skills/pirate-skill'), skillDir('./skills-root')],
});
```

At runtime, `GeminiCliSession.initialize()` loads skills into Core
`SkillManager` and re-registers `ActivateSkillTool` schema when skills are
available.

## 8. Session persistence and resume

```ts
const agent = new GeminiCliAgent({ instructions: 'Remember details.' });

const session1 = agent.session();
const id = session1.id;
for await (const _ of session1.sendStream('Remember: BANANA')) {
}

const session2 = await agent.resumeSession(id);
for await (const event of session2.sendStream(
  'What word did I ask you to remember?',
)) {
  if (event.type === 'content') process.stdout.write(event.value || '');
}
```

Resume logic reads chat history from project temp chat files under the
configured `cwd`.

## 9. Cancellation and control

Use `AbortController`:

```ts
const controller = new AbortController();
const stream = session.sendStream('long task...', controller.signal);

setTimeout(() => controller.abort(), 5000);
```

## 10. Deterministic tests and replay

`GeminiCliAgentOptions` supports:

- `recordResponses: string` -> record model interactions
- `fakeResponses: string` -> replay from file

This is useful for stable CI integration tests around tool orchestration.

## 11. Practical integration patterns

### Pattern A: Single long-lived session per user workspace

- Keep one `GeminiCliSession`
- Stream events to your UI layer
- Persist `session.id` for resume

### Pattern B: One short session per task

- Create a fresh session for each user action
- Better isolation, simpler failure boundaries

### Pattern C: Tool-first workflow

- Keep instructions concise
- Add strongly typed tools for your domain actions
- Use model-visible tool errors to let model self-correct

## 12. Known current limitations

At current state of this package, first-class SDK APIs are not yet provided for:

- Hooks
- Subagents
- Extensions
- ACP server mode

If you need these, integrate with Core directly or track future SDK expansions.

## 13. Troubleshooting

### No output / early failure on first call

- Check auth env vars and credentials
- Verify `cwd` points to an accessible project directory

### Tool not called

- Improve tool description and schema clarity
- Prompt explicitly: “Use tool X to ...”

### Session resume fails

- Ensure same `cwd` used for creation and resume
- Ensure chat files were persisted in previous run

---

If you are building an SDK wrapper for your own product, start with:

1. a typed event adapter around `sendStream`
2. a domain tool package (all business operations as tools)
3. a session persistence layer storing `session.id` and metadata
