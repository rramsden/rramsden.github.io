---
layout: post
title: "Loop Engineering Series - Part II"
date: 2026-07-07 09:00:00 +0900
tags: [programming, tutorial, ai, agents, typescript, openai]
unlisted: true
sitemap: false
---

In Part I we built a weather assistant with a single tool call. It worked, but it had no memory beyond the current conversation — close the terminal and everything is gone.

That's fine for a weather lookup. It's not fine for an assistant that's supposed to know who you are, where you live, who your contacts are, and what's waiting on your reply.

Today we give our agent **long-term memory**: facts that persist across restarts, stored in a file the agent reads and writes itself — by calling memory tools, the same way it calls `get_weather`.

## Two kinds of memory

Before we write code, it helps to be precise about what "memory" means for an agent:

- **Working memory** — the `messages` array from Part I. It's the conversation context window. It dies when the process exits. We already have this.
- **Long-term memory** — persists beyond the run. This is the new concept. It splits further into *semantic* (key facts: "user lives in Tokyo") and *episodic* (a journal of past interactions). Today we're building semantic memory. Episodic and retrieval-based memory come later in the series.

The key design decision: **the agent manages its own memory by calling tools.** We don't hard-code any extraction logic. The model decides what's worth remembering and what to recall, just like it decides when to fetch the weather. Memory is just another tool in the loop.

## Dependencies

We're building on the Part I scaffold. If you still have your `tutorial-part-1` project, you can reuse it. The only addition is Node's built-in `fs/promises` module — no new install needed.

We'll split our tools into a separate file this time, since the toolset is growing. Your project should look like:

```
tutorial-part-1/
├── main.ts
├── tools.ts
├── memory.json   ← created at runtime
└── package.json
```

## The memory store

First, the persistence layer. We'll use a simple JSON file with three categories — `preferences`, `contacts`, and `open_threads`. The first two are obvious; `open_threads` is a taste of where this series is heading (it will become the spine of a Slack triage agent in a later part).

Add this to `tools.ts`:

```typescript
// tools.ts
import { readFile, writeFile, access } from "node:fs/promises";

const MEMORY_FILE = "memory.json";

async function loadMemory(): Promise<Record<string, Record<string, string>>> {
  try {
    await access(MEMORY_FILE);
    const raw = await readFile(MEMORY_FILE, "utf-8");
    return JSON.parse(raw);
  } catch {
    // First run — return a fresh schema
    return { preferences: {}, contacts: {}, open_threads: {} };
  }
}

async function persistMemory(mem: Record<string, Record<string, string>>) {
  await writeFile(MEMORY_FILE, JSON.stringify(mem, null, 2), "utf-8");
}
```

The `try/catch` around `access` handles the first-run case gracefully — no file yet, so we hand back an empty schema. This is the pattern you'll see a lot: the agent's first run is always a special case, and file-based stores handle it by treating "missing" as "empty."

## The memory tools

Now the two tools the agent will call. `save_memory` writes a fact; `recall_memory` reads one (or a whole category if no key is given):

```typescript
export async function save_memory(
  category: string,
  key: string,
  value: string
): Promise<string> {
  const mem = await loadMemory();
  if (!mem[category]) mem[category] = {};
  mem[category][key] = value;
  await persistMemory(mem);
  return `Saved ${category}.${key} = ${value}`;
}

export async function recall_memory(
  category: string,
  key?: string
): Promise<string> {
  const mem = await loadMemory();
  if (!mem[category]) return `No memory in category "${category}".`;
  if (key) {
    const v = mem[category][key];
    return v ? `${category}.${key} = ${v}` : `No memory for ${category}.${key}`;
  }
  return JSON.stringify(mem[category], null, 2);
}
```

Note the return type: these tools return **strings**, not objects. The model reads tool results as text, so we serialize everything to a string before handing it back. This matters — if you return a raw object, the OpenAI client will stringify it, and you get `[object Object]` in the model's context. Always return strings from tools.

Now the tool metadata. We're keeping `get_weather` from Part I and adding the two memory tools. The descriptions are doing real work here — they tell the model *when* to use each tool and *what categories exist*:

```typescript
export const metadata = [
  {
    type: "function",
    function: {
      name: "get_weather",
      description:
        "Fetch current weather for a location. You may pass just a city name (e.g. 'Tokyo'), or add a country/region qualifier after a comma to disambiguate (e.g. 'San Francisco, US' or 'Portland, Oregon').",
      parameters: {
        type: "object",
        required: ["location"],
        properties: {
          location: {
            type: "string",
            description:
              'City name, optionally with a country/region after a comma, e.g. "Tokyo", "San Francisco, US", or "Portland, Oregon".',
          },
        },
      },
    },
  },
  {
    type: "function",
    function: {
      name: "save_memory",
      description:
        "Persist a fact about the user for later recall. Use category 'preferences' for things like location and units, 'contacts' for people, and 'open_threads' for things waiting on the user's reply.",
      parameters: {
        type: "object",
        required: ["category", "key", "value"],
        properties: {
          category: {
            type: "string",
            description: "One of: preferences, contacts, open_threads",
          },
          key: { type: "string", description: "A short identifier for the fact" },
          value: { type: "string", description: "The fact to remember" },
        },
      },
    },
  },
  {
    type: "function",
    function: {
      name: "recall_memory",
      description:
        "Recall a previously saved fact. Pass a category and optional key. If key is omitted, returns everything in that category.",
      parameters: {
        type: "object",
        required: ["category"],
        properties: {
          category: { type: "string", description: "One of: preferences, contacts, open_threads" },
          key: {
            type: "string",
            description: "Optional specific key; omit to recall the whole category",
          },
        },
      },
    },
  },
];
```

## Two ways to load memory

There are two patterns for getting memory into the agent's context, and we'll use both:

**Startup injection** — read `memory.json` once when the program boots and append it to the system prompt. The agent knows your preferences *before the first turn*. This is cheap (one file read) and good for small, always-relevant facts like location and units.

