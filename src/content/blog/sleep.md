---
title: 'Sleep Is Not Scheduling'
description: 'Timers are easy on one machine. Cross-instance scheduling is a coordination problem, and the broker or database should own it.'
pubDate: 'Mar 21 2026'
heroImage: '../../assets/blog-placeholder-2.jpg'
---

Every team hits this moment eventually: someone asks for a background task that should run “once on Tuesday at 5 PM,” and the first implementation is a timer in application memory.

That works right up until it doesn’t.

As soon as you run more than one instance, `sleep` stops being a scheduling strategy and starts being a gamble. Two nodes may wake up and do the same work. One node may restart and forget the task existed. A deployment might quietly erase the only process that remembered the deadline.

The core mistake is letting an app instance own time-sensitive state. In distributed systems, the schedule should live in a shared system: a database row, a broker message, or a workflow engine. Your app should only claim and execute work.

In Go, the anti-pattern usually looks harmless:

```go
go func() {
    time.Sleep(time.Until(runAt))
    sendReminder(ctx, userID)
}()
```

It is compact. It is readable. It is also tied to one process. If that process dies, the schedule dies with it.

A safer shape is to put the task into shared infrastructure and let workers react when it becomes due. With NATS JetStream, that can be as simple as publishing a scheduled message:

```go
msg := nats.NewMsg("schedule.jobs")
msg.Header.Set("Nats-Schedule", "@at 2026-03-24T17:00:00+01:00")
msg.Header.Set("Nats-Schedule-Target", "jobs.run")
msg.Header.Set("Nats-Msg-Id", "reminder-user-42")
msg.Data = []byte(`{"user_id":42}`)

_, err := js.PublishMsg(ctx, msg)
if err != nil {
    log.Fatal(err)
}
```

Rust pushes you toward the same lesson. `tokio::sleep_until` is excellent for local async coordination, but it still does not become cross-instance just because the syntax is better:

```rust
use tokio::time::{sleep_until, Instant};

let when = Instant::now() + std::time::Duration::from_secs(3600);
sleep_until(when).await;
// fine for one process, not a distributed scheduler
```

And Zig, with its explicit style, makes the tradeoff even clearer: a local timer is still only local state.

```zig
const std = @import("std");

pub fn main() !void {
    std.time.sleep(5 * std.time.ns_per_s);
    std.debug.print("task fired\n", .{});
}
```

There is nothing wrong with these primitives. They are just solving a different problem.

The useful mental model is this: if the task matters after a restart, it should not live only in memory. Once you accept that, the implementation becomes much less magical. Store the intent durably. Let multiple workers compete safely. Make the actual handler idempotent. Everything else is detail.

Distributed scheduling sounds fancy, but most of the time it is really just durable state plus careful ownership.

There is a strong temptation in backend engineering to solve every reliability problem with a new platform. Sometimes that is the right call. Sometimes it is a very expensive way to avoid saying, “our database or broker can already do this.”

For one-off scheduled work, the simplest production-worthy solution is often the infrastructure already sitting in your stack.

If your system already depends on Postgres, a row with `run_at`, `status`, and a safe claim query can take you surprisingly far. If you already run Redis, a sorted set and an atomic claim script can be enough. If NATS is your event backbone, JetStream scheduling may fit naturally into the flow you already understand.

That matters because operational familiarity is part of architecture. A “less elegant” design that your team can debug at 2 AM is often the better system.

Here is the Go shape I like when a worker should be safe across instances: pull work from shared infrastructure, then make execution idempotent.

```go
func handleReminder(ctx context.Context, store Store, userID int64, fireAt string) error {
    key := fmt.Sprintf("reminder:%d:%s", userID, fireAt)

    alreadyDone, err := store.MarkOnce(ctx, key)
    if err != nil {
        return err
    }
    if alreadyDone {
        return nil
    }

    return sendReminder(ctx, userID)
}
```

Rust ends up with the same philosophy. The syntax changes, but the contract stays the same: do not assume “at most once,” defend against duplicates.

```rust
async fn handle_reminder(store: &Store, user_id: i64, fire_at: &str) -> anyhow::Result<()> {
    let key = format!("reminder:{}:{}", user_id, fire_at);

    if store.mark_once(&key).await? {
        return Ok(());
    }

    send_reminder(user_id).await?;
    Ok(())
}
```

Zig is a nice reminder that good systems design is not about having the cleverest runtime. Sometimes it is just about being honest and explicit about state:

```zig
const std = @import("std");

pub fn buildJobKey(
    buf: []u8,
    user_id: u64,
    fire_at: []const u8,
) ![]const u8 {
    return try std.fmt.bufPrint(buf, "reminder:{d}:{s}", .{ user_id, fire_at });
}
```

That small bit of explicitness goes a long way. Once a job has a durable identity, retries stop feeling scary. You can acknowledge messages later. You can recover from crashes. You can let multiple instances race to process work without fearing duplicate side effects.

The biggest improvement usually does not come from adding a heavier scheduler. It comes from deciding, very clearly, which part of the system owns the truth and how workers prove a job has already been handled.

When teams get this right, background tasks stop feeling mystical. They become ordinary, observable work. And that is usually the best sign that the design is healthy.
