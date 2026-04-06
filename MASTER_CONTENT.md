# Linux CPU Scheduling: CFS, EEVDF and What the Kernel Actually Guarantees

## Introduction

Linux's fair scheduling story now has to be told in two layers. The first layer is historical and conceptual: the Completely Fair Scheduler (CFS) modeled fairness through weighted virtual runtime and a run queue ordered by `vruntime`.[1, 5] The second layer is current: Linux's fair class is now documented in terms of Earliest Eligible Virtual Deadline First (EEVDF), which keeps the same weighted-service foundation but adds lag, eligibility and virtual deadlines as the explicit selection vocabulary.[2, 4]

This article is deliberately source-conservative. It focuses on claims that can be grounded in Linux kernel documentation, Linux kernel source and one historical pre-EEVDF fair-scheduler source file used for the CFS implementation walkthrough.[1, 2, 3, 4, 5]

## Why Scheduling Is Hard

Three objectives dominate the fair-scheduler design space.[1, 2, 5]

1. **Fairness**: over time, CPU time should track task weight.
2. **Latency**: short and interactive work should not wait arbitrarily behind long-running work.
3. **Efficiency**: the scheduler must avoid turning fairness into excessive context-switch and migration overhead.

The CFS design document framed fairness as an approximation to an "ideal, precise multi-tasking CPU" that runs all runnable tasks simultaneously at a fractional rate. The real scheduler cannot run that machine, so it instead tracks each task's normalized progress through virtual runtime (`vruntime`).[1] The nice-design document then explains why Linux's nice values were redesigned to be multiplicative: the effect of a relative nice difference should not depend on where in the nice scale the tasks happen to sit.[3]

Two historical cautions matter for the rest of the article:

- The CFS design document describes the original CFS model and its priorities; it is not a full description of later EEVDF-era implementation details.[1]
- The EEVDF documentation describes the current fair scheduler's mathematical model, but it does **not** claim that `SCHED_OTHER` provides hard real-time wall-clock deadlines.[2]

Before going further, the basic terms matter:

- A **process** is a running program instance with its own address space and resources.
- A **thread** is an execution stream inside a process; Linux scheduling often acts on threads, even though operating-systems courses often begin by talking about "process scheduling."
- A **scheduler** is the kernel component that decides which runnable task executes on a CPU next.
- A **time slice** is a bounded amount of CPU service a scheduler is willing to give a task before reconsidering the decision.
- **Fairness** means that over time, CPU service should track policy such as task weight.
- **Latency** means how long a task waits from becoming runnable to actually receiving CPU time.
- A **run queue** is the scheduler's data structure for tasks that are currently available to run.

Pedagogically, one useful analogy is a checkout line. Fairness asks whether everyone eventually gets service in proportion to the policy. Latency asks how long someone waits before the cashier starts handling the next item. The rest of the article formalizes that analogy without relying on it.

## CFS

CFS begins from a simple problem: among runnable tasks, which one has received the least normalized progress so far?[1]

### 1.1 Overview and Core Concept

CFS made fairness its primary invariant. Its central idea was simple: if all runnable tasks progressed on an ideal processor, then each task would accumulate the same normalized amount of progress; on real hardware, the task with the smallest normalized progress is the most deserving of the CPU.[1] CFS encoded that normalized progress as `vruntime` and the core selection rule was therefore "pick the runnable entity with the smallest `vruntime`".[1, 5]

Unlike classic fixed-timeslice schedulers, CFS treated slice length as a derived quantity rather than a fixed per-task allotment. The historical CFS source explicitly states that the configured latency target is not a persistent "timeslice length"; CFS slices are variable and derived from weight and runnable-task count.[5]

### 1.2 Fairness Guarantees, Weights and Key Data Structures

In the historical CFS implementation, the per-entity and per-run-queue state that matters most is:

- `struct sched_entity`: per-task or per-group fair-scheduler state, including `vruntime`, `load.weight` and the red-black-tree node used for queue ordering.[5]
- `struct cfs_rq`: the fair run queue, including `tasks_timeline`, `min_vruntime`, runnable counts and aggregate load information.[1, 5]
- `tasks_timeline`: the cached red-black tree of runnable entities, ordered by `vruntime` in pre-EEVDF CFS.[5]
- `min_vruntime`: the monotonic lower envelope of run-queue progress, used to keep task placement from moving backward in virtual time.[1, 5]

The historical source shows the core ordering primitives:

`kernel/sched/fair.c` in Linux v6.5 (lines 3158-3164 and 3243-3253)  
<https://github.com/torvalds/linux/blob/v6.5/kernel/sched/fair.c#L3158-L3164>  
<https://github.com/torvalds/linux/blob/v6.5/kernel/sched/fair.c#L3243-L3253>

```c
static inline bool entity_before(const struct sched_entity *a,
				 const struct sched_entity *b)
{
	return (s64)(a->vruntime - b->vruntime) < 0;
}
```

```c
struct sched_entity *__pick_first_entity(struct cfs_rq *cfs_rq)
{
	struct rb_node *left = rb_first_cached(&cfs_rq->tasks_timeline);
	if (!left)
		return NULL;
	return __node_2_se(left);
}
```

These are not pseudocode. They are the actual historical comparison and leftmost-pick operations that implement the classic CFS run-queue story.[5]

### 1.3 Time Slice Calculation, Accounting and Weighted Virtual Time

CFS normalized real execution time by weight. The common primitive was and remains `calc_delta_fair()`:

`kernel/sched/fair.c` in Linux v6.5 (lines 3323-3331)  
<https://github.com/torvalds/linux/blob/v6.5/kernel/sched/fair.c#L3323-L3331>

```c
static inline u64 calc_delta_fair(u64 delta, struct sched_entity *se)
{
	if (unlikely(se->load.weight != NICE_0_LOAD))
		delta = __calc_delta(delta, NICE_0_LOAD, &se->load);
	return delta;
}
```

Source: Linux v6.5 `kernel/sched/fair.c`.[5]

When a task executes, CFS increases `vruntime` by weighted runtime, so heavier tasks accumulate `vruntime` more slowly for the same wall-clock service. That is the mechanism behind proportional sharing: if task `i` has a larger weight, it can run longer before becoming "too far ahead" in virtual time.[1, 3, 5]

