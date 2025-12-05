Linux Schedulers: Fairness, CFS and EEVDF

PART 1: CFS (Completely Fair Scheduler)

1.1 Overview and Core Concept

Ideal Multitasking Model
CFS models each task as if it runs simultaneously with all others at equal speed (1/nr_running of CPU speed).

Key Concepts:
* Virtual Runtime (vruntime): Measures how much CPU time a task has received, accounting for its priority/weight
* Red-Black Tree: Time-ordered rbtree where tasks are sorted by vruntime; leftmost node has lowest vruntime
* min_vruntime: Monotonically increasing value tracking the smallest vruntime among all tasks in the runqueue

1.2 Fairness Guarantees and Proof

Theoretical Guarantee
Each task should receive CPU time proportional to its weight:

```
time_share = (weight_i / Σweights) × total_time
```

Proof through vruntime progression: All tasks' vruntime converges towards the same value over time.

Weight-based Priority
* Nice values (-20 to +19) mapped to weights through prio_to_weight array
* Weight at nice 0 = 1024 (baseline)
* Each nice level represents ~1.25× weight difference
* Virtual runtime calculation: `Δvruntime = Δreal_time × (NICE_0_LOAD / task_weight)`

1.3 Time Slice Calculation: nr_running, sched_period, min_granularity, and vslice

Key Parameters
* `sched_latency_ns` (default 6ms): target period for all tasks to run once
* `sched_min_granularity_ns` (default 0.75ms): minimum time slice to avoid excessive context switches

Vslice Calculation Details
* Physical time slice for a task: `slice_i = sched_period × (weight_i / Σweights)`
* Virtual time slice (vslice): `vslice_i = slice_i × (NICE_0_LOAD / weight_i)`
* The vslice represents how much vruntime a task accumulates when running for its allocated time slice
* Ensures high-priority tasks (higher weight) accumulate vruntime slower for the same physical time
* Bounded by min_granularity to prevent starvation

Scheduling Period Calculation

```
if (nr_running <= sched_latency / min_granularity):
    base_sched_period = sched_latency
else:
    base_sched_period = nr_running × min_granularity
```

Example with 10 Tasks (Different Weights)

Setup:
* Task A-H: weight=1024 (nice 0) - 8 tasks
* Task I: weight=820 (nice 1)
* Task J: weight=1277 (nice -1)
* Total weight = 8×1024 + 820 + 1277 = 10,289

Step 1: Base scheduling period

```
10 tasks > (6ms / 0.75ms = 8): period determined by min_granularity
base_sched_period = 10 × 0.75ms = 7.5ms
```

Step 2: Weighted physical slices (before clamping)

```
Task A-H each: 7.5ms × (1024/10289) = 0.746ms
Task I: 7.5ms × (820/10289) = 0.597ms
Task J: 7.5ms × (1277/10289) = 0.931ms
```

Step 3: Min_granularity clamping

```
All physical slices ≥ 0.75ms? NO
Tasks A-H: 0.746ms < 0.75ms → clamped to 0.75ms
Task I: 0.597ms < 0.75ms → clamped to 0.75ms
Task J: 0.931ms ≥ 0.75ms → remains 0.931ms
```

Step 4: Effective scheduling period

```
effective_sched_period = 8×0.75ms + 0.75ms + 0.931ms = 6.75ms + 0.931ms = 7.681ms
```

This is slightly longer than the base period due to min_granularity enforcement.

1.4 Scheduling Decision Example: 10 Consecutive Decisions with 4 Tasks

Scenario Setup
* Task A: weight=2048 (nice -3), CPU-bound, starts at vruntime=0
* Task B: weight=820 (nice 1), CPU-bound, starts at vruntime=0
* Task C: weight=1277 (nice -1), CPU-bound, starts at vruntime=0
* Task D: weight=1024 (nice 0), interactive (wakes every 8ms, runs 1ms), starts at vruntime=0
* Total running weight initially = 4145 (A, B, C)
* sched_period = 6ms (< 8 tasks)

Timeslices when 3 CPU-bound tasks running:

```
Total weight = 4145
Task A: 6ms × (2048/4145) = 2.965ms
Task B: 6ms × (820/4145) = 1.187ms
Task C: 6ms × (1277/4145) = 1.848ms
```

10 Consecutive Scheduling Decisions

