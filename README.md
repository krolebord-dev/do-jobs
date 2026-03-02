# do-jobs

Type-safe delayed and interval jobs for Cloudflare Durable Objects.

## Getting Started

### 1. Install

```bash
pnpm add do-jobs zod
```

`do-jobs` validates job payloads with the [Standard Schema](https://standardschema.dev/) interface, so you can use libraries like Zod.

### 2. Define jobs and wire your Durable Object

```ts
import { createDefineJob, setupJobs, type JobRuntime } from "do-jobs";
import { z } from "zod";

type Env = {
  WEBHOOK_URL: string;
};

const defineJob = createDefineJob<{ env: Env }>();

const sendWebhook = defineJob({ type: "send-webhook" })
  .input(
    z.object({
      message: z.string().min(1),
    }),
  )
  .handler(async ({ input, context, job }) => {
    await fetch(context.env.WEBHOOK_URL, {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({ id: job.id, message: input.message }),
    });
  });

export class JobsDO extends DurableObject<Env> {
  private runtimePromise: Promise<JobRuntime>;

  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env);
    this.runtimePromise = setupJobs({
      ctx,
      context: { env },
      jobs: [sendWebhook],
    });
  }

  async fetch(request: Request): Promise<Response> {
    const runtime = await this.runtimePromise;
    const url = new URL(request.url);

    if (url.pathname === "/enqueue") {
      await runtime.schedule(sendWebhook, {
        input: { message: "hello" },
        at: Date.now() + 5_000,
      });

      return new Response("queued", { status: 202 });
    }

    return new Response("not found", { status: 404 });
  }

  async alarm(): Promise<void> {
    const runtime = await this.runtimePromise;
    await runtime.onAlarm();
  }
}
```

## Docs

### `createDefineJob<TContext>()`

Creates a typed builder for declaring jobs.

```ts
const defineJob = createDefineJob<{ env: Env }>();
```

A job definition requires:
- `type`: unique string identifier.
- `input(schema)`: any Standard Schema compatible validator.
- `handler(({ input, context, job }) => ...)`: execution function.

### `setupJobs({ jobs, ctx, context, maxJobsPerAlarm? })`

Initializes runtime state and returns a `JobRuntime`.

- `jobs`: list of registered job definitions.
- `ctx`: Durable Object state.
- `context`: object injected into every handler.
- `maxJobsPerAlarm` (default `50`): upper bound of jobs executed per alarm tick.

`setupJobs(...)` also:
- creates/migrates internal SQLite tables,
- recalculates the next Durable Object alarm.

### `JobRuntime`

`setupJobs(...)` returns methods:

- `onAlarm()`: run due jobs and schedule the next alarm.
- `setNextAlarm()`: force recalculation of next alarm time.
- `schedule(job, { input, at })`: queue a one-off run.
- `scheduleInterval(job, { input, dedupeKey, everyMs, startAt? })`: create/update an interval schedule.
- `cancelInterval(job, { dedupeKey })`: cancel a previously scheduled interval.

### Behavior Notes

- Inputs are validated before enqueueing and again before handler execution.
- Persisted payloads must remain valid after JSON serialization.
- Interval jobs are deduplicated by `(job.type, dedupeKey)`.
- Failed handlers are marked as `failed` and store error message/stack.
- Job statuses: `queued`, `running`, `completed`, `failed`, `cancelled`.