The historical source also converts wall-clock slices into virtual slices through `sched_vslice()`:

`kernel/sched/fair.c` in Linux v6.5 (lines 3448-3452)  
<https://github.com/torvalds/linux/blob/v6.5/kernel/sched/fair.c#L3448-L3452>

```c
static u64 sched_vslice(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	return calc_delta_fair(sched_slice(cfs_rq, se), se);
}
```

Source: Linux v6.5 `kernel/sched/fair.c`.[5]

That is the exact code bridge between CFS's wall-clock slice calculation and the virtual-time accounting that ultimately drives tree order.[9]

### 1.3.1 Ten-Task Slice Example

To make the slice calculation concrete, consider ten runnable tasks: eight at weight 1024, one at weight 820 and one at weight 1277. Under the historical CFS tunables, the runqueue is beyond the nominal latency-width threshold, so the scheduling period is driven by minimum granularity rather than by the base latency target.[5]

A compact version of the arithmetic is:

- base period: `10 × 0.75 ms = 7.5 ms`;[5]
- eight weight-1024 tasks: about `0.746 ms` each before clamping;[5]
- one weight-820 task: about `0.597 ms` before clamping;[5]
- one weight-1277 task: about `0.931 ms` before clamping.[5]

The exact decimals matter less than the structure of the result. Once runnable-task count is high enough, CFS's global granularity rules begin to shape the actual wall-clock slices. Lighter tasks can be pulled upward by minimum granularity, while heavier tasks still accumulate `vruntime` more slowly for the same real execution time.[5, 9] This is a useful beginner example because it shows both halves of the CFS story at once: weight controls fair sharing, but global slice policy still affects how that sharing is delivered in time.[1, 5]

### 1.4 Simplified Scheduling Walkthrough

As a compact teaching example, consider three always-runnable CPU-bound tasks with weights 2048, 1277 and 820, all starting at `vruntime = 0`. Under classic CFS, each task's wall-clock slice is computed from its weight share of the scheduling period, but the *virtual* effect of running that slice is much closer across the three tasks because `calc_delta_fair()` advances heavier tasks more slowly in virtual time.[5, 9]

That means a plausible early evolution looks like this:

1. the heaviest task runs first and advances its `vruntime` by a relatively small amount for the wall-clock service it receives;[5]
2. the lighter tasks then run and catch up in `vruntime` more quickly for smaller amounts of real service;[5]
3. after a few such turns, the runnable tasks cluster near the same `vruntime`, which is exactly the state CFS is trying to maintain.[1, 5]

Now add a fourth equal-weight task that sleeps and wakes periodically for short bursts. While sleeping, it does not accumulate `vruntime`, so on wakeup it can re-enter with less virtual progress than the always-runnable tasks and therefore compete strongly for the CPU.[1, 5] Classic CFS can reward bursty sleepers naturally, but it does so through `vruntime` position and wakeup heuristics rather than through an explicit per-task short-request mechanism.[1, 5]

### 1.5 Wakeup Handling, Preemption and Heuristic Interactions

That simple walkthrough sets up the harder question: what happens when a task wakes in the middle of ongoing work?

Classic CFS had a strong heuristic component in the wakeup path. The historical source documents `sysctl_sched_wakeup_granularity` explicitly as a knob that delays preemption for some decoupled workloads while preserving immediate wakeup/sleep latency for synchronous ones.[5] This is important for comparative examples because it means wakeup behavior was not controlled solely by "smallest `vruntime` wins"; the scheduler also used a threshold to avoid over-eager preemption.[5]

This does not make CFS unsound. It means that its fairness model and its latency behavior were represented differently:

- fairness lived in normalized progress (`vruntime`);
- latency behavior was influenced indirectly through placement and granularity heuristics.[1, 5]

### 1.6 How CFS Uses Heuristics

The CFS design document famously described the original design as avoiding the large complexity of earlier heuristics, but the historical implementation still contained practical knobs and wakeup rules such as minimum granularity and wakeup granularity.[1, 5] For an academic reading, the right interpretation is:

- the *core fairness criterion* is simple and mathematical;
- the *latency and wakeup behavior* is still shaped by additional scheduler policy choices.[1, 5]

### 1.7 Drawbacks Leading to EEVDF

In general, three CFS limitations matter most for understanding why later fair-scheduler work moved toward EEVDF:

1. CFS did not provide a direct per-task way to express service-window preference separately from long-run fair share;[1, 5]
2. Short-term wakeup responsiveness depended partly on global thresholds and placement heuristics rather than only on an explicit urgency model;[5]
3. Tasks that needed low latency often had to obtain it indirectly through weight, sleep behavior or scheduler policy details rather than through a dedicated fair-class latency control;[1, 3, 5]

These are the pressure points the EEVDF documentation addresses most directly.[2]

### 1.8 Kernel Walkthrough

Historical CFS used global tunables to shape latency and context-switch behavior. The v6.5 source exposes `sysctl_sched_latency`, `sysctl_sched_min_granularity` and `sysctl_sched_wakeup_granularity`, together with the derived threshold `sched_nr_latency`.[5] It then computes wall-clock slices through `sched_slice()` and virtual slices through `sched_vslice()`.[5]

Two design consequences follow directly:

1. slice length depends on run-queue size and total load, not just on the task itself;[5]
2. latency tuning is largely global, because those tunables are global scheduler knobs rather than per-task deadlines.[5]

That is one of the reasons the later EEVDF work is best understood as a change in representation: Linux moved from global fairness-shaping tunables plus `vruntime` ordering toward explicit service debt and deadline state.[2]

### 1.8.1 Enqueue, placement and dequeue

The historical CFS logic can be summarized in four stages.[1, 5]

1. place a newly runnable entity near the current run-queue progress;
2. account its load into the fair run queue;
3. insert it into `tasks_timeline`;
4. on sleep or block, dequeue it and advance `min_vruntime` as needed.[1, 5]

The tree operations themselves were simple:

`kernel/sched/fair.c` in Linux v6.5 (lines 3228-3239)  
<https://github.com/torvalds/linux/blob/v6.5/kernel/sched/fair.c#L3228-L3239>

