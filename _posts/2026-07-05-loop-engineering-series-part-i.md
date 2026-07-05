---
layout: post
title: "Loop Engineering Series - Part I"
date: 2026-07-05 09:00:00 +0900
tags: [programming, tutorial, ai, agents, typescript, openai]
---

It’s 2026 and it’s all about the agent loop.

This guide follows my work understanding loop engineering. As I want to learn from the fundamentals I am building this tutorial part series under my blog where I will progressively cover more advanced concepts from the ground up.

Today we will build a simple weather assistant which will be a simple agentic loop with a tool call for fetching the weather in a city.

## Dependencies

There are a few dependencies we will need to follow along

### 1. Typescript Project Scaffold

You will need to create a base project. In this case I am creating a very simple project scaffold using three package including openai which will use below

```bash
mkdir tutorial-part-1 && cd tutorial-part-1
npm i -D typescript tsx openai
npx tsc --init
```

After installing the dependencies copy and paste the following into `package.json` 

```json
{
  "type": "module",
  "scripts": {
    "dev": "npx tsx main.ts"
  },
  "devDependencies": {
    "openai": "^6.45.0",
    "tsx": "^4.23.0",
    "typescript": "^6.0.3"
  }
}
```

Then create your `main.ts` file

```bash
console.log("Hello World!")
```

Finally run `npm run dev` to see if the project works

### 2. LLM Studio Desktop

Grab the latest version of LLM Studio ([https://lmstudio.ai/](https://lmstudio.ai/)) and we will build an agent that uses a local model. 

Once you boot up you will want to download a light-weight model (one that supports tool calling). Note, I am using `qwen-3-4b` for the examples below

## Getting Started

An agentic loop contains the following:

- The loop (in this case a while loop)
- The LLM model for generating responses based on an input
- Available tool calls e.g. in our case today `get_weather`

Here is an example of the most simple agentic loop (minus tool calling):

```typescript
import { createInterface } from "node:readline/promises"
import { stdin as input, stdout as output } from "node:process";

const SYSTEM_PROMPT = `You are a weather assistant`;
const MODEL = 'qwen/qwen3-4b';

const messages = [{ role: "system", content: SYSTEM_PROMPT }];

async function main() {
  const rl = createInterface({ input, output });
  
  // Gracefully handle CTRL+C on CLI
  rl.on("SIGINT", async () => {
    rl.close();
    process.exit(0);
  })
  
  // The agentic loop
  while (true) {
    let user_input = await rl.question("You> ");
    user_input = user_input.trim(); // Strip any whitespace
    
    // Check if user has typed "exit"
    if (user_input == "quit") break;
    
    // Push the users question onto our context
    messages.push({ role: "user", content: user_input });
    
    // Mock out an LLM response from the user input
    let mocked_response = { role: "assistant", content: "Hello!" };
    messages.push(mocked_response);
    
    console.log(`Assistant> ${mocked_response.content}\n\n`);
  }
}
```

## Prompting the LLM model

First step is initializing OpenAI wrapper so we can communicate with LLM Studio. You will need to enable the “Developer Server” and in “Server Settings” listen on your local network. It should then give you an IP address to make requests. Copy and paste this into the OpenAI wrapper as follows (note, you may not need a valid apiKey if password protection is disabled):

```typescript
const client = new OpenAI({
  baseURL: "http://100.121.13.119:1234/v1",
  apiKey: 'dummy_value'
});
```

Next let’s replace the mocked call out example above with an OpenAI completion request

```typescript
// Call our model and push the response to messages
const completion = await client.chat.completions.create({
  model: MODEL,
  messages,
});
const reply = completion.choices[0].message;
messages.push(reply);

```

We should now be able to have a conversation with our agent

```
You> Hello
Assistant>

Hello! How can I assist you today? If you need weather information or have any questions about the forecast, feel free to ask! 🌤️ 

You> What's the weather, right now, in Tokyo, Japan.
Assistant>

I currently don’t have access to live weather data or internet resources to provide the exact current weather in Tokyo. However, I can share that Tokyo typically has a humid subtropical climate, so:

- **Summer** (June–August): Hot and rainy, with temperatures often exceeding 30°C (86°F).
- **Winter** (December–February): Mild and mostly sunny, with temperatures averaging around 10–15°C (50–59°F).

For the most accurate and up-to-date information, I recommend checking a reliable weather service like [AccuWeather](https://www.accuweather.com) or [The Weather Channel](https://www.weather.com). Let me know if you’d like help with something else! 🌤️ 
```

## Adding tool calls

In the example above our LLM doesn’t have the ability to call tools so when its asked for real-time data it doesn’t work.

Will expand our program to include a `tools.ts` file which will contain the tool calls

```typescript
// tools.ts
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
          }
        }
      }
    }
  }
];

```

Next in `main.ts` import the tools and pass them to completions

```typescript
import { metadata } from "./tools"

const completion = await client.chat.completions.create({
  model: MODEL,
  messages,
  tools: metadata // <---------
});
```

As an experiment we can now ask our model if it has access to any tools

```
You> What tools do you have access to?
Assistant>

I have access to a weather tool that allows me to fetch current weather information for any location. You can simply provide a city name (e.g., "New York") or specify a location with country/region details (e.g., "London, UK" or "Sydney, Australia"). Would you like me to check the weather for a specific place?
```

Of course if you ask the agent for the weather in a City it will return blank since there is no tool defined yet. Let’s define `get_weather`

```typescript
// tools.ts
export async function get_weather(location: string): Promise<string> {
  try {
    // Native fetch with an 8-second timeout inline
    const res = await fetch(`https://wttr.in/${encodeURIComponent(location)}?format=j1`, {
      signal: AbortSignal.timeout(8000)
    });
    
    if (!res.ok) throw new Error(`HTTP ${res.status} ${res.statusText}`);
    const wx = await res.json();
    
    const current = wx.current_condition?.[0];
    const area = wx.nearest_area?.[0];
    if (!current || !area) return `Weather data unavailable for ${location}.`;

    const name = area.areaName?.[0]?.value || location;
    const country = area.country?.[0]?.value || "";
    const { temp_C: temp, windspeedKmph: wind, cloudcover: clouds } = current;
    const condition = current.weatherDesc?.[0]?.value || "Unknown";

    return `${name}${country ? `, ${country}` : ""}: ${temp}°C, wind ${wind} km/h, cloud coverage: ${clouds}% (${condition})`;
  } catch (err) {
    return `Sorry, I couldn't fetch the weather for "${location}" right now (${err instanceof Error ? err.message : err}). Please try again shortly.`;
  }
}
```

Once defined you will need to handle the reply from the LLM to call tools. You can do this by checking the reply from the first completion and looping through `tool_calls` array in the reply:

```typescript
// Call our model with our user input
const completion = await client.chat.completions.create({
  model: MODEL,
  messages,
  tools: metadata
});
const reply = completion.choices[0].message;