Decision 1 (t=0ms):
* Pick: Task A (leftmost, vruntime=0)
* Runs: 2.965ms
* Update: vruntime_A = 2.965ms × (1024/2048) = 1.483ms

Decision 2 (t=2.965ms):
* Vruntimes: A=1.483ms, B=0, C=0
* Pick: Task B (leftmost, vruntime=0)
* Runs: 1.187ms
* Update: vruntime_B = 1.187ms × (1024/820) = 1.483ms

Decision 3 (t=4.152ms):
* Vruntimes: A=1.483ms, B=1.483ms, C=0
* Pick: Task C (leftmost, vruntime=0)
* Runs: 1.848ms
* Update: vruntime_C = 1.848ms × (1024/1277) = 1.483ms

Decision 4 (t=6.000ms):
* Vruntimes: A=1.483ms, B=1.483ms, C=1.483ms (all equal - pick first in tree, Task A)
* Pick: Task A
* Runs: 2.965ms
* Update: vruntime_A = 1.483 + 1.483 = 2.966ms

At t=8.000ms: Task D wakes up (before Decision 5):

```
Task D was sleeping, vruntime_D was frozen at 0ms
On wakeup, vruntime_D adjusted: vruntime_D = max(0, min_vruntime - 6ms) = max(0, 1.483 - 6) = 0ms
Task D now has lowest vruntime (0ms), will be scheduled next
min_vruntime at t=8ms = 1.483ms
Total weight now = 5169 (A, B, C, D all runnable)
```

