# Features

## Wailtpoints

"Waitpoint tokens pause execution until you complete the token or it times out. Perfect for approval workflows, human validation, or AI agents that need human oversight" - https://trigger.dev/launchweek/2/trigger-v4-ga#bun-and-node-22-runtimes

Interesting conceptions, they doesn't differ between waits for time pass and waits for action takes place.

"You can manually complete or timeout waitpoints directly from the dashboard during testing" - they allow to manually force waiting of waitpoints from their ui for debugging purpose.

"When you create a Waitpoint token, we return a callback URL that you can use to complete the waitpoint. This is really useful when you want to hand some work off to a third party service" - they give ability to fulfill waitpoint by rest endpoint

Example of usage:

```ts
const token = await wait.createToken({
  timeout: "10m",
});

await replicate.predictions.create({
  version:
    "27b93a2413e7f36cd83da926f3656280b2931564ff050bf9575f1fdf9bcd7478",
  input: {
    prompt: "A painting of a cat by Andy Warhol",
  },
  // pass the token url to Replicate's webhook, so they can "callback"
  //       👇
  webhook: token.url,
  webhook_events_filter: ["completed"],
});

const result = await wait.forToken<Prediction>(token).unwrap();
```

One more addition: "When Replicate has finished processing the image it will make an HTTP POST request to the URL we gave it. That will complete the Waitpoint and the JSON body is set as the output of the Waitpoint. The run is continued with the output of the Waitpoint." - so we could take data passed by webhook to waitpoint and work with it later.


```ts
// This runs at most once every 5 minutes per user
const result = await childTask.triggerAndWait(
  { userId: user.id },
  {
    idempotencyKey: user.id,
    idempotencyKeyTTL: "5m",
  }
);
```

"All trigger and wait operations now support idempotency keys, letting you cache results and avoid duplicate work" - They implement idempotency keys on request to run tasks. Its expected. But also the implement idempotency on waits:

```ts
// Skip waits on retry using run ID as the key
await wait.for({
  seconds: 30,
  idempotencyKey: ctx.run.id,
  idempotencyKeyTTL: "1h",
});
```

I can't understand how it works and when to use it.

## Prioritising tasks in queue

```ts
// This run will start before runs triggered 10 seconds ago (with no priority)
await myTask.trigger(payload, { priority: 10 });
```

"Priority being relative to the queued time means you can avoid the "starvation" problem. Starvation is when lower priority runs will never be executed if you have a steady stream of medium/high priority runs. This is a significant problem in production use cases for many queue systems, that effectively makes priority useless.

If you want to achieve absolute priority you can do that by using a large number for high priority runs – there's the flexibility to do both." - Quite interesting approach to priorityzing tasks in queue with valid arguments.

```ts
// Pause/resume queues programmatically
await queues.pause(ctx.queue.id);
await queues.resume({ type: "task", name: "my-task-id" });

// Get queue stats
const queue = await queues.retrieve(ctx.queue.id);
console.log(queue.stats);
```

"You can now pause entire environments or individual queues for emergency stops, and get detailed queue statistics" - Quity handy, but I didn't get how do rey resume, do they resume whole queu or handling some particular tasks? And how one defines how many queue he has? Do they do queue per task type? - Usually every task type has its own queue, but you could manage that, you could cereate queues and assign tasks to them explicitly.

## Shemas

```ts
const myToolTask = schemaTask({
  id: "my-tool-task",
  schema: z.object({
    input: z.string().describe("The input to the tool"),
  }),
  run: async ({ input }) => {
    // Your tool logic
    return { result: processInput(input) };
  },
});
```

They use zod for schemas. Approach looks promising

## Vercel's sdk compatability

Its interesting, how Trigger's tasks compatible with Vercel's AI SDK:

```ts
import { ai } from "@trigger.dev/sdk/ai";
import { generateText } from "ai";

const myToolTask = schemaTask({
  id: "my-tool-task",
  schema: z.object({
    input: z.string().describe("The input to the tool"),
  }),
  run: async ({ input }) => {
    // Your tool logic
    return { result: processInput(input) };
  },
});

// Converts a task into an AI tools for the AI SDK
const myTool = ai.tool(myToolTask);

// AI SDK function, passing in the tool we created above
const { text } = await generateText({
  prompt,
  model: openai("gpt-4o"),
  tools: { myTool },
});
```

## Exporting logs and metrics

It's considered bad practice if application itself manages its logs and metrics delivery. But still its usefull for library developer to provide a way to explicitly export logs and metrics. Trigger does that:

```ts
import { defineConfig } from "@trigger.dev/sdk";
import { OTLPLogExporter } from "@opentelemetry/exporter-logs-otlp-http";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";

export default defineConfig({
  project: process.env.TRIGGER_PROJECT_REF,
  dirs: ["./src/trigger"],
  telemetry: {
    logExporters: [
      new OTLPLogExporter({
        url: "https://api.axiom.co/v1/logs",
        headers: {
          Authorization: `Bearer ${process.env.AXIOM_TOKEN}`,
          "X-Axiom-Dataset": "test",
        },
      }),
    ],
    exporters: [
      new OTLPTraceExporter({
        url: "https://api.axiom.co/v1/traces",
        headers: {
          Authorization: `Bearer ${process.env.AXIOM_TOKEN}`,
          "X-Axiom-Dataset": "test",
        },
      }),
    ],
  },
  maxDuration: 3600,
});
```

## Checkpoints

It looks like they use https://criu.org/Main_Page for implementing checkpointing