**In-turn recall** — the agent calls `recall_memory` during a conversation when it needs something specific. This is better for large or selectively-relevant memory (you don't want to dump your entire contact list into the system prompt every run).

For Part II we'll use startup injection for preferences (small, always useful) and leave in-turn recall for the agent to discover on its own. Here's the startup loader:

```typescript
// main.ts
import OpenAI from "openai";
import { createInterface } from "node:readline/promises";
import { stdin as input, stdout as output } from "node:process";
import { readFile, access } from "node:fs/promises";
import { metadata, get_weather, save_memory, recall_memory } from "./tools";

const SYSTEM_PROMPT = `You are a personal weather assistant with memory.

You can remember facts about the user by calling save_memory, and recall them by calling recall_memory.
Memory categories: "preferences", "contacts", "open_threads".

If the user tells you a durable fact about themselves (where they live, their temperature unit preference, who they know), save it.
If the user asks for the weather without naming a city, recall their location from memory first.

When you report weather for a location, use exactly this format and nothing else:

Here is the weather information for <city>, <country>:

🌡️ <temp>°C
💨 wind <speed> km/h
☁️ cloud coverage <cloud coverage>%

If the lookup failed, reply with exactly one line:
⚠️ <one concise sentence explaining why>

Do not add greetings, extra commentary, or markdown code fences.`;

const MODEL = "qwen/qwen3-4b";

const client = new OpenAI({
  baseURL: "http://localhost:1234/v1",
  apiKey: "dummy_value",
});

async function loadMemoryContext(): Promise<string> {
  try {
    await access("memory.json");
    const raw = await readFile("memory.json", "utf-8");
    const mem = JSON.parse(raw);
    const prefs = mem.preferences || {};
    if (Object.keys(prefs).length === 0) return "";
    return `\n\nKnown user preferences: ${JSON.stringify(prefs)}`;
  } catch {
    return "";
  }
}
```

We inject only `preferences` at startup — small and always relevant. Contacts and open_threads stay on disk for the agent to recall on demand.

## Fixing the loop

In Part I our tool handling was one-shot: call the model once, execute any tools, call the model once more, done. That breaks the moment the agent needs to *chain* calls — for example, recall a memory and *then* fetch weather based on what it remembered.

We need a real inner loop: keep calling the model and executing tools as long as it keeps requesting them. Here's a single dispatcher for all our tools, then the loop:

```typescript
async function executeTool(name: string, args: any): Promise<string> {
  switch (name) {
    case "get_weather":
      return await get_weather(args.location);
    case "save_memory":
      return await save_memory(args.category, args.key, args.value);
    case "recall_memory":
      return await recall_memory(args.category, args.key);
    default:
      return `Unknown tool: ${name}`;
  }
}
```

Two things every tool call shares: `await` (these are all async) and string returns (enforced by the function signatures). Centralizing dispatch means we can't accidentally forget an `await` or mistype a tool name in the loop body.

Now the main loop:

```typescript
async function main() {
  const rl = createInterface({ input, output });
  rl.on("SIGINT", async () => {
    rl.close();
    process.exit(0);
  });

  const memoryContext = await loadMemoryContext();
  const messages = [{ role: "system", content: SYSTEM_PROMPT + memoryContext }];

  while (true) {
    let user_input = await rl.question("You> ");
    user_input = user_input.trim();
    if (user_input === "quit") break;

    messages.push({ role: "user", content: user_input });

    // First completion
    let reply = (
      await client.chat.completions.create({
        model: MODEL,
        messages,
        tools: metadata,
      })
    ).choices[0].message;
    messages.push(reply);

    // Inner loop: execute tools and re-prompt until the model stops calling tools
    while (reply.tool_calls && reply.tool_calls.length > 0) {
      for (const toolCall of reply.tool_calls) {
        const args = JSON.parse(toolCall.function.arguments || "{}");
        const result = await executeTool(toolCall.function.name, args);
        messages.push({
          role: "tool",
          tool_call_id: toolCall.id,
          content: result,
        });
      }
      reply = (
        await client.chat.completions.create({
          model: MODEL,
          messages,
          tools: metadata,
        })
      ).choices[0].message;
      messages.push(reply);
    }

    console.log(`Assistant> ${reply.content}\n`);
  }
}

main();
```

Two details matter here, and both bite silently if you get them wrong:

1. **Push the assistant's `tool_calls` message before the tool results.** See `messages.push(reply)` right after the first completion. The API requires every `role: "tool"` message to follow the assistant message that requested the call. Skip this and you get a 400 error.
2. **`await` every tool call.** Our tools are async. Without `await`, `result` is a `Promise`, which serializes to `"[object Promise]"` in the model's context. The model will happily hallucinate a response based on that garbage. `tsx` doesn't type-check, so nothing catches this at runtime — it just degrades quietly.

The inner `while` loop is the heart of it. The model calls tools, we execute them, we hand the results back, and the model decides whether to call more tools or produce a final answer. This is what lets the agent *reason through steps* — recall memory, then fetch weather, then format — all in one turn.

## The demo

Start the agent fresh (no `memory.json` yet) and tell it about yourself:

```
You> I live in Tokyo, and I prefer Celsius.
Assistant>

Got it — I've saved that you live in Tokyo and prefer Celsius. 🌡️
```

Check `memory.json` — the agent wrote it:

```json
{
  "preferences": {
    "location": "Tokyo",
    "units": "Celsius"
  },
  "contacts": {},
  "open_threads": {}
}
```

Now ask for the weather without naming a city:

```
You> What's the weather?
Assistant>

Here is the weather information for Shikinejima, Japan:

🌡️ 26°C
💨 wind 36 km/h
☁️ cloud coverage 25%
```

The agent recalled "Tokyo" from memory, then called `get_weather` — a chained tool call that Part I's one-shot loop couldn't have done.

Now the payoff. Quit the program (`quit`), close the terminal, come back tomorrow. Start it again:

```
You> What's the weather?
Assistant>

Here is the weather information for Shikinejima, Japan:

🌡️ 24°C
💨 wind 14 km/h
☁️ cloud coverage 10%
```

It still knows where you live. The memory survived the restart because it lives in a file, not in the `messages` array.

## Memory that grows

Let's show the schema doing real work. Tell the agent about a contact and something waiting on you:

```
You> Sarah Chen is my engineering manager, she owns the Q3 roadmap.
Assistant>

Saved Sarah Chen to contacts.
```

```
You> I'm waiting on Sarah about the Q3 staffing decision — she needs my answer by Friday.
Assistant>

Got it — I've saved that as an open thread: Sarah Chen, Q3 staffing decision, due Friday.
```

```json
{
  "preferences": { "location": "Tokyo", "units": "Celsius" },
  "contacts": {
    "sarah_chen": "Engineering manager, owns the Q3 roadmap"
  },
  "open_threads": {
    "sarah_q3_staffing": "Waiting on Sarah Chen about Q3 staffing decision — answer needed by Friday"
  }
}
```

That `open_threads` entry is a seed. In a later part of this series, when we connect this agent to Slack, `open_threads` becomes the backbone of a triage assistant: the agent will read your unread messages, match them against open threads, and tell you what actually needs your attention today. But that's later — for now it's just the agent remembering what you told it to remember.

## What we built

We now have an agent that:

- Persists facts about you across restarts, in a file it reads and writes itself
- Injects known preferences at startup so it has context before the first turn
- Recalls memory mid-conversation when it needs something specific
- Chains tool calls (recall → fetch → format) thanks to the inner loop

## What we're not doing yet

- **Retrieval over a large corpus.** Our memory is a flat JSON file. Once you have hundreds of contacts and thousands of past messages, you can't dump it all into the system prompt. You need semantic search — embeddings and a vector store. That's a later part.
- **Episodic memory.** We store facts, not a history of what happened. "What did Sarah say about the roadmap last March?" needs episodic memory backed by retrieval. Also later.
- **Concurrency.** The read-modify-write on `memory.json` is not safe if two agents hit it at once. Fine for a single-user CLI assistant; a problem later.

Next time we'll connect this agent to a real external surface — Slack — and turn `open_threads` from a static list into something the agent keeps updated by reading your messages.