Decision 5 (t=8.965ms - Task A's slice completes):
* Vruntimes: A=2.966ms, B=1.483ms, C=1.483ms, D=0ms
* Pick: Task D (leftmost, vruntime=0ms)
* Runs: 1ms
* Update: vruntime_D = 0 + 1ms × (1024/1024) = 1ms
* After running, Task D sleeps again (remains at vruntime=1ms, frozen)

Decision 6 (t=9.965ms):
* Vruntimes: A=2.966ms, B=1.483ms, C=1.483ms (D is sleeping)
* Total weight back to 4145
* Pick: Task B (leftmost among running)
* Runs: 1.187ms
* Update: vruntime_B = 1.483 + 1.483 = 2.966ms

Decision 7 (t=11.152ms):
* Vruntimes: A=2.966ms, B=2.966ms, C=1.483ms
* Pick: Task C (leftmost)
* Runs: 1.848ms
* Update: vruntime_C = 1.483 + 1.483 = 2.966ms

Decision 8 (t=13.000ms):
* Vruntimes: A=2.966ms, B=2.966ms, C=2.966ms
* Pick: Task A (first in tree)
* Runs: 2.965ms
* Update: vruntime_A = 2.966 + 1.483 = 4.449ms

At t=16.000ms: Task D wakes up again:

```
vruntime_D was frozen at 1ms
Current min_vruntime = 2.966ms
Adjusted: vruntime_D = max(1, 2.966 - 6) = 1ms (no change, already within range)
Task D has lowest vruntime, will preempt or be scheduled next
```

Decision 9 (t=15.965ms - before D wakes):
* Vruntimes: A=4.449ms, B=2.966ms, C=2.966ms
* Pick: Task B (leftmost among running)
* Runs: starts at t=15.965ms
* At t=16.000ms (0.035ms into B's slice): Task D wakes

Wakeup preemption check:

```
D.vruntime (1ms) < B.vruntime (2.966ms) - wakeup_gran(~1ms)?
1 < 2.966-1 = 1.966? YES → Preempt B
B gets preempted after running only 0.035ms
Update: vruntime_B = 2.966 + 0.035 × (1024/820) = 3.010ms
```

Decision 9 (t=16.000ms after preemption):
* Pick: Task D (preempted B)
* Runs: 1ms
* Update: vruntime_D = 1 + 1 = 2ms
* Task D sleeps again

Decision 10 (t=17.000ms):
* Vruntimes: A=4.449ms, B=3.010ms, C=2.966ms (D sleeping at vruntime=2ms)
* Pick: Task C (leftmost)
* Runs: 1.848ms
* Update: vruntime_C = 2.966 + 1.483 = 4.449ms

Observation: Task D (interactive) naturally gets scheduled quickly when it wakes because its vruntime remains low from sleeping. The wakeup preemption mechanism ensures responsive scheduling. Over time, all CPU-bound tasks maintain proportional CPU shares matching their weight ratios.

1.5 Wakeup Handling

Placement in RB-tree
When unblocked, vruntime is adjusted to prevent unfair advantage. Sleeper bonus amount depends on task type:
* Normal tasks (default): 6ms worth of vruntime (sched_latency_ns)
* GENTLE_FAIR_SLEEPERS enabled: 3ms worth of vruntime (sched_latency_ns / 2) - gentler boost to avoid over-rewarding sleepers
* Idle tasks: 0.75ms worth of vruntime (sched_min_granularity_ns) - minimal boost since they're lowest priority

Calculation:

```
new_vruntime = max(old_vruntime, min_vruntime - sleeper_bonus)
```

Prevents tasks from accumulating unfair advantage through sleeping. Places task near front of queue but not necessarily ahead of all running tasks.

Wakeup Preemption Decision
* Check if `woken_task.vruntime < current_task.vruntime - sched_wakeup_granularity`
* `sched_wakeup_granularity_ns` (default ~1ms): threshold to prevent excessive preemption
* If condition met, trigger preemption and mark current task for rescheduling
* May set wakee task as "next buddy" for cache locality

1.6 CFS Heuristics and Their Interactions

1.6.1 Sleeper Fairness
Interactive tasks naturally get priority because they sleep awaiting user input, resulting in relatively low vruntime. No explicit interactivity detection needed.

1.6.2 Buddy Hints (next_buddy, last_buddy, skip_buddy)
Cache affinity optimization: recently scheduled tasks preferred (last_buddy), tasks that just woke another task preferred (next_buddy). Skip_buddy marks tasks to defer.

Interaction with wakeup preemption: In `pick_next_entity()`, buddy hints can override leftmost task if vruntime difference is small. This creates corner cases where latency-sensitive tasks wait despite lower vruntime.

1.6.3 Wakeup Preemption Threshold (wakeup_granularity)
Prevents excessive context switches from eager preemption. Default ~1ms means woken task must have vruntime advantage >1ms to preempt.

Trade-off: Reduces context switch overhead but increases worst-case scheduling latency. A task with slightly lower vruntime may wait 1-2ms if buddy hints also favor current task.

Example 1: Sleeper Fairness + Buddy System Unpredictability

Scenario:
* Audio player (Task A, nice 0) wakes every 10ms, runs 1ms
* CPU-intensive compile (Task B, nice 0)
* Background indexer (Task C, nice 5) wakes every 50ms, runs 5ms

Timeline:

```
t=0ms: Task A wakes with vruntime=0
  Runs 1ms → vruntime_A=1ms, goes to sleep

t=1ms-10ms: B runs
  Advancing vruntime_B to ~9ms

t=10ms: Task A wakes again
  vruntime_A adjusted to max(1, 9-6) = 3ms

t=10ms: Task C also wakes (coincidentally) after 50ms sleep
  vruntime_C adjusted to max(old_value, 9-6) = 3ms
  Both A and C have vruntime=3ms (same)
```

Case 1: C wakes 0.001ms before A

```
C triggers wakeup preemption first at vruntime=3ms
C is set as next_buddy

When A wakes 0.001ms later (also vruntime=3ms):
  A also triggers preemption check
  But C already has next_buddy status
  
  In pick_next_entity(): next_buddy (C) is checked first
    wakeup_preempt_entity(C, leftmost) where both have vruntime=3ms
    vdiff = 3 - 3 = 0 → return -1
    Condition satisfied: -1 < 1, so C gets scheduled
  
  C runs for its full 5ms slice

Result: Audio task A waits 5ms (C's full slice), causing audio glitch
```

Case 2: A wakes 0.001ms before C

```
A triggers preemption, becomes next_buddy, runs 1ms, completes
C wakes, also gets scheduled promptly
No audio glitch
```

Unpredictability: 1μs difference in wakeup timing dramatically changes scheduling order due to next_buddy assignment racing with sleeper fairness.

1.6.4 Combined Heuristic Interactions

Example: Audio Playback + Background Indexer

Setup:
* Audio task (A): weight=1024, wakes every 10ms, needs 1ms CPU
* Compile task (C): weight=1024, CPU-bound
* Indexer task (I): weight=290 (nice 9), CPU-bound
* Total weight (all running) = 2338

When A and C are running together (weight=2048):
* sched_period = 6ms
* A gets 3ms slice, C gets 3ms slice
* A wakes, runs 1ms, sleeps
* C runs its slice
* Pattern: A has low latency, gets scheduled quickly

At t=100ms: Indexer I starts

Now total weight = 2338 (A, C, I)
* A: slice = 6ms × (1024/2338) = 2.63ms
* C: slice = 6ms × (1024/2338) = 2.63ms  
* I: slice = 6ms × (290/2338) = 0.74ms → clamped to 0.75ms

Timeline showing race condition:

```
t=110.000ms: Audio A and Indexer I both wake from sleep
Both have low vruntime due to sleeper bonus
Suppose both get vruntime = 48ms (similar after adjustment)
Compile C currently running at vruntime_C = 50ms

Case 1: A wakes 0.001ms before I (t=110.000ms vs t=110.001ms)
  - A evaluated first in wakeup path
  - A.vruntime (48ms) < C.vruntime (50ms) - wakeup_gran(1ms)? 
  - 48 < 49? YES → A preempts C immediately
  - A runs 1ms, sleeps
  - I wakes at t=110.001ms while A is running
  - I.vruntime (48ms) vs A.vruntime (48ms + ~0.001ms)
  - Very close, but A now has next_buddy status (just woke)
  - Buddy hint favors A to complete
  - I waits for A to finish (~1ms more), then scheduled
  - Result: A gets <0.1ms latency, perfect

Case 2: I wakes 0.001ms before A (t=110.000ms vs t=110.001ms)
  - I evaluated first in wakeup path
  - I.vruntime (48ms) < C.vruntime (50ms) - wakeup_gran(1ms)?
  - 48 < 49? YES → I preempts C
  - I starts running at t=110.000ms
  - A wakes at t=110.001ms
  - A.vruntime (48ms) vs I.vruntime (48ms + ~0.001ms)
  - Close vruntimes, but I now has next_buddy status
  - Buddy hint check in pick_next_entity():
    - wakeup_preempt_entity(I_next_buddy, A_leftmost)
    - Vruntime difference < wakeup_gran(1ms)
    - Buddy preference activates
  - I continues running its 0.75ms slice
  - A waits ~0.75ms for I to complete
  - Result: A experiences ~1ms latency (missed <0.5ms target)
```

Heuristic interactions:
1. Wakeup order (microseconds apart) determines who gets next_buddy status
2. Next_buddy + wakeup_gran threshold (1ms) allows I to block A despite identical vruntimes
3. Unpredictable: 0.001ms wake time difference causes order-of-magnitude latency variation (0.1ms vs 1ms)

Why problematic: Audio glitches when I happens to wake first. User experiences random audio dropouts. No deterministic guarantee.

1.6.5 Timing Sensitivity

Previous example shows microsecond-level wake ordering matters. Buddy hints + wakeup threshold create race conditions. Makes latency guarantees impossible.

1.7 CFS Drawbacks and Failure Cases

1.7.1 No Latency Expression

Problem
In CFS, achieving <5ms latency requires high priority (low nice), but this gives >5% CPU share (unfair to other tasks). Cannot express "I need quick response but don't need much total CPU time".

Specific Example: VoIP Application

Scenario:
* VoIP codec processes 20ms audio chunks, needs to run every 20ms with <5ms jitter
* VoIP thread: wakes every 20ms, processes for 2ms (10% CPU)
* Competing workload: Video encoding (8 threads, nice 0, CPU-bound)
* All tasks nice 0

Setup for specific scheduling example:
* 1 VoIP thread (V): weight=1024, wakes every 20ms, runs 2ms
* 8 Encoder threads (E1-E8): weight=1024 each, CPU-bound
* Total weight = 9×1024 = 9216
* sched_latency = 6ms, but 9 tasks > 8, so period = 9×0.75ms = 6.75ms
* Each task gets ~0.75ms slice

Scheduling timeline showing latency problem:

```
t=0ms: VoIP wakes
  vruntime_V = 0 (sleeper bonus)
  All E threads have vruntime = 50ms
  VoIP scheduled immediately (lowest vruntime)
  Runs 2ms, vruntime_V = 2ms, sleeps

t=2ms-20ms: E1-E8 run in rotation

t=20ms: VoIP wakes again
  vruntime_V still = 2ms (frozen during sleep)
  Current min_vruntime = 64ms (E threads advanced)
  VoIP vruntime adjusted: max(2, 64-6) = 58ms
  E threads have vruntimes ranging from 62-65ms
  VoIP has vruntime=58ms, should be scheduled soon
  
  BUT: If E3 just started its 0.75ms slice at t=20ms with vruntime=62ms:
    - Wakeup preemption check: 58 < 62-1? → 58 < 61? → YES, should preempt
    - However, if E3 has last_buddy status (just scheduled), wakeup_gran threshold may defer preemption
    - VoIP waits for E3's remaining slice (~0.5-0.7ms)
  
  Then E4 (vruntime=62.5ms) is next in tree
    - VoIP still has vruntime=58ms but if E4 just became next_buddy from another wake event:
    - E4 gets preference, runs 0.75ms
  
  VoIP accumulates 1.5-2ms of delays before finally being scheduled

Over many iterations, VoIP experiences 8-15ms scheduling latency spikes when multiple E threads cluster around similar vruntimes.
```

Nice 0 for VoIP result:
* Gets ~11% CPU (1024/9216 = correct share for its weight)
* But scheduling latency is 8-15ms during periods when encoder threads dominate
* Violates <5ms jitter requirement

Nice -5 for VoIP (weight=3277):
* Gets ~35% CPU (3277/20469 total weight = excessive!)
* Scheduling latency <3ms (high priority ensures prompt scheduling)
* Achieves latency target but wastes 25% of CPU

Cannot express desired behavior: Want 10% CPU share with <5ms latency - CFS cannot provide this combination. Must choose between fair CPU share (high latency) or low latency (unfair CPU share).

1.7.2 Lack of Latency Guarantees and Corner Cases

Problem
CFS does not enforce any scheduling deadlines and has many corner cases where a latency-critical task is not scheduled on time.

Setup:
* Task L: Latency-critical, nice 0, wakes every 100ms, runs 5ms (5% CPU), requires <1ms response time
* Tasks A-D: Batch processing, nice 0, CPU-bound
* wakeup_granularity = 1ms (default)

Timeline:

```
t=0-95ms: Tasks A-D run continuously
  Vruntimes advance to ~23.75ms each (each getting 25% of 95ms)

t=95ms: Task L wakes from long sleep (100ms)
  L's vruntime was frozen at 0ms
  min_vruntime = 23.75ms
  Sleeper bonus adjustment: vruntime_L = max(0, 23.75-6) = 17.75ms
  L has vruntime=17.75ms, A-D have ~23.75ms
  Task A currently running (vruntime_A=23.75ms)
  
  Wakeup preemption check:
    L.vruntime (17.75ms) < A.vruntime (23.75ms) - wakeup_gran(1ms)?
    17.75 < 22.75? → YES
    A gets preempted immediately, L scheduled at t=95ms
  
  L runs for 5ms, vruntime_L = 17.75 + 5 = 22.75ms, sleeps

t=195ms: L wakes again
  Current min_vruntime = 48ms (after 100ms of A-D running)
  vruntime_L = max(22.75, 48-6) = 42ms
  
  Corner case occurs: At t=195ms, Task A just started running (t=194.9ms, vruntime_A=42.8ms) and has last_buddy status from being recently scheduled:
    - L has vruntime=42ms (lower than A's 42.8ms)
    - Wakeup preemption check: 42 < 42.8-1? → 42 < 41.8? → NO, normally would not preempt immediately
    
    BUT, in pick_next_entity(), last_buddy check occurs:
      - wakeup_preempt_entity(A_last_buddy, L_leftmost) is called
      - Checks: (42.8 - 42) = 0.8ms > wakeup_gran(1ms)? NO
      - Since 0.8ms ≤ 1ms, wakeup_preempt_entity returns ≤0
      - last_buddy preference activates: A continues running for ~0.7ms more
    
    Then Task B (vruntime=43ms, also within 1ms of L) might be next if it also has buddy status
    
    L accumulates 0.7-1.5ms wait before finally being scheduled
    
    Latency miss: L required <1ms response time, but actually experienced ~1-1.5ms scheduling latency due to last_buddy preference being allowed when vruntime differences are within wakeup_granularity threshold
```

Why corner case: Last_buddy heuristic (cache optimization) conflicts with strict latency requirements. The wakeup_preempt_entity() function in pick_next_entity() allows buddy hints to override fairness when the vruntime difference is within wakeup_granularity (1ms by default), causing delays even when the latency-critical task has lower vruntime.

1.7.3 Timeslice Rigidity

Problem
All tasks with identical weights get similar-length timeslices within a scheduling period. Cannot express "I need CPU soon but only for a short time".

Example Scenario: High-frequency Small Tasks vs. Batch Task

Setup:
* Task B: Batch processing (nice 0, weight=1024, runs continuously)
* Task S: Service task (nice 0, weight=1024, wakes every 10ms, needs 1ms of work, requires <3ms response time)
* Task X: Background task (nice -5, weight=3277, CPU-bound, arrives at t=50ms)
* sched_latency = 6ms

Timeline showing latency issue:

```
t=0-50ms: B and S coexist peacefully
  When both runnable: each gets 3ms timeslices
  S wakes, runs 1ms of its 3ms slice, sleeps
  Response time: <1ms ✓

t=50ms: Task X arrives (nice -5, CPU-bound)
  New configuration: S (weight=1024), B (weight=1024), X (weight=3277)
  Total weight=5325
  Timeslices: S=1.15ms, B=1.15ms, X=3.7ms
  wakeup_granularity = 1ms (default)

t=60ms: S wakes with vruntime_S=10ms
  min_vruntime=54ms
  Adjusted: vruntime_S = max(10, 54-6) = 48ms
  X currently running (vruntime_X=48.7ms) with next_buddy status (just woke and preempted B moments ago)
  
  Wakeup preemption: 48 < 48.7-1? → 48 < 47.7? → NO, normally would not preempt
  
  BUT in pick_next_entity(), wakeup_preempt_entity(X_next_buddy, S_leftmost) is called
    Checks: (48.7 - 48) = 0.7ms > wakeup_gran(1ms)? NO
    Since 0.7ms ≤ 1ms, next_buddy preference activates
    X continues for ~0.8ms more of its 3.7ms slice before yielding to S
  
  S finally scheduled at t=60.8ms
  Scheduling latency: 0.8ms

t=70ms: S wakes again with vruntime_S=49ms
  X has vruntime_X=56ms, B has vruntime_B=50.5ms
  
  If B has last_buddy status and is currently running (vruntime_B=50.5ms):
    Wakeup preemption: 49 < 50.5-1? → 49 < 49.5? → YES, normally would preempt
    BUT last_buddy deferral check: (50.5 - 49) = 1.5ms > wakeup_gran(1ms)? YES
    Since 1.5ms > 1ms, last_buddy preference does NOT activate
    B gets preempted, S runs immediately
  
  If instead B has no buddy status but vruntime_B=49.9ms:
    last_buddy deferral: (49.9 - 49) = 0.9ms ≤ 1ms → B continues ~0.5ms
    Total latency: 0.5-0.8ms, within tolerance

t=80ms: S wakes with vruntime_S=50ms
  B currently running with vruntime_B=50.8ms and last_buddy status
  Buddy deferral check: (50.8 - 50) = 0.8ms ≤ 1ms → B continues ~0.6ms
  Then X (vruntime_X=64ms) is next candidate
  Scheduling latency: 0.6-1.2ms, approaching limit in some cases
```

CFS Failure: Timeslice rigidity means S must accept whatever slice its weight ratio dictates (~1.15ms). Cannot request "schedule me more frequently with shorter slices (0.5ms)" to improve responsiveness without changing nice value (which affects CPU share unfairly). The combination of:
1. Fixed timeslice based on weight ratio
2. Buddy heuristics (next_buddy, last_buddy) interfering with preemption when vruntime differences are within wakeup_granularity (1ms)
3. Inability to express "low latency, normal CPU share"

...causes S to experience unpredictable 0.6-1.2ms latencies, with occasional spikes approaching its 3ms tolerance limit when multiple buddy deferrals compound.

1.7.4 Heuristic Complexity

Sleeper bonus, wakeup preemption threshold, buddy hints all tuned together. These are often-fragile heuristics that CFS depends on. Difficult to reason about combined behavior. Performance varies significantly across workload types. (See detailed examples in section 1.6.3 and 1.6.4)

1.7.5 No Direct Control Over Scheduling Order

Only indirect control through nice values (affects CPU share). Cannot say "schedule this task more frequently but with shorter slices". Led to development of "latency nice" patches as workaround (later incorporated into EEVDF approach).