```c
static void __enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	rb_add_cached(&se->run_node, &cfs_rq->tasks_timeline, __entity_less);
}

static void __dequeue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	rb_erase_cached(&se->run_node, &cfs_rq->tasks_timeline);
}
```

Source: Linux v6.5 `kernel/sched/fair.c`.[5]

The more subtle step was *placement*: a waking task was not simply dropped into the tree unchanged. The CFS design document emphasizes that sleepers and interactive tasks were recognized in placement and the historical code combined that idea with wakeup granularity, minimum granularity and other heuristics.[1, 5]

The exact historical decision point is still the tree pick itself: once runnable entities are placed and enqueued, `__pick_first_entity()` returns the leftmost runnable entity and the surrounding wakeup/preemption logic decides whether a newly awakened task should force a reschedule before the current slice naturally ends.[5]

For accounting, the historical runtime-update path begins here:

`kernel/sched/fair.c` in Linux v6.5 (lines 3632-3652)  
<https://github.com/torvalds/linux/blob/v6.5/kernel/sched/fair.c#L3632-L3652>

```c
static void update_curr(struct cfs_rq *cfs_rq)
{
	struct sched_entity *curr = cfs_rq->curr;
	u64 now = rq_clock_task(rq_of(cfs_rq));
	u64 delta_exec;

	if (unlikely(!curr))
		return;

	delta_exec = now - curr->exec_start;
	if (unlikely((s64)delta_exec <= 0))
		return;

	curr->exec_start = now;
	/* ... */
}
```

This is where elapsed wall-clock runtime becomes scheduler-visible state. The omitted continuation updates execution statistics and then uses the weighted-accounting helpers discussed above.[5]

## From CFS Limitations to EEVDF

CFS's core limitation here is not fairness failure but representation. Weighted virtual progress is explicit, but short service-window urgency is not represented as a first-class per-task quantity in the original design.[1, 5]

Three consequences follow.[1, 5]

1. two equal-share tasks that want different service-window lengths are not directly expressible in the original CFS model;[1, 5]
2. wakeup behavior depends partly on global granularity and threshold policy rather than only on per-task state;[5]
3. interactive or bursty behavior is explained through placement and `vruntime` position rather than through an explicit debt-and-deadline pair.[1, 5]

## Why EEVDF Was Introduced

The EEVDF documentation can be read as a response to those specific CFS frictions. Instead of representing urgency mostly through position on a single `vruntime` timeline, it adds:

- lag, which expresses whether a task is behind or ahead of fair service;[2, 4]
- virtual deadline, which expresses the urgency of the task's current service request once it is eligible.[2, 4]

EEVDF therefore extends the fair scheduler's internal language rather than discarding weighted fairness.[2]

## EEVDF

EEVDF keeps weighted fairness but separates two questions that CFS mostly compressed into one: who is owed service and which owed task has the most urgent current request?[2]

### 2.1 Overview and Core Concepts

EEVDF keeps weighted fairness but changes how urgency is represented. The current kernel documentation describes the fair scheduler in terms of three quantities:[2]

1. `vruntime`: weighted service already received;
2. lag: whether the task is behind or ahead of fair service;
3. virtual deadline: the ordering key among eligible tasks.[2, 4]

The defining EEVDF idea is two-stage selection:

- first, restrict competition to tasks that are *eligible* because they are owed service;[2, 4]
- then, among those tasks, run the one with the earliest virtual deadline.[2]

This is stronger than "pick the least-served task" and still weaker than a hard real-time deadline scheduler. The model gives the fair class explicit control over *relative urgency*, not arbitrary hard wall-clock admission guarantees.[2, 4]

### 2.2 Fairness, Lag and Key Data Structures

The current fair scheduler still uses `struct sched_entity` and `struct cfs_rq`, but the fields that now matter most differ from the historical CFS explanation:

- `se->vruntime`: current normalized service position;[4]
- `se->vlag`: clamped virtual lag;[4]
- `se->deadline`: current virtual deadline;[4]
- `se->slice`: requested service slice used to derive the deadline;[4]
- `cfs_rq->sum_w_vruntime`, `cfs_rq->sum_weight` and `cfs_rq->zero_vruntime`: the tracked state used to approximate weighted-average virtual time.[4]

The current source comments explain that EEVDF's average-virtual-runtime machinery exists so the scheduler can represent lag without directly summing unmanageably large absolute values.[4]

### 2.3 Lag, Eligibility and Virtual Deadline Formation

The current source documents the central identity:

```text
lag_i = S - s_i = w_i * (V - v_i)
```

where `s_i` is service received by entity `i`, `w_i` is its weight, `v_i` is its virtual runtime and `V` is the weighted average virtual runtime.[4]

The same comment derives the eligibility condition:

```text
lag_i >= 0  ->  V >= v_i
```

So an eligible task is one that has not received more than its weighted fair share.[4]

The code-level update is compact:

`kernel/sched/fair.c` in current Linux (lines 3487-3501)  
<https://github.com/torvalds/linux/blob/master/kernel/sched/fair.c#L3487-L3501>

```c
static void update_entity_lag(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	u64 max_slice = cfs_rq_max_slice(cfs_rq) + TICK_NSEC;
	s64 vlag, limit;

	vlag = avg_vruntime(cfs_rq) - se->vruntime;
	limit = calc_delta_fair(max_slice, se);
	se->vlag = clamp(vlag, -limit, limit);
}
```

Source: current `kernel/sched/fair.c`.[4]

This replaces hand-wavy "interactive boost" language with an explicit tracked quantity: service debt or surplus.[7]

### 2.4 Simplified Scheduling Walkthrough

As a parallel teaching example, consider three always-runnable CPU-bound tasks with weights 2048, 1277 and 820, together with a fourth task of weight 1024 that wakes periodically and requests a short slice. In the simplified EEVDF model, the first three tasks start at the same `vruntime`, but their lag and deadline state quickly diverge as they run. A heavier task can receive more wall-clock service before advancing as far in virtual time, while a task that has run recently can become ineligible because it is now ahead of fair service.[4]

That produces a useful first-pass pattern:

1. among tasks that are still eligible, the earliest virtual deadline is preferred;[2, 4]
2. a task that has just consumed service can lose urgency by moving to negative lag;[4]
3. a periodically waking short-slice task can re-enter with competitive lag and an earlier deadline than longer-slice competitors.[2, 4]

After the first few CPU-bound turns, the waking task can rejoin the runqueue with little recent service debt and a short requested slice, so its deadline becomes earlier than that of the currently running longer-slice competitor. EEVDF therefore makes the "who is owed service?" question and the "whose current request is more urgent?" question visible as separate quantities, lag and deadline, instead of folding both into one `vruntime` story.[2, 4]

As with the compact CFS walkthrough, this is an analytical example under the scheduler's documented model, not a claim that the kernel will always produce these exact numeric transitions for those weights and slices.[2, 4]

### 2.5 Wakeup Handling

The walkthrough above explains steady runnable competition. The next issue is what happens across sleep and wakeup.

The current EEVDF documentation puts unusual emphasis on sleeping tasks because this is where lag becomes visibly different from older sleeper heuristics. A task with negative lag can remain marked for deferred dequeue, so briefly sleeping does not erase the fact that it already received more than its fair share. As virtual time advances, that lag decays; long sleeps eventually forgive the debt.[2]

This means the sleep/wakeup path is no longer just "was the task idle for a while?" It is instead "what service balance should the task carry across that sleep?"[2, 4]

### 2.5.1 Requested Slice and `sched_setattr()`

The EEVDF documentation explicitly states that tasks can request a specific slice using `sched_setattr()`.[2] The academically safe claim is:

- EEVDF lets a fair-class task express a shorter requested service window without changing its weight;[2]
- the kernel converts that request into an earlier virtual deadline through `calc_delta_fair(se->slice, se)`;[4]
- long-run share is still governed by weight and lag, not by "who asked for the shortest slice".[2, 4]

### 2.6 Where Heuristics Still Exist

It would be incorrect to conclude that current Linux fair scheduling is "purely mathematical" in every detail. Current `fair.c` still contains preemption actions, next-buddy handling and wakeup-special-case logic. For example, the wakeup path still contains `PREEMPT_WAKEUP_SHORT`, `PREEMPT_WAKEUP_PICK` and next-buddy handling that explicitly discusses deadline-aware buddy choice.[4]

So the careful comparison is:

- CFS represented both fairness and much of responsiveness indirectly through `vruntime` placement and granularity heuristics;[1, 5]
- EEVDF represents fairness and relative urgency more directly through lag, eligibility and deadlines, while still retaining implementation caveats in the wakeup path.[2, 4]

### 2.8 What EEVDF Still Does Not Guarantee

Three caveats keep the EEVDF story accurate:

1. EEVDF is part of the fair scheduler, not a replacement for `SCHED_DEADLINE`.[2]
2. The strongest latency claim here is relative ordering among eligible tasks, not a universal wall-clock bound.[2, 4]
3. Current Linux still includes implementation details such as buddy and wakeup logic, so "earliest eligible deadline" is the mathematical center of the policy, not the only line of code that matters.[4]

### 2.9 Practical Questions

#### 2.9.1 Deferred Dequeue, Lag Decay and When a Sleeping Task Can Run Again

The current EEVDF documentation says that sleeping tasks with negative lag can remain marked for deferred dequeue so that a short sleep does not let them discard overservice debt immediately.[2] The exact implementation-level answer to the question is:

1. **Sleeping** means the task blocked and is no longer runnable in the usual sense.
2. **Deferred dequeue** means the task is not immediately removed from the fair runqueue if the implementation decides to preserve its negative lag debt.[2, 4]
3. As other tasks continue to run, the runqueue's weighted average virtual time advances, so the sleeping task's lag can move toward zero and eventually become nonnegative.[4]
4. **That does not mean the task is immediately chosen to execute.** If it is still sleeping, it is not runnable in the ordinary sense.
5. When the picker later encounters such an entity while it is still marked delayed, `pick_next_entity()` calls `dequeue_entities(... DEQUEUE_SLEEP | DEQUEUE_DELAYED)` and returns `NULL` instead of dispatching it.[4]

The exact distinctions are:

- **dequeued**: physically removed from the fair runqueue data structures;
- **eligible**: passes the `V >= v_i` condition and is no longer ahead of fair service;
- **runnable**: the task has actually woken and is available for execution;
- **chosen to run**: among runnable and eligible competitors, it wins the actual selection step.[2, 4]

So the decisive condition for *being allowed to compete again* is eligibility; the decisive condition for *actually executing* is stronger: the task must be awake and runnable, then it must still win the deadline comparison among eligible runnable entities.[2, 4]

Implementation caveat: exact behavior depends on current scheduler feature settings such as delayed-dequeue handling and delayed-zeroing behavior, so the answer is implementation-specific rather than a timeless abstract law of EEVDF.[4]

#### 2.9.2 If a Task Should Run Every 10 ms, What Does EEVDF Actually Provide?

Under Linux fair scheduling, "scheduled every 10 ms" can mean several different things and they should not be conflated.[2, 4, 6]

1. **Periodic activation every 10 ms.**  
   This is primarily a wakeup and activation question. The available EEVDF sources describe how fair-class tasks are ordered once they are runnable; they do not provide an independent mechanism that periodically activates a task every 10 ms.[2]

2. **Relative urgency once the task wakes.**  
   EEVDF can help here. A task can request a shorter slice through `sched_setattr()`, which influences its virtual deadline once it is runnable and eligible.[2, 4] This can make a freshly awakened 10 ms periodic task more likely to run promptly relative to longer-slice fair-class competitors.

3. **Hard guarantee that it will run every 10 ms.**  
   The available EEVDF sources do not support that claim for `SCHED_OTHER`.[2, 4] If the requirement is truly "10 ms period with reservation-style or deadline-style guarantees," the primary-source scheduler designed for that is `SCHED_DEADLINE`, which uses explicit runtime, deadline and period parameters and is documented as the class for periodic or sporadic real-time tasks that need timing guarantees.[6]

The strongest accurate statement is therefore:

- EEVDF can help a task that wakes every 10 ms obtain *lower-latency fair-class dispatch* by combining positive lag after sleep with a shorter requested slice;[2, 4]
- it cannot, by itself, guarantee that a task will be activated or executed exactly every 10 ms under all conditions;[2, 4]
- for reservation-style periodic timing guarantees, use a different scheduling class such as `SCHED_DEADLINE`.[6]

### 2.10 Kernel Walkthrough

EEVDF uses requested slice plus weight-normalized accounting to construct a virtual deadline:

`kernel/sched/fair.c` in current Linux (deadline ordering at lines 3155-3179; protected-slice region at lines 3787-3814)  
<https://github.com/torvalds/linux/blob/master/kernel/sched/fair.c#L3155-L3179>  
<https://github.com/torvalds/linux/blob/master/kernel/sched/fair.c#L3787-L3814>

```c
/* EEVDF: vd_i = ve_i + r_i / w_i */
se->deadline = se->vruntime + calc_delta_fair(se->slice, se);
```

Source: current `kernel/sched/fair.c`.[4]

That one line is the heart of the "same share, shorter service window" story. Weight still controls long-run sharing. Slice controls how soon the scheduler wants the task to finish its current service request, **provided** the task is eligible.[2, 4]

The current source also uses deadline-based comparison among eligible entities:

`kernel/sched/fair.c` in current Linux (lines 3153-3179 and 3561-3565)  
<https://github.com/torvalds/linux/blob/master/kernel/sched/fair.c#L3153-L3179>  
<https://github.com/torvalds/linux/blob/master/kernel/sched/fair.c#L3561-L3565>

```c
static inline bool entity_before(const struct sched_entity *a,
				 const struct sched_entity *b)
{
	return vruntime_cmp(a->deadline, "<", b->deadline);
}
```

```c
int entity_eligible(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	return vruntime_eligible(cfs_rq, se->vruntime);
}
```

Source: current `kernel/sched/fair.c`.[4]

The formal selection statement is therefore simple: among entities that pass the eligibility test, the earlier deadline wins.[2, 4]

The exact current decision point is not just "compute a deadline." It is the combination of:

1. maintaining lag and deadline state on enqueue and runtime updates;[4]
2. selecting an entity through the EEVDF picker;[4]
3. finalizing delayed sleepers before they are allowed to disappear from the runqueue or compete as ordinary runnable entities.[4]

The exact delayed-dequeue picker path is important, but it is also implementation-sensitive and line-number-fragile in current `fair.c`. The relevant behavior is that a delayed sleeping entity can still be encountered during selection, but the implementation may finalize its delayed dequeue instead of dispatching it immediately.[4]

## Where EEVDF Handles CFS Frictions More Directly

The examples in this section turn the structural comparison into concrete workload stories.[1, 2, 4, 5]
These are analytical reconstructions of the scheduler rules in the primary sources, **not** benchmark measurements. In every case, the same workload is examined first as a CFS limitation story and then as a case where EEVDF gives a more direct mechanism or explanation.[1, 2, 4, 5]

### Scenario 1: Low-latency bursty task versus equal-weight CPU-bound workers

**Workload.** Two CPU-bound workers run continuously at nice 0. A third task at the same nice level wakes periodically, performs a short burst and sleeps again.

**CFS.** The bursty task benefits from sleeping because its `vruntime` advances less than the workers' `vruntime`, so it often re-enters the run queue with less virtual progress and therefore competes strongly for selection.[1] However, classic CFS does not express "same share, but shorter request" explicitly. Wakeup responsiveness is filtered through placement and wakeup granularity and the historical source documents `sysctl_sched_wakeup_granularity` precisely as a threshold that can delay preemption for some workloads.[5]

If the bursty task wakes with a clearly smaller `vruntime`, CFS has a good reason to run it soon. But if the gap to the current runner is small, the next few decisions can still depend on wakeup timing, granularity thresholds and buddy-style cache locality preferences rather than on an explicit "short request first" rule.[5]

**EEVDF.** The same workload can now be described using explicit mechanisms. The bursty task may wake with nonnegative lag because it consumed less service while sleeping, making it eligible; if it also requests a short slice, its virtual deadline becomes earlier than the workers' deadlines at comparable `vruntime`.[2, 4] The improvement is therefore not a vague "interactive boost." It is the combination of:

- lag preserving the fact that the task is owed service;[2, 4]
- eligibility preventing overserved tasks from dominating the competition;[4]
- requested slice turning short service windows into earlier deadlines.[2, 4]

**Strongest defensible claim.** EEVDF gives a clearer and more direct representation of why the bursty task should run early; it does not justify a hard wall-clock dispatch bound from the available sources.[2, 4]

### Scenario 2: Audio or VoIP bursts versus encoder threads

**Workload.** One fair-class audio or VoIP thread performs short periodic bursts. Several encoder threads of equal weight run continuously.

**CFS.** Under classic CFS, the fair-share story is easy: equal nice values imply equal weights, so the audio task receives the same long-run share as any encoder thread.[3, 1] The harder part is urgency. CFS has no documented per-task mechanism in its original design for saying "preserve equal share, but prefer this task's short burst over another task's long burst." Instead, urgency emerges indirectly from `vruntime` position and wakeup heuristics.[1, 5]

**EEVDF.** The EEVDF documentation explicitly introduces such a mechanism: a latency-sensitive application can request a shorter slice via `sched_setattr()`.[2] The audio thread can therefore keep the same weight as the encoders while asking for a shorter service window. In EEVDF terms, that changes the thread's virtual deadline, not its long-run weight-derived entitlement.[2, 4]

In the CFS version of the story, an audio thread "does better" mostly because it slept and therefore reappears with favorable `vruntime`. In the EEVDF version, the two ideas separate cleanly: sleep can preserve positive lag, while a short requested slice can make the audio burst the more urgent eligible request.[2, 4]

**Why this matters.** This workload captures the most important conceptual difference between the two schedulers:

- in CFS, CPU share and urgency are coupled more tightly because both are represented mostly through weighted `vruntime` plus global tunables;[1, 5]
- in EEVDF, CPU share still comes from weight, but urgency among eligible tasks also depends on the requested slice.[2, 4]

The clearest defensible version of the claim is that EEVDF gives VoIP-style bursts a more direct way to express urgency.