// Check if the model requested any tool calls
if (reply.tool_calls && reply.tool_calls.length > 0) {
  for (const toolCall of reply.tool_calls) {
    if (toolCall.function.name == "get_weather") {
      const args = JSON.parse(toolCall.function.arguments)
      const toolResult = get_weather(args.location);
      
      messages.push({
        role: "tool",
        tool_call_id: toolCall.id,
        content: toolResult
      });
    }
  }
}

// Finally follow up with results from any tools calls or initial output
const followup = await client.chat.completions.create({
  model: MODEL,
  tools: metadata,
  messages,
});
const finalMessage = followup.choices[0].message;
messages.push(finalMessage);
```

You should now be able to see the result inside the reply:

```
You> What's the weather in Tokyo?
Assistant>

The current weather in Tokyo is 26°C, with winds at 36 km/h and light cloud coverage of 25%.
```

## Fixing the format

If we don’t fix the format of our agent it will always come up with some unique response. That’s usually undesirable if were trying to get deterministic results.

We can fix this with a slight system prompt update

```typescript
const SYSTEM_PROMPT = `
You are a weather assistant.

When you report weather for a location, your reply to the user MUST use exactly this format and nothing else:

Here is the weather information for <city>, <country>:

🌡️ <temp>°C
💨 wind <speed> km/h
☁️ cloud coverage <cloud coverage>%

If the lookup failed for any reason, reply with exactly one line:
⚠️ <one concise sentence explaining why>

Do not add greetings, extra commentary, or markdown code fences.
`;
```

Now let’s try again

```
You> What's the weather in Tokyo?
Assistant>

Here is the weather information for Shikinejima, Japan:

🌡️ 26°C
💨 wind 36 km/h
☁️ cloud coverage 25%
```

Perfect. That’s it for the first tutorial. We now have an agent that will tell us the weather to any location.
