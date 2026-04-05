---
title: 'Scheduling One-Off Jobs in Go Without Adding Another Service'
description: 'A practical look at cross-instance background tasks with Postgres, Redis, NATS, and why the simplest option usually wins.'
pubDate: 'Apr 05 2026'
heroImage: '../../assets/blog-placeholder-2.jpg'
---

There is a very specific kind of backend problem that looks tiny at first and then quietly grows teeth: “run this task on Tuesday by 5 PM.”

If you only have one app instance, the answer is easy. Start a timer, wait, run the job.

But once your app runs on multiple instances, that innocent little requirement becomes a coordination problem. If every instance schedules the same task, you risk duplicate execution. If only one instance owns the schedule, restarts become dangerous. And if the job matters, “probably runs” is not good enough.

The first instinct is often to reach for a dedicated workflow engine. Tools like Temporal are excellent, but they also come with an operational cost: you either pay for the cloud product or host it yourself. For many teams, that is too much machinery for a single delayed task.

The better question is usually not “what is the most powerful scheduler?” but “what do we already trust in our stack?”

If you already have Postgres, a database-backed scheduler is often the most boring and reliable solution. Store the task in a table, let all instances poll for due work, and atomically claim it so only one worker wins. It is simple, durable, and easy to reason about.

If Redis is already in the picture, a sorted set can work well too. Put jobs in a shared queue keyed by execution time, then use an atomic claim step to move due jobs into a processing state. It is fast, lightweight, and effective, but you do take on a bit more responsibility around leases, retries, and recovery.

And if NATS is already part of your stack, JetStream can be a very clean fit. Instead of inventing a scheduler in your app, you let the broker hold the delayed message and deliver it at the right time. That keeps scheduling cross-instance by design, and it fits naturally with worker-style consumers.

The real lesson is that background scheduling is less about timers and more about ownership. Where does the schedule live? What happens if a node dies? How do you prevent two instances from doing the same work? Once you answer those three questions, the implementation tends to get much simpler.

In practice, the best solution is usually the one that adds the least new infrastructure. If your team already runs NATS well, use NATS. If Postgres is your most dependable shared system, use Postgres. If Redis is already your coordination layer, that is a perfectly reasonable place too.

A one-off task does not need a dramatic architecture. It just needs one durable source of truth, one safe claim path, and a handler that is happy to be retried.