### Scenario 3: Frame-based game loop with audio, physics and render work

**Workload.** A frame-based application has equal-weight audio, physics and render tasks. Audio runs in short periodic bursts. Render and physics execute longer bursts tied to the frame cadence.

**CFS.** Classic CFS can favor a task that has fallen behind in `vruntime`, but the original design does not provide a direct way to say that one equal-weight task should repeatedly ask for short requests while another equal-weight task should prefer longer quanta for throughput.[1] Historically, the scheduler instead used run-queue placement, global latency tunables and minimum granularity to balance response time against context-switch cost.[5]

Equal-weight tasks can still experience meaningfully different short-term response depending on where in the current slice they wake and how close they are in `vruntime` to the current runner. That is enough to explain why frame-critical audio or input work may feel harder to reason about under classic CFS, even when long-run fairness is behaving as intended.[5]

**EEVDF.** EEVDF gives the same workload a more explicit ordering story. If the audio task requests a short slice, then once it is eligible it receives an earlier virtual deadline than render or physics tasks that request longer slices.[2, 4] The mechanism is again concrete:

- equal weights preserve equal long-run share;[3, 2]
- a shorter requested slice shrinks `calc_delta_fair(se->slice, se)`;[4]
- earlier deadline ordering can move the short audio request ahead of longer eligible requests.[2, 4]

**Caveat.** The sources do not justify a proof that every frame deadline will be met. They support a narrower statement: EEVDF gives fair-class tasks a direct way to express short service windows and the scheduler orders eligible requests accordingly.[2, 4]

### Scenario 4: Overserved sleeper and wakeup behavior

**Workload.** A task runs heavily, becomes overserved, then sleeps briefly. Other equal-weight tasks continue running.

**CFS.** The original CFS story about sleepers is largely heuristic and placement-based. A waking task is reinserted near current run-queue progress, but the documentation and historical source do not express this as explicit service debt carried through sleep.[1, 5]

**EEVDF.** The current EEVDF documentation addresses the exact pathology directly. A negative-lag sleeper can remain marked for deferred dequeue, so a short sleep does not erase the fact that the task already consumed more than its fair share. As virtual time advances, the overservice debt decays and long sleeps eventually forgive it.[2]

**Why this is better specified.** The improvement here is not "EEVDF is nicer to sleepers." It is the opposite: EEVDF is stricter about overserved sleepers and more explicit about why. This is one of the cleanest examples where EEVDF replaces an older folklore explanation with an explicit algebraic quantity: lag.[2, 4]

### Scenario 5: Mixed interactive and batch workloads

**Workload.** One user-facing interactive task and one or more batch workers share the CPU under the fair scheduler.

**CFS.** Classic CFS already had a strong argument in this workload: tasks that block for I/O tend to accumulate less `vruntime`, so they often regain the CPU quickly when they wake.[1] That is one reason CFS worked well for desktop-style interactivity. But the implementation still balanced that behavior against wakeup granularity and minimum granularity so that one stream of wakeups did not cause pathological over-preemption.[5]

**EEVDF.** EEVDF preserves the same high-level intuition but makes it more explicit. An interactive task that has received less than its fair share wakes with nonnegative lag, which makes it eligible; if it also requests a short slice, it can obtain an earlier deadline than longer-slice batch tasks that are also eligible.[2, 4]

**Balanced conclusion.** This is not a proof that every mixed interactive workload improves under EEVDF. The scheduler does, however, expose the mechanisms for such improvement more directly: eligibility captures owed service and requested slice captures short service windows.[2, 4]

These examples are strongest when they illustrate relative urgency and ordering logic, not when they try to predict a fixed frame time or exact wakeup-to-dispatch latency.[1, 2, 4, 5]

The next section states the same comparison in a more formal register, focusing on what can and cannot be proved from the primary sources.[1, 2, 4, 5]

## Mathematical Analysis

### Formal setup and notation

Consider a set of runnable entities indexed by `i`.

- `w_i > 0`: weight of entity `i`;
- `s_i(t)`: service actually received by entity `i` up to time `t`;
- `v_i(t)`: virtual runtime of entity `i`;
- `V(t)`: weighted average virtual runtime of the runnable set;
- `lag_i(t)`: service deficit or surplus of entity `i` relative to fair service.[4]

The current `fair.c` comment states the central identity:

```text
lag_i = S - s_i = w_i * (V - v_i)
```

where `S` is the service the entity ought to have received under the weighted fair idealization.[4]

### CFS mathematics

Let:

- `w_i` be the weight of task `i`;
- `s_i(t)` be the wall-clock CPU service task `i` has received by time `t`;
- `v_i(t)` be the task's virtual runtime;
- `K = NICE_0_LOAD` be the normalization constant used by `calc_delta_fair()`.[5]

The CFS accounting rule is:

```text
dv_i = (K / w_i) ds_i
```

This equation is a derivation from the historical weight-normalized accounting code, not a direct verbatim source sentence.[5] It expresses the central CFS mathematical idea: heavier tasks accumulate virtual time more slowly for the same real service.

#### Formal derivation

If always-runnable tasks are kept close in `vruntime`, which is the policy CFS implements by always preferring the smallest-`vruntime` entity,[1, 5] then:

```text
v_i(t) ≈ v_j(t)
```

and integrating the accounting rule yields:

```text
s_i(t) / w_i ≈ s_j(t) / w_j
```

Therefore:

```text
s_i(t) : s_j(t) ≈ w_i : w_j
```

That is the proportional-fairness interpretation of CFS under its idealized processor-sharing model.[1, 5]

#### Proof sketch

1. CFS always prefers the runnable entity with the smallest `vruntime`.[1, 5]
2. Running increases `vruntime` at a rate inversely proportional to task weight.[5]
3. A lighter task therefore reaches the "I am no longer the leftmost task" frontier sooner for the same real service.
4. A heavier task reaches that frontier later and can accumulate more real service before relinquishing its advantage.
5. Keeping tasks near the same `vruntime` therefore approximates equalizing `s_i / w_i`.[1, 5]

#### Intuition

CFS approximates ideal processor sharing by turning service imbalance into virtual-time skew. Tasks behind fair service remain to the left in the tree; tasks ahead drift to the right.[1]

