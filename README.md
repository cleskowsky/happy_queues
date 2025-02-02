# The secret to happy queues

Companion for my Rubyconf 2022 talk: "The secret to happy queues"

[![alt text](talk_thumbnail_rubyconf.jpg)](https://www.youtube.com/watch?v=pxHQ-hYjZKA)

If you haven't seen the talk, I'd recommend doing that first, otherwise this won't make a lot
of sense. :-)

This document includes information that is hopefully useful to implement the change i'm describing,
but that didn't fit in the time slot for the talk, or was too distracting from the main point.

## Slides

[Slides deck from the talk (without script)](Slides.pdf)  
[Slides deck from the talk (with script)](Slides_Script.pdf)

# Implementation Considerations

## An easy starting point

When setting up latency-based queues, which thresholds to use can be a matter of great debate.

It is possible that you already know what the right values would be for your particular situation,
but if you don't, I propose an "easy starting point" of:

- `within_1_minute`
- `within_10_minutes`
- `within_1_hour`
- `within_1_day`

It seems [other companies have gone for similar](https://engineering.gusto.com/scaling-sidekiq-at-gusto/#it-begins-with-latency).

This is not going to be perfect for your app, but it'll be a starting point, and can get around
much of the initial bikeshedding.

I also recommend, however, that as developers start placing jobs in these new queues, they add
comments at the top of the file describing their **actual** latency tolerance, besides these 
pigeonholes we've defined.

If a job can tolerate 30 minutes of latency, you'll put it in `within10minutes` because `within1hour` is too slow... 
**Document that**. If we ever add a a `within30minutes` queue, you know you can move that job immediately, 
and relieve some pressure on the faster queue.

Ideally, you'd also want a "why" in those comments. "If this job goes over 30 minutes, X consequence happens".

Further down the road, you can use these comments to review your queues setup. 
If the setup is very out of alignment, you'll end up with too many jobs in a queue that is faster than they need.
If and when you notice that, you can re-evaluate your queue threshold, and these comments will
provide actual data "from the field."

This will make the decision much easier than speculating about ideal thresholds before seeing
actual data from real jobs.

**Don't**:
- Be too granular from the get-go: It will actually make it harder to figure out where to put jobs.
- Be too ambitious with your fastest queue: More of this below


## Migrating from an "old" queue setup

I touch on this at the end of the talk, but I think it bears bullet-pointing.

It's possible you are lucky and have a greenfield project, I'm very jealous.
But if like most of us you have a running system... You'll need to migrate to this new structure.
And it can be quite painful, going through hundreds or thousands of job classes and figuring 
out where they should fit.

A key concept to keep in mind here is **why** we want this new queue setup: We want to isolate
loads so they don't interfere with each other, and make other jobs "late". 
The corollary from this is that if you get to a point where your jobs aren't likely to be
"late", you don't need to keep going. This will make more sense in a second.

**Do**:
- Set up these new queues, with dedicated servers running them, and with alerts if they go
  over their allowed latency. If you use autoscaling, set it up for them.
- Look over your existing queues. Are there queues that you can obviously move wholesale?
  - e.g. If you have queues with "nightly db maintenance" jobs, or "nightly sync" jobs, etc,
    all those jobs can probably just go to `within_1_day`.
  - If you have jobs that regularly take well over an hour to finish, those can also probably 
    go to `within_1_day`. No one is expecting them to finish quickly. YMMV, though, you might
    have time sensitive long-running batch jobs! If you do, those might require a special queue...
  - Other jobs might give away their lack of latency sensitivity by their sheer name.
- After that, start categorizing and moving your longest running jobs. These are the most likely to
  ruin your day, so are the most important to categorize. And they won't "fit" in most of your queues,
  due to their time limits, making the choice easier.
- Follow by categorizing you highest volume (most enqueued per day) jobs. Those are the other
  greatest candidate for ruining your day.
- Queues that have the fewest different job classes in them are also a good candidate to take on. 
  They will take a lot less work, and will help with the momentum as your teams see the total number of queues
  go down. This is a slight "mis-prioritization", but motivation will be key in this project!
- Once you move all jobs out of a queue, **delete the queue**. Remove the servers. 
  This saves you some money, but most importantly, seeing the queue list shorten will give 
  your teams a sense of progress that is extremely motivational!
- Importantly, know when to stop. Once you've categorized your longest and most frequent jobs,
  you may end up with a mixed bag of hundreds of different classes that remain uncategorized...
  If they are, in aggregate, low volume enough and they don't take huge amounts of time to run...
  They may not be worth categorizing. They'll be unlikely to get in each other's ways, so:
  - Bundle them all into the `default` queue.
  - Set a latency limit for `default` that is 2x as high as it gets in a week.
  - Call it a day.
    - If that latency gets breached at some point, looking at your metrics will probably point 
      at a clear culprit. Categorize that one into the correct queue and leave the rest alone.
    - If at some point someone notices "one of those jobs didn't run" on time, you'll have a 
      clear latency tolerance defined for you. Categorize that one into the correct queue and leave the rest.
    - **This will feel unsatisfying**, but you probably have bigger technical debt fish to fry at this point.
      You're not aiming for perfection here, you're aiming for queues that are easy to keep healthy. 

**Dont**:
- Don't put a job in a queue faster than it needs, just because it finishes fast. 
  - There's going to be a huge temptation to do this, because it's easier than figure out
    how much latency your job can actually tolerate. Remember that the runtime limits we just 
    discussed are a "minimum speed" requirement, they are not a queue recommendation. 
  - Just because you **can** be in a queue doesn't mean you **should**. If no one will notice 
    when your job runs 10 hours later, it doesn't matter that it finishes in 1 millisecond, 
    put it in the slower queues, don't let it get in the way of the jobs that need the quick latency!
  - Make sure to communicate this point **super** clearly with your teams, or you'll end up with your 
    fastest queue clogged with very-fast very-unimportant jobs!
- Don't allow folks to put new jobs into the "old queues". If you use Rubocop, 
  use something like [this example custom Cop](latency_queues_cop.rb)
  that will reject files that go into the queues that don't follow the new naming pattern... 
  - Add all the existing jobs to the "Rubocop ignore list", so existing jobs are grandfathered in,
    but nothing new will be allowed in the old queues..

## Realistic Expectations

It's important to have realistic expectations. Don't create a queue called `within1second`. 
You're writing checks you can't cash. It's **possible**, that under some very tight circumstances 
you **might** be able to pull that off... But for the vast majority of projects out there, 
it's extremely unlikely, and you're making life really hard for yourself.

Sure, `within1minute` doesn't sound great. Waiting for 1 entire minute sounds like a lot, 
and some colleagues will want to have guarantees of faster jobs... But you'll need to keep in mind some considerations...

1. Deployments: The normal way to deploy an app is to shut down your servers, and boot up 
   new ones with the new code. That takes a while, during which your queue isn't running. 
   If you want your queues to guarantee latencies lower than or near that boot time... 
   You will need to overlap your servers, boot up new ones first, shut down old ones later. 
   Under some environments, this can be harder to do. If you use Heroku, for example, you simply can't.
2. Autoscaling: If you want to use autoscaling to deal with spikes, you need to keep in mind 
   that the new servers take a while to boot up. If your queue is `within10seconds`, and your 
   server takes 30 seconds to start, there's no latency value you can set autoscale to. 
   If at any point you will need more servers, you've breached your SLO already.
3. Spikes: The point of queues is to be able to handle variability in demand. In other words, spikes. 
   If your SLO is extremely tight, it'll be very sensitive to spikes, and again, you won't be able 
   to autoscale to deal with them. You will need your number of threads to be set high enough 
   to deal with worst case scenario traffic **at all times**. That gets expensive fast.

I believe `within1minute` is pretty achievable. 
`within20seconds` **could be**, if you have a lot of control over your infrastructure, 
and you are **very disciplined**. But it'll be **hard**. Anything less than that... 
I personally think you're going to have a sad time.

Also keep in mind that the latency you're declaring is going to be the **maximum** that queue can have... 
It's not the "typical" latency. For your fast queues, you'll normally have enough threads 
that latency will be at 0 the vast majority of the time. But you're giving a fair notice 
that latency can go up to 1 minute at times.

**Don't:**
- Set up a "realtime" queue. This is a bad idea for all the reasons presented in the talk,
  namely that it's a very vague queue name with no clear threshold for alerting, no clear SLO,
  and by the nature of its name, it's guaranteed to disappoint everyone putting their jobs there.


## Using priorities instead of separate queues

Just don't. 

If you use Sidekiq, you aren't even allowed to do this (good!), but some 
queueing systems let you set priorities for your jobs. What this means is, threads will 
always pick up the highest priority job they can first. So if you have a bunch of low 
priority jobs, and you enqueue a high priority one, that one "jumps the queue".

This sounds like a good idea in principle, if a job is high priority, we **do** want to 
run it before any lower priority ones... But it's one of those behaviours of queues that 
can be unintuitive.

The problem is that you will have **some** jobs in your queues that are low priority and 
run for a long time. As long as one of those start running, that thread is now gone for 
every other job, no matter its priority.

Inevitably, at some point, all your threads will be stuck running slow low priority jobs, and 
there will be no one left to pick up the higher priority ones. **For quite a while**. Autoscale may 
help to some extent, but either you are committing to having infinite threads, or you'll end up clogging
all of them eventually.


**But I really like priorities!**

Ok, ok, there is **one** way. If your queue system allows you to have a "max priority" threshold
setting, you **can** have priorities, as long as you set you servers up correctly:
- For each priority level you want in your system, you need a separate set of servers that'll
  pick up jobs **up to that priority** but nothing lower.
- That way, you will have a "high priority" server that will **only** pick up high priority jobs. Its threads will run
  idle frequently, even if you have lower priority jobs waiting to get picked up, but you must
  let them be idle.
- A "lower priority" server can, however, pick up higher priority jobs, since those will finish
  faster than its "normal load", and shouldn't become a problem.

More on this in the "Helping other queues" section below.


## Know when to break the rules

The latency-based queue structure I describe in the talk... It's a guiding principle. 
It's going to be good for most of your jobs and most of your use cases, and you should generally follow it...
But there are good reasons to break out of this mold...

For example:
- Sometimes you need "singleton" queues. These are rare, and generally an anti-pattern, but sometimes 
  there are good reasons to make sure no 2 jobs in a queue are running at the same time. 
  - Make sure the queue name still declares an SLO, and suffix it with `singleton`, so everyone knows not to autoscale it!
- Sometimes you need special hardware for some jobs. In Ruby this generally means "a lot of RAM". 
  Instead of making all the servers for that queue bigger, you could have a `high_memory` version
  of the queue, for only those hungry jobs, and have only a few big, expensive servers.
  - But make sure the queue name still declares an SLO, and suffix it with `high_memory` so it's obvious what it's for.
- You may also have jobs that are "high I/O", which spend most of their time waiting and not using CPU. 
  Sending out emails or webhooks are typical examples. For those jobs, you could **wildly** crank up 
  the thread concurrency and save money on servers... But if you also get jobs that need a lot of CPU
  in the same queue, those will suffer from the thread contention. You can have a "high I/O" queue for 
  these types of jobs, with tons of threads per process, for jobs that can take advantage of the extra concurrency. 
  - You know what i'll say by now... Just make sure the queue name still declares an SLO, and suffix it with something like `high_io`.

Of course these are not the only cases, but I hope you're sensing the theme here. 
You can set up more specific queues if you need to... But always be explicit about the
latency guarantees for those queues, or you'll run into trouble again.



# Operational Considerations

Thoughts and tips for running in production with this queue structure.

## Sizing the servers for the new queues

A very common pattern with Ruby queues is that "a queue will run out of RAM", so the servers
get configured to have more memory available, but it'll frequently be hard to track down
which particular job(s) are causing the issue to try and optimize them.

Because of this, these servers tend to only grow.

The migration to new queues will provide a great opportunity to improve on this. Once. If you size
the new servers to an amount of RAM you consider "should be reasonable for your app", the fact
that you're gradually adding jobs to them will make it easier to pinpoint which jobs may be problematic,
by looking at which new jobs just got added in the last few hours / days.

Unfortunately, this is a one-time gain. Once the migration is complete, you will be back in the
position of existing jobs getting gradually bigger over time, and it being hard to figure out
which is the problematic one.

But as a one-time gain, you can try an under-size your servers and identify troublesome existing jobs.


## Helping other queues

Some background job systems like Sidekiq allow you to specify a number of queues to run "together" in one process.
Generally, these are ordered, such that while there are jobs in the "first" queue, the rest are ignored.

In these circumstances, you might be tempted to set your servers to run not only "their" queue, but also
the others, so that idle threads can serve as extra capacity while their queues are empty.

Be careful doing this. If you allow the server for a given queue to "help" higher latency (slower) queues,
you might end up picking up a "low priority, long running" job, which may tie up your thread for a while.
After a while, all of your threads may be clogged with long running jobs, and there may be no one left to
run the "higher priority" jobs.

This problem is precisely the reason we want time limits on our jobs. If you try and "help" slower
queues, you're undoing those time limits.

**When it is a good idea**

That said... You **can** do this in one-direction only. And it may actually be a good idea.
You can have your servers that process "slow" queues also help out with faster ones. That's fine.
Those "faster" jobs should get out of the way faster than that queue's own jobs would, and arguably
if there are jobs to run in lower latency queues, those should go first.

There are also circumstances where this may be actually necessary. As mentioned in "Realistic Expectations"
above, it's possible that your server's boot time might not allow you to autoscale effectively for your 
fastest queue, you may not be able to get new servers fast enough.

If you find yourself in this situation, it might be a good idea to set **all** your servers to run the fastest
queue in addition to their own, thus providing you with extra threads for your most sensitive queue. This is not
a perfect solution (all those threads from "slower" queues could still get tied up with long-running jobs), but in 
practice it can help significantly.

Just remember to never have any queues "help out" with anything slower than they normally do.
This means your fastest queue will frequently have idle threads. This is sadly actually necessary to keep 
the latency down.


## Interrupting potentially intensive jobs (see also: data migrations and backfills)

One of the advantages of having queues organized by latency is that you can stop serving a queue
at any point, and you know how long you have until you need to start it back up again, and you also 
know for sure the functionality driven by that queue won't be affected as long as you stay within
the latency guarantee.

Some jobs can put a lot of strain on your database. Data migrations and backfills are a great
example of this. You should put them in your "slowest" queue. (I'm going to use `within_1_day` 
for this example).

If you do, and one of these jobs is adding a lot of load to your database and causing trouble,
you can easily stop that queue's servers, and know for a fact nothing else should be affected.
This will fail the job and generally make it retry, and it gives you time to remove the job 
from that "retry" queue and start the queue again.

This is useful in systems like Sidekiq were stopping a running job is hard or not at all possible.

It also makes sense to put these database intensive jobs in the "slowest" queue 
because it's the one more likely to get turned off under intense load (see point below).


## My database is getting slammed!

Similar to the previous point, if you are experiencing high load on your system, you now know
that you can very easily shut down the "high latency" queues without causing any trouble to the
system. If shutting down `within_1_day` is not enough, you can continue with `within_1_hour`, and
you may have reduced the load enough to have bought yourself an hour to deal with the issue at hand.
Hopefully.

Even if that time is not enough, breaching latency in the "slower" queues is likely to be more tolerable
to your system that breaching latency on the "faster" queues, which can happen if your database is getting
hammered.

This can be a convenient "killswitch" to have, with its guarantee that nothing will go wrong if you 
turn it off for a specific amount of time.


## Manage your developer's expectations 

This is a great time to quote [Hyrum's law](https://www.hyrumslaw.com/) ([Obligatory XKCD](https://xkcd.com/1172/))

> With a sufficient number of users of an API, it does not matter what you promise in the contract:
> all observable behaviors of your system will be depended on by somebody.

This applies to our queues too, and there's a decision to make about the way you run your system in 
production.

When you have queues that declare they might be high latency, it's sometimes still easy to keep
their latency near 0 most of the time. Generally speaking, a queue with _only a little bit_ more capacity than its
workload demands will often have a latency near 0. The key word is "often". They will also spike to very high
latency occasionally.

You might want to do this. Having queues whose latency is near 0 most of the time feels **great**.

You might also not want to do this, however. For queues where that latency **can** spike to significant delays, if it almost
never does, it can lead to jobs that unintentionally depend on that "artificial" low latency, which
may cause issues when the system start experiencing higher load. Keep in mind that you won't
add more servers until that latency goes *significantly* high compared to it's frequent near-zero
state...

This is tricky to fight against. You don't want to intentionally slow down your queues (although 
it may not be the worst idea, if done carefully!). But you might want to keep your server resources
low when latency is, say, under 20% of your queue's limit, to let it grow a bit and have jobs 
regularly experience *some* latency, so any problems can be detected early.


# Monitoring your queues

You know you need good monitoring for your queues, but it may not be obvious what that looks like, or how to do it. 
Here are some recommendations on what yo aim for, but keep in mind that, in practice, what the best setup is 
may vary quite a bit depending on your background job system, and your metrics / observability platform.

## Ideal Metrics

The metrics you absolutely want to have are: (higher ones in the list are more important)
- Queue latency per queue
- Jobs Worked (Finished) Counters per queue and per job class
- Jobs Enqueued Counters per queue and per job class
- Job Duration Statistics per queue and per job class
- Total time spent per queue and per job class
- Queue Size per queue
- Workers / Threads running per queue

### Queue latency per queue

First, a couple of definitions:
- The "first" job in your queue is the one that will execute next, and it's by definition the oldest / the one that's been waiting the longest.
- Queue latency is the time elapsed between that "first" job getting enqueued, and now. In other words, how long has the longest-waiting job has been waiting.

This is your most important metric, it's the most important signal of whether you're in trouble,
and pretty much the thing i'm talking about through the entire talk. If you can only get one metric,
this is the one.

Most importantly, this metric is what your alerts will be based on, to know if your queues are unhealthy.

You want to have one time series per queue. If your metrics system does tagging / labeling, 
the `queue` name will be the main tag / label.

In most metrics system, this metric will be a Gauge (because it can go up and down over time).

**A handy trick for alerting: Normalized latency**

In addition to reporting the actual latency, in seconds, it can also be a good idea to report
the latency as a percentage: how much of the allowed latency are we experiencing right now.

So, if your `within_10_minutes` queue has a latency of 4 minutes, you'll report those 240 seconds,
but in a separate, "normalized" metric you'll report 0.4.
A latency of 4 minutes on your `within_1_hour` queue, however, would report 0.066.

This makes it very easy to set up alerts that are agnostic to your actual queue setup.
Instead of having separate alerts with separate thresholds for your queues, you can set only
one alert, that triggers for any queue where `normalized_latency > 1`.

It's also useful to visualize in your dashboards. A high latency on a queue that allows it may
look bad in a graph of "latency per queue", and it may seem worrying when it isn't. On the normalized graph,
lines only go up proportional to how much that matter.

### Jobs Worked (Finished) Counters per queue and per job class

You want a counter per Job Class, tagged with both `job_class` and `queue`, that is incremented
every time a job finishes. This allows you to know how frequently each of your jobs is run, which
is going to be very important to determine what are your highest volume jobs in any queue.

You might also want to add a `status` label, with "success / error", which will let you easily see
how often a given job class is failing (or overall jobs in a queue are failing). This is a vital health
metric, if those numbers start shooting up, you're probably having issues somewhere.

### Jobs Enqueued Counters per queue and per job class

Same as before, but the counters would be incremented every time a job is enqueued. This is useful
mostly to compare with the previous metric, which will allow you to see if jobs are getting enqueued
faster than the system can cope with. 

This is fine for a short window of time (and to some extent, dealing with those spikes is a large part of why we use queues),
but if sustained, you may find yourself in trouble, with runaway queues that keep growing and can't keep up.

It will also make it very easy to find out why a queue is suddenly in trouble. You might be able to spot that a 
given job got suddenly enqueued **way more** than normal, and you can take actions based on that.

### Job Duration Statistics per queue and per job class

You want to know, for each job, how long they are taking to run. You will update this metric each time
a job finishes, and you should be able to get statistics out of it.

Different metric systems have different ways of doing this. In Datadog, for example, you could use a 
Histogram or a Distribution (the difference between them is subtle and picking between them depends on
your particular situation). If you are using Prometheus, you would use a Histogram.

Generally this will give you an `avg` duration, and "approximate percentiles" (median, p95, etc). 
Please note, **THESE PERCENTILES WILL ALWAYS BE A LIE** to some extent or another. They will be approximate, 
and how much you can trust them depends on the specifics of your data, and your particular metric system.
So take them with a pinch of salt. `avg` (mean) can either be accurate or also a lie, again depending on 
how your particular metric system stores samples. Generally speaking, you can probably trust it more than percentiles. 

That said, even though these won't be exact, they will still give you the information that you need to understand
the performance characteristics of your jobs.

In particular, this will help you find your "trespassers", jobs that are too slow for the queue they're in.

### Total time spent per queue and per job class

This is the total time you spent on a given job. Imagine you enqueue 1000 jobs of a given class, 
and they all run in 2 seconds... The average duration (the previous metric) is 2 seconds. The total
duration (this metric) is 33 minutes (2000 seconds).

This is an important metric for similar reasons to "Jobs Worked". You want to know not only which 
jobs you run most frequently, but also where you are spending most of your time, and especially
how much time in total you're spending on your queues, to allow you to estimate how much hardware /
how many threads you will need.

This one is once again very dependent on your Metrics system, and you may or may not get it for free.
If you are using Prometheus, the Histogram you'll use for Job Duration will also report a `sum` for you,
which is exactly this metric. It will also report a `count`, which is exactly "Jobs Worked", so just one
histogram covers all three cases.

If you are using Datadog, you don't get that `sum` (if you can find it, **please** tell me where it is!).
You can approximate it doing `jobs worked * avg(duration)`, but you will find that this is also "a lie".
If you track your actual total (which I recommend), you'll see these don't match, and they can be quite off
sometimes. This implies to me that the `avg` I talked about earlier is also and approximation and not exact, 
but I may be wrong here.

Unfortunately, with this level of detail, the devil is in the internal of your metrics system.
I recommend tracking `total_duration` as a **counter** that you increment for each job you run, 
so you get a precise number. And then you can also get a precise average doing `total_duration / jobs worked`,
if the `avg` you get from your metrics is not precise enough.

### Queue Size per queue

The Queue Size is the number of jobs currently in the queue. It will be a Gauge. 

This is useful for diagnosing problems sometimes, it can let you easily see if a queue is much larger than normal.
Combined with Jobs Enqueued, it can make it easy to spot problems quickly.

**BUT**, it is the least important metric in the list. You will not use this for analysis of your queues, 
for doing any sort of planning or making decisions, and you will **definitely** not use it for alerts.

Some background job systems unfortunately don't make it easy to get this metric, so if you can't get it, 
don't worry about it too much.

### Workers / Threads running per queue

Also a gauge, indicating how many workers you have processing a given queue at any given time.
This is generally only important if you use autoscaling for your queues, and it lets you see,
when you're in trouble, whether autoscaling did what it was supposed to.

If you can obtain it, a separate gauge letting you know how many of those threads are busy at the 
moment is also useful (mostly to know whether all threads are busy / clogged at a given time).


## Obtaining / reporting these metrics

### Queue latency and Queue Size per queue

Your background job runner might give you these directly (Sidekiq does), or if you're able to peek into the queue,
you may be able to calculate this yourself.

You want to run a cron every N seconds (don't run more often than your metrics system's granularity!), read the latency
and size off your queues, and report the value for each of the gauges. This is also where you'd calculate and report normalized latency.

Check `sidekiq_periodic_metrics.rb` for an example you can use as a starting point and modify for your needs.

### Jobs Enqueued Counters per queue and per job class

You will need to increment / report this metric every time a job gets enqueued. Ideally, your
metrics system allows you to "hook" into the enqueueing process so you can do this 
(Sidekiq does this through the use of Client Middlewares).

Check `sidekiq_enqueue_metrics.rb` for an example you can use as a starting point and modify for your needs.

This example is unfortunately very Sidekiq-specific. For other queueing systems, the logic may be quite different,
but you're essentially just figuring out what Queue and what Job you're dealing with, and incrementing a counter.


### Jobs Worked Counters and Duration statistics per queue and per job class

Similarly to the previous one, you will need to increment / report this metric every time a job 
finishes running. Hopefully you have a way of "wrapping" jobs that run by hooking into some part
of your queueing system, through a common base class for all your jobs, or something similar.
(Sidekiq does this through the use of Server Middlewares).

Check `sidekiq_execution_metrics.rb` for an example you can use as a starting point and modify for your needs.

This example is unfortunately very Sidekiq-specific. For other queueing systems, the logic may be quite different,
but you're essentially just figuring out what Queue and what Job you're dealing with, and timing the execution.