#### Fairness-error caveat

The primary sources used here do not state a compact formal fairness-error bound for CFS analogous to the EEVDF lag bound. The implementation's slice and granularity mechanics imply bounded deviations shaped by scheduling period and granularity, but this article does not elevate that intuition into a formal theorem because the available primary sources here do not present it in that form.[1, 5]

#### Implementation implication

This mathematics is not separate from the code; it is the code's policy interpretation. The leftmost-tree rule prefers the task with the smallest `v_i` and `calc_delta_fair()` decides how quickly a given amount of real service changes that `v_i`. Together, those two mechanisms are why heavier tasks can accumulate more wall-clock CPU service before they lose the scheduler's preference.[1, 5]

### EEVDF mathematics

#### Teaching Order

The EEVDF mathematics is easiest to read in this order: proportional sharing, then lag, then eligibility, then deadline ordering and only after that bounded error and implementation caveats. That ordering matches the kernel documentation's own emphasis that EEVDF keeps weighted fairness while making urgency explicit.[2, 4]

In the idealized model, long-run CPU time trends toward proportional allocation by weight:

```text
time_share_i ∝ w_i
```

and the reason is still weighted virtual time:

```text
dv_i = (K / w_i) ds_i
```

with `K = NICE_0_LOAD` as the normalization constant used by the scheduler's weighted accounting code.[4, 5]

This preserves the same weighted-service foundation across the CFS and EEVDF presentations; what changes is how fairness debt and urgency are represented on top of that foundation.[2, 4, 5]

#### Interpretation of lag

The next step is to interpret lag directly. The algebra gives the correct interpretation immediately:

- if `V - v_i > 0`, then `lag_i > 0`, so entity `i` is behind its fair share;
- if `V - v_i < 0`, then `lag_i < 0`, so entity `i` is ahead of its fair share.[4]

This is not merely intuition; it is the formal meaning of lag in the current source comment.[4]

A task with positive lag is owed service relative to the weighted average, while a task with negative lag has already consumed more than its fair share.[4]

#### Eligibility as a formal guarantee

The source then derives:

```text
lag_i >= 0  ->  V >= v_i
```

So eligibility is equivalent to "the task has not yet received more than its fair service according to the weighted average virtual timeline."[4]

This yields a clean formal guarantee:

> **Formal guarantee.** A task with negative lag is not eligible under the EEVDF model and a task with nonnegative lag is eligible.[4]

This is stronger than the older CFS intuition that a waking task "looks interactive." It is an algebraic statement.

#### Ordering once fairness is enforced

After deciding who is owed service, the next question is how to choose among those tasks. The strongest latency-related claim supported by the available sources is **not** a hard wall-clock bound. Instead it is a relative ordering statement:

> **Formal guarantee.** Among eligible entities, the scheduler orders execution by earliest virtual deadline.[2, 4]

This follows from the documentation's policy statement and the code-level comparison function `entity_before()` on `deadline`.[2, 4]

This is the mathematical version of the fairness-versus-urgency distinction:

- lag answers who is owed service;[4]
- deadline answers which owed task has the shortest current request.[2, 4]

#### Zero-sum intuition

Because `V` is the weighted average virtual runtime, the weighted lags sum to zero over the runnable set. Intuitively, one task being owed service means some other task must be ahead. This is why EEVDF can use eligibility as a fairness gate rather than as a heuristic hint.[4]

#### Proof sketch of the relative latency-ordering claim

Let `E(t)` be the set of eligible entities at decision time `t`. For each `i in E(t)`, define its current virtual deadline `d_i(t)`.

1. By the documented EEVDF policy, only eligible tasks compete.[2]
2. By `entity_before()`, ordering among competing entities is by increasing `deadline`.[4]
3. Therefore, if entity `i` is eligible at time `t` and no new eligible entity with `d_j(t) < d_i(t)` arrives before the next selection, then no eligible entity with a later deadline can be selected ahead of `i`.[4]

This is the strongest clean proof sketch available from the sources used here. It is a *relative latency-ordering* argument, not a hard time bound.

#### Bounded fairness error and what it means

The current source also states a steady-state lag bound:

```text
-r_max < lag < max(r_max, q)
```

This is the strongest fairness-error bound stated directly in the source material used here.[4] It supports the claim that service error remains bounded in steady state; it does **not** support a universal bound on wakeup-to-dispatch wall-clock delay under all system conditions.[2, 4]

#### Implementation implication

In code terms, the mathematics explains why EEVDF needs both `entity_eligible()` and deadline comparison. Eligibility answers the fairness question by testing whether the task is still owed service. Deadline comparison answers the urgency question only after that fairness filter has been applied. A scheduler that skipped the eligibility test could give low-latency preference to tasks that were already ahead of their fair share, which would contradict the source's own lag formulation.[2, 4]

#### Implementation caveats

The proof sketch above depends on a single-decision view of the ready set and therefore has implementation caveats:

- SMP migration and load balancing can change where an entity competes;[4]
- new arrivals can insert earlier eligible deadlines after time `t`;[4]
- current Linux still contains wakeup-path and buddy-related policy logic around preemption, so the mathematical model is central but not literally the only source of scheduling behavior in the implementation.[4]

#### What cannot be claimed from the available sources

The following claim would be an overstatement and is therefore rejected:

> "Linux EEVDF guarantees a fixed wall-clock latency bound for fair-class tasks that CFS could not provide."

The sources support a narrower and stronger academically respectable statement:

- EEVDF gives fair-class tasks explicit latency control through requested slice;[2]
- eligibility and earliest-deadline selection provide a formal relative-ordering guarantee among eligible tasks;[2, 4]
- steady-state fairness error is bounded in the source's documented lag sense.[4]

## Kernel Walkthroughs

This section ties the historical CFS story and the current EEVDF story into a concrete control-flow view.

### CFS control flow: historical pre-EEVDF path

The historical CFS path can be read as:

1. **update runtime** with `calc_delta_fair()` and `update_curr()`;[5]
2. **place waking entities** near `min_vruntime`;[1, 5]
3. **enqueue into `tasks_timeline`** ordered by `vruntime`;[5]
4. **pick the leftmost entity**;[5]
5. **apply wakeup granularity and related heuristics** in the preemption path.[5]

The important point is not that CFS lacked mathematical structure. It is that its mathematical structure centered on one primary timeline, `vruntime` and then layered practical wakeup policy around it.[1, 5]

### EEVDF control flow: current path

The current fair scheduler can be read as:

1. **update runtime** with `calc_delta_fair()` and update the running entity's `vruntime`;[4]
2. **refresh lag** through `avg_vruntime()` and `update_entity_lag()`;[4]
3. **maintain eligibility state** using the `V >= v_i` test;[4]
4. **refresh deadlines** with `update_deadline()` from `vruntime` plus weighted slice;[4]
5. **select by earliest virtual deadline among eligible entities**;[2, 4]
6. **still apply implementation-specific wakeup and buddy logic where present**.[4]

The key architectural difference is therefore representational: CFS mainly exposed *progress*; EEVDF exposes *progress*, *debt* and *service-window urgency* separately.[1, 2, 4, 5]

### Code snippets and interpretation

**Shared weighted accounting**

`kernel/sched/fair.c` in Linux v6.5 (lines 3323-3331)  
<https://github.com/torvalds/linux/blob/v6.5/kernel/sched/fair.c#L3323-L3331>

```c
static inline u64 calc_delta_fair(u64 delta, struct sched_entity *se)
{
	if (unlikely(se->load.weight != NICE_0_LOAD))
		delta = __calc_delta(delta, NICE_0_LOAD, &se->load);
	return delta;
}
```

This survives across both eras because both CFS and EEVDF need weight-normalized service accounting.[4, 5]

**Historical CFS leftmost pick**

`kernel/sched/fair.c` in Linux v6.5 (lines 3243-3253)  
<https://github.com/torvalds/linux/blob/v6.5/kernel/sched/fair.c#L3243-L3253>

```c
struct sched_entity *__pick_first_entity(struct cfs_rq *cfs_rq)
{
	struct rb_node *left = rb_first_cached(&cfs_rq->tasks_timeline);
	if (!left)
		return NULL;
	return __node_2_se(left);
}
```

This is the literal implementation of the classic "pick the smallest `vruntime`" story.[5]

**Current EEVDF eligibility**

`kernel/sched/fair.c` in current Linux (lines 3561-3565)  
<https://github.com/torvalds/linux/blob/master/kernel/sched/fair.c#L3561-L3565>

```c
int entity_eligible(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	return vruntime_eligible(cfs_rq, se->vruntime);
}
```

This is the code form of the formal eligibility condition derived in the source comment.[8]

**Current EEVDF deadline update**

`kernel/sched/fair.c` in current Linux (lines 3787-3814)  
<https://github.com/torvalds/linux/blob/master/kernel/sched/fair.c#L3787-L3814>

```c
static bool update_deadline(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	if (vruntime_cmp(se->vruntime, "<", se->deadline))
		return false;

	if (!se->custom_slice)
		se->slice = sysctl_sched_base_slice;
	se->deadline = se->vruntime + calc_delta_fair(se->slice, se);
	return true;
}
```

This is the code-level bridge from requested slice to virtual deadline ordering.[9]

### Implementation balance

The article's implementation claim is therefore balanced:

- **CFS** is best understood through a single weighted-progress axis plus heuristics for wakeup behavior and granularity;[1, 5]
- **EEVDF** is best understood through a weighted-progress axis, an explicit service-debt metric and deadline ordering among eligible tasks, while still retaining practical policy logic in the implementation.[2, 4]

## References
- [1] Linux kernel documentation, "CFS Scheduler."  
  <https://docs.kernel.org/scheduler/sched-design-CFS.html>
- [2] Linux kernel documentation, "EEVDF Scheduler."  
  <https://docs.kernel.org/scheduler/sched-eevdf.html>
- [3] Linux kernel documentation, "Scheduler Nice Design."  
  <https://docs.kernel.org/scheduler/sched-nice-design.html>
- [4] Linux kernel source, current `kernel/sched/fair.c`.  
  <https://raw.githubusercontent.com/torvalds/linux/master/kernel/sched/fair.c>
- [5] Linux v6.5 source, historical pre-EEVDF `kernel/sched/fair.c`.  
  <https://raw.githubusercontent.com/torvalds/linux/v6.5/kernel/sched/fair.c>
- [6] Linux kernel documentation, "Deadline Task Scheduling."  
  <https://docs.kernel.org/scheduler/sched-deadline.html>
- [7] Linux kernel source, current `kernel/sched/fair.c`, lag update machinery including `update_entity_lag()`.  
  <https://github.com/torvalds/linux/blob/master/kernel/sched/fair.c#L3487>
- [8] Linux kernel source, current `kernel/sched/fair.c`, eligibility path including `entity_eligible()`.  
  <https://github.com/torvalds/linux/blob/master/kernel/sched/fair.c#L3561>
- [9] Linux kernel source and historical source, deadline and slice/accounting paths.  
  <https://github.com/torvalds/linux/blob/master/kernel/sched/fair.c#L3787>  
  <https://github.com/torvalds/linux/blob/v6.5/kernel/sched/fair.c#L3451>
- [10] Jonathan Corbet, "An EEVDF CPU scheduler for Linux." Secondary background reading.  
  <https://lwn.net/Articles/925371/>
- [11] Jonathan Corbet, "EEVDF scheduling comes to Linux." Secondary background reading.  
  <https://lwn.net/Articles/969062/>

## Qualifications and Scope Limits

- Exact scenario timelines and millisecond outcomes are not treated here as factual measurements because no benchmark traces or direct experiments are provided.
- Current `fair.c` is already EEVDF-era code, so historical CFS implementation details are sourced from Linux v6.5 rather than projected backward from current code.
- The article makes no claim of a hard wall-clock latency guarantee for EEVDF under `SCHED_OTHER`; the strongest supported claim is relative ordering among eligible tasks plus a documented steady-state lag bound.
- The precise "delayed sleeper becomes positive-lag" behavior is implementation-sensitive because it depends on current delayed-dequeue feature handling in `fair.c`, not just on the abstract EEVDF documentation.
