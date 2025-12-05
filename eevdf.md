PART 2: EEVDF (Earliest Eligible Virtual Deadline First)

2.1 Overview and Core Concepts

Fundamental Model
EEVDF distributes CPU time equally among runnable tasks by using virtual runtime, lag values, and virtual deadlines. Unlike CFS's heuristic-based approach, EEVDF uses a deadline-driven model with mathematical fairness guarantees.

Key Concepts:
* Virtual Runtime (vruntime): Measures accumulated CPU time scaled by weight
   * Formula: `Δvruntime = Δreal_time × (NICE_0_LOAD / task_weight)`
   * Higher weight tasks accumulate vruntime slower
* Lag: Difference between average vruntime and this task's vruntime
   * Formula: `lag = avg_vruntime - vruntime`
   * Positive lag: task underserved (owed CPU time)
   * Negative lag: task overserved (exceeded fair share)
   * Lag has no units (it's a vruntime difference)
   * Sum of weighted lags is always zero
* Eligibility: Task is eligible if lag ≥ 0
   * Only eligible tasks can be scheduled
   * Enforces fairness by blocking overserved tasks
* Slice (request_slice/vslice - same concept): Virtual time slice in vruntime units
   * Default: `slice = base_slice × (NICE_0_LOAD / weight)` where base_slice = 3ms
   * Custom: `slice = custom_slice × (NICE_0_LOAD / weight)` (set via sched_setattr)
   * Range for custom_slice: 100µs to 100ms
   * Higher weight → SMALLER slice → earlier deadline → lower latency
   * Example: weight=2048 → slice=1.5ms; weight=512 → slice=6ms
* Virtual Deadline (VD): Future vruntime after task runs
   * Formula: `VD = vruntime + slice`
   * Scheduler picks eligible task with earliest VD
   * Represents what vruntime will be after running

Data Structure
Augmented Red-Black Tree indexed by virtual deadline:
* Task's virtual deadline (tree key)
* Task's vruntime
* Task's lag value
* Minimum VD in subtree (O(log n) queries)

Scheduling Algorithm

```
1. Find eligible tasks (lag ≥ 0)
2. Among eligible, find task with earliest VD
3. Run task until slice consumed
4. vruntime increases by slice amount
5. Update lags for all tasks
6. Check for preemption (earlier VD)
```

2.2 Fairness Guarantees and Proof

Theoretical Guarantee
Each task receives CPU time proportional to its weight:

```
time_share_i = (weight_i / Σweights) × total_time
```

Slice calculation:

```
slice_i = base_slice × (NICE_0_LOAD / weight_i)
```

Higher weight → smaller slice → earlier deadline → lower latency

In-Depth Fairness Proof

Step 1: The Goal
Three tasks with different priorities:
* Task A: weight=2048 (57.1% of 3584 total)
* Task B: weight=1024 (28.6% of 3584 total)
* Task C: weight=512 (14.3% of 3584 total)

Over time T, each should get its proportional share.

Step 2: Virtual Runtime Mechanics
When tasks run for Δreal_time:

```
Δvruntime = Δreal_time × (NICE_0_LOAD / weight)
```

Example (10ms wall-clock):
* Task A: Δvruntime = 10 × (1024/2048) = 5ms
* Task B: Δvruntime = 10 × (1024/1024) = 10ms

Higher weight → slower vruntime growth → more wall-clock time

Step 3: Lag and Fairness
Task A runs 10ms wall-clock (vruntime += 5):

```
vruntime_A = 5, vruntime_B = 0, vruntime_C = 0

avg_vruntime = (2048×5 + 1024×0 + 512×0) / 3584 = 2.857

lag_A = 2.857 - 5 = -2.143 (overserved)
lag_B = 2.857 - 0 = +2.857 (underserved)
lag_C = 2.857 - 0 = +2.857 (underserved)
```

Proof positive lag = deserves CPU:
If perfectly fair over 10ms:
* A gets 5.71ms wall-clock → vruntime = 2.857
* B gets 2.86ms wall-clock → vruntime = 2.857
* C gets 1.43ms wall-clock → vruntime = 2.857

All should have vruntime = avg_vruntime!

Actual vs expected:
* A: 5 vs 2.857 → ran too much
* B: 0 vs 2.857 → ran too little (owed +2.857)
* C: 0 vs 2.857 → ran too little (owed +2.857)

Positive lag = task underserved = deserves CPU time

Step 4: Eligibility Enforces Fairness
Only lag ≥ 0 tasks can run. Task A (lag=-2.143) is ineligible.

Task B runs 10ms wall-clock (vruntime += 10):

```
avg_vruntime = (2048×5 + 1024×10 + 512×0) / 3584 = 5.714

lag_A = 5.714 - 5 = +0.714 (eligible again!)
lag_B = 5.714 - 10 = -4.286 (now ineligible)
```

System self-balances: overserved become ineligible, underserved become eligible.

Step 5: Deadlines and Priority
Slices (base_slice=3ms):

```
A: slice = 3 × (1024/2048) = 1.5ms
B: slice = 3 × (1024/1024) = 3ms
C: slice = 3 × (1024/512) = 6ms
```

Higher weight → SMALLER slice → earlier deadline

Current state: A at vruntime=5, C at vruntime=0

```
VD_A = 5 + 1.5 = 6.5
VD_C = 0 + 6 = 6.0
```

C has earlier deadline → runs first.

Higher priority tasks naturally get scheduled sooner due to smaller slices.

Step 6: Complete Fairness Example
t=0: All at vruntime=0
* Deadlines: A=1.5, B=3, C=6
* Pick A (earliest)
* Runs: vruntime_A = 1.5, wall-clock = 3ms
* avg = 0.857, lags: A=-0.643, B=+0.857, C=+0.857

Next: Pick B (eligible, earliest deadline)
* vruntime_B = 3, wall-clock = 3ms
* avg = 1.714, lags: A=+0.214, B=-1.286, C=+1.714

Next: Pick A (eligible again)
* vruntime_A = 3, wall-clock = 3ms
* avg = 2.571, lags: A=-0.429, B=-0.429, C=+2.571

Next: Pick C (only eligible)
* vruntime_C = 6, wall-clock = 3ms
* avg = 3.429

CPU shares over 12ms:
* A: 6ms = 50% (target 57%, converging)
* B: 3ms = 25% (target 29%)
* C: 3ms = 25% (target 14%)

Over longer periods → exact proportions.

Step 7: Mathematical Proof
Claim: Long-term, time_i ∝ weight_i

Proof:
1. Lag bounded by one slice
2. Negative lag → ineligible → forced wait
3. Waiting increases avg, thus increases lag
4. Eventually lag ≥ 0 → eligible
5. All vruntimes converge to avg_vruntime
6. Since Δvruntime = Δtime × (NICE_0_LOAD/weight):
   * Higher weight → slower vruntime growth → more wall-clock time

Result: Proportional CPU allocation guaranteed.

2.3 Why EEVDF Doesn't Use sched_period, min_granularity, wakeup_granularity

CFS Parameters (Eliminated)
* `sched_latency`: 6ms period
* `min_granularity`: 0.75ms minimum
* `wakeup_granularity`: ~1ms threshold

EEVDF Approach
1. No sched_period: Eligibility + deadlines replace fixed period
2. No min_granularity: Custom slices down to 100µs via sched_setattr()
3. No wakeup_granularity: Pure deadline comparison, no threshold

Advantage: Per-task control via sched_setattr() instead of global parameters.

2.4 Scheduling Example: 10 Decisions with 4 Tasks

Setup
* A: weight=2048, slice=1.5ms (3×1024/2048), CPU-bound
* B: weight=820, slice=3.75ms (3×1024/820), CPU-bound
* C: weight=1277, slice=2.41ms (3×1024/1277), CPU-bound
* D: weight=1024, custom slice=1ms, wakes every 8ms

All start: vruntime=0, lag=0

Decision 1 (t=0):
* VDs: A=1.5, B=3.75, C=2.41
* Pick A (earliest)
* Runs to deadline: vruntime_A = 1.5
* avg = (2048×1.5)/(2048+820+1277) = 0.741
* Lags: A=-0.759, B=+0.741, C=+0.741

Decision 2 (t=3ms):
* Eligible: B, C
* VDs: B=3.75, C=2.41
* Pick C
* vruntime_C = 2.41
* avg = (2048×1.5+1277×2.41)/4145 = 1.487
* Lags: A=-0.013, B=+1.487, C=-0.923

Decision 3 (t=6ms):
* Eligible: B
* Pick B
* But D wakes at t=8ms...

At t=8ms: D Wakes
B has run 2ms wall-clock: vruntime_B = 2×(1024/820) = 2.498

Wakeup adjustment:
* avg_before = (2048×1.5+820×2.498+1277×2.41)/4145 = 1.810
* D's preserved lag = 0 (first wake)
* Sum of task weights before D joins = 4145
* Sum of task weights after D joins = 5169
* D's lag on wakeup = 0 × (5169 / 4145) = 0
* vruntime_D = avg - lag = 1.810 - 0 = 1.810

After D joins:
* avg = (2048×1.5+820×2.498+1277×2.41+1024×1.810)/5169 = 1.810

Preemption check:
* B: VD = 2.498 + (3.75-2.498) = 3.75
* D: VD = 1.810 + 1 = 2.810
* 2.810 < 3.75 → D preempts

Decision 4 (t=8ms):
* Pick D
* vruntime_D = 2.810
* D sleeps, lag frozen

Decision 5-10: Pattern Continues
Tasks rotate by eligibility/deadline. D consistently gets <1ms latency due to 1ms slice creating early deadlines.

Key Properties:
* Fairness via lag
* D's low latency via small slice
* No heuristics
* Deterministic

2.5 Wakeup Handling

Sleeping with Positive Lag
* Lag preserved
* Task removed from rbtree immediately
* NOT marked for deferred dequeue
* On wake: vruntime adjusted using formula: lag_on_wakeup = lag_when_slept × (Sum(task-weights) + task_weight) / Sum(task-weights)
* Then: vruntime = avg - lag_on_wakeup
* Immediately eligible

Sleeping with Negative Lag
* Marked "deferred dequeue"
* Stays in rbtree (ineligible)
* As avg increases, lag increases
* When lag ≥ 0: actually dequeued
* Prevents "sleep gaming"

Negative Lag Decay Process
Task sleeps with lag=-2:

```
t=0: vruntime=5, avg=3, lag=-2 (deferred dequeue)
t=10: Other tasks run, avg=13
      lag = 13-5 = +8 (now positive!)
      Task dequeued from rbtree
```

Long sleep → automatic forgiveness. Short sleep → penalty preserved.

Wakeup Vruntime Calculation
When task wakes, we use the lag that was computed when the task slept in the above formula to determine its new lag on wakeup:

```
lag_on_wakeup = lag_when_slept × (Sum(task-weights) + task_weight) / Sum(task-weights)
vruntime_new = avg_vruntime - lag_on_wakeup
```

This adjustment preserves the task's relative position when rejoining (since adding the task changes avg).

Preemption on Wakeup

```
if woken_VD < current_VD:
    preempt immediately
```

No threshold, no buddy system, pure deadline comparison.

2.6 Heuristic Reduction

CFS Heuristics (Eliminated)
1. Sleeper fairness bonus → lag-based eligibility
2. Buddy system (next/last/skip) → deadline ordering
3. Wakeup granularity threshold → deadline comparison
4. Min granularity clamping → explicit slice requests

Complexity Comparison

```
Aspect              CFS                 EEVDF
Decision factors    5+                  2 (lag, deadline)
Global parameters   4                   0
Timing sensitivity  High                None
Determinism         Low                 High
```

2.7 EEVDF Performance on CFS Failures

Example 1: Audio + Indexer Race
CFS: Microsecond wake order matters, causes audio glitches.

EEVDF Setup:
* Audio: weight=1024, custom slice=1ms
* Compile: weight=1024, slice=3ms
* Indexer: weight=290, slice=10.6ms (3×1024/290)

At t=10ms, Audio and Compile wake 0.001ms apart:

Audio wakes first at t=10.000ms:
```
Current avg_vruntime ≈ 5ms (Compile has been running)
Audio's preserved lag from when it slept ≈ 5
Sum of weights before Audio joins = 1314 (Compile + Indexer)
Sum of weights after Audio joins = 2338
Audio's lag on wakeup = 5 × (2338 / 1314) ≈ 8.9
vruntime_Audio = 5 - 8.9 = -3.9
VD_Audio = -3.9 + 1 = -2.9
```

Compile wakes at t=10.001ms:
```
Same avg_vruntime ≈ 5ms
Compile's preserved lag from when it slept ≈ 3
Sum of weights before Compile joins = 2338 (Audio + Indexer)
Sum of weights after Compile joins = 3362
Compile's lag on wakeup = 3 × (3362 / 2338) ≈ 4.3
vruntime_Compile = 5 - 4.3 = 0.7
VD_Compile = 0.7 + 3 = 3.7
```

Deadlines:
```
Audio: VD = -2.9
Compile: VD = 3.7
Indexer: VD ≈ 15.6

-2.9 < 3.7 < 15.6 → Audio runs first
```

Why Audio always gets <0.1ms latency:
1. Audio has positive preserved lag from its sleep pattern (runs briefly, sleeps while others run)
2. Wakeup formula adjusts lag proportionally to weight changes
3. vruntime = avg - lag_adjusted puts it ahead of current tasks
4. Small slice (1ms) → early deadline (vruntime + 1)
5. Indexer's low weight → large slice (10.6ms) → late deadline (vruntime + 10.6)
6. Deadline comparison is deterministic, no buddy randomness

Result: No glitches, deterministic scheduling regardless of microsecond wake order differences.

Example 2: Video Decoder
CFS: wakeup_granularity blocks preemption, 7.5ms delay.

EEVDF Setup:
* Video: weight=1024, custom slice=8ms, wakes every 33ms
* Heavy: weight=1024, slice=3ms, CPU-bound
* I/O: weight=1024, custom slice=2ms, wakes every 50ms

Timeline:
t=0-33ms: Heavy runs continuously
```
vruntime_Heavy = 33ms
avg_vruntime = 33ms
```

t=33ms: Video wakes (has been running periodically, preserved lag ≈ 8)
```
Sum of weights before Video joins = 1024 (Heavy only)
Sum of weights after Video joins = 2048
Video's lag on wakeup = 8 × (2048 / 1024) = 16
vruntime_Video = 33 - 16 = 17
VD_Video = 17 + 8 = 25

Heavy: vruntime = 33, VD_Heavy = 33 + 3 = 36
```

25 < 36, so Video runs next. When Video is partway through at t=38ms, I/O wakes:

```
Video at vruntime = 22 (midway in execution)
I/O last vruntime = 20
Current avg (before I/O joins) ≈ 27
I/O's preserved lag when it slept = 27 - 20 = 7

Sum of weights before I/O joins = 2048
Sum of weights after I/O joins = 3072
I/O's lag on wakeup = 7 × (3072 / 2048) = 10.5
vruntime_I/O = 27 - 10.5 = 16.5
VD_I/O = 16.5 + 2 = 18.5

Video: VD = 22 + (8 - amount_run) ≈ 22 + 4 = 26

18.5 < 26 → I/O preempts immediately
```

Result: I/O gets <0.1ms latency.

Why Better:
* No wakeup_granularity threshold
* I/O's positive lag from sleep → wakeup formula preserves advantage
* 2ms slice → early deadline
* Precise deadline comparison

Example 3: Latency-Critical Task
CFS: last_buddy causes 1-1.5ms delays.

EEVDF Setup:
* Task L: weight=1024, custom slice=1ms, needs <1ms response
* Tasks A-D: weight=1024, slice=3ms, CPU-bound

Timeline:
t=0-100ms: A-D run continuously
```
Each accumulates vruntime ≈ 25ms
avg_vruntime ≈ 25ms
```

t=100ms: L wakes (second invocation, preserved lag ≈ 2)
```
Sum of weights before L joins = 4096 (4 batch tasks)
Sum of weights after L joins = 5120
L's lag on wakeup = 2 × (5120 / 4096) = 2.5
vruntime_L = 25 - 2.5 = 22.5
VD_L = 22.5 + 1 = 23.5

Batch tasks at vruntime ≈ 25:
VD_batch = 25 + 3 = 28

23.5 < 28 → L preempts immediately
```

Latency: <0.1ms

Pattern repeats: L consistently gets <0.1ms latency.

Why It Works:
* L's brief execution (1ms) keeps positive lag
* Sleep while others run increases avg, preserves L's good position
* Wakeup formula: lag_adjusted = lag × (new_weights / old_weights) maintains relative advantage
* vruntime_new = avg - lag_adjusted places L competitively
* 1ms slice → early deadline
* No buddy system, no threshold

Example 4: High-Frequency Trading
CFS: Sleeper bonus clusters vruntimes, buddy randomness, millisecond latencies.

EEVDF Setup:
* 100 threads: weight=1024, custom slice=10µs
* Wake every 10-100µs, run 1-5µs

Behavior:
Thread runs 5µs:
```
vruntime += 5µs
avg increases by ~0.05µs
lag = avg - vruntime = -4.95µs (negative, ineligible)
Others: lag ≈ +0.05µs (eligible)
```

Thread sleeps 10µs while others run:
```
Others run, avg increases to ≈ 10µs
Thread's preserved lag when it slept = -4.95µs
```

Thread wakes:
```
Current avg ≈ 10µs
Sum of weights before thread joins ≈ 101376 (99 threads)
Sum of weights after thread joins = 102400
Thread's lag on wakeup = -4.95 × (102400 / 101376) ≈ -5.0µs
vruntime_new = 10 - (-5.0) = 15µs

Wait, this is worse! Let me recalculate...

Actually, after sleeping 10µs:
Thread's vruntime frozen at 5µs
Current avg ≈ 10µs (others advanced)
Thread's lag = 10 - 5 = 5µs (positive now from sleep!)

Thread's lag on wakeup = 5 × (102400 / 101376) ≈ 5.05µs
vruntime_new = 10 - 5.05 = 4.95µs
VD = 4.95 + 10 = 14.95µs

Currently running thread: vruntime ≈ 10, VD ≈ 20µs
14.95 < 20 → preempts
```

Latency: 10-30µs typical (meets <50µs requirement).

Why Better:
* No sleeper bonus clustering
* Each thread's vruntime tracks actual execution
* Wakeup formula: lag_adjusted = lag × (new_weights / old_weights) preserves fairness
* Lag mechanism: vruntime = avg - lag_adjusted provides fair positioning
* Ultra-short slices (10µs) possible
* Deterministic deadline ordering

Example 5: Game Engine 60fps
CFS: Random scheduling order, inconsistent frame times (16-21ms variation).

EEVDF Setup:
* Render: weight=1024, custom slice=10ms
* Physics: weight=1024, custom slice=3ms
* AI: weight=1024, custom slice=8ms
* Audio: weight=1024, custom slice=0.5ms

Frame boundary (all wake at t=0):
All starting fresh:
```
avg_vruntime = 0
All have lag ≈ 0 or equal initial position
vruntime for all ≈ 0

Deadlines:
Audio: VD = 0 + 0.5 = 0.5ms
Physics: VD = 0 + 3 = 3ms
AI: VD = 0 + 8 = 8ms
Render: VD = 0 + 10 = 10ms

Deterministic order: Audio → Physics → AI → Render
```

Every subsequent frame (t=16.67ms, 33.34ms, ...):
After one frame cycle:
```
Audio: vruntime = 2 (ran 4 times)
Physics: vruntime = 3
AI: vruntime = 8
Render: vruntime = 10
avg_vruntime ≈ 5.75 (weighted average)

All wake simultaneously:
Audio lag when slept = 5.75 - 2 = 3.75
Sum weights before Audio = 3072, after = 4096
Audio lag on wakeup = 3.75 × (4096 / 3072) = 5.0
vruntime_Audio = 5.75 - 5.0 = 0.75
VD_Audio = 0.75 + 0.5 = 1.25

Physics lag when slept = 5.75 - 3 = 2.75
Physics lag on wakeup = 2.75 × (4096 / 3072) = 3.67
vruntime_Physics = 5.75 - 3.67 = 2.08
VD_Physics = 2.08 + 3 = 5.08

AI lag when slept = 5.75 - 8 = -2.25 (negative)
AI lag on wakeup = -2.25 × (4096 / 3072) = -3.0
vruntime_AI = 5.75 - (-3.0) = 8.75
VD_AI = 8.75 + 8 = 16.75

Render lag when slept = 5.75 - 10 = -4.25 (negative)
Render lag on wakeup = -4.25 × (4096 / 3072) = -5.67
vruntime_Render = 5.75 - (-5.67) = 11.42
VD_Render = 11.42 + 10 = 21.42

Order: Audio(1.25) → Physics(5.08) → AI(16.75) → Render(21.42)
Same deterministic order!
```

On 2 cores:
* Core 1: Audio(0.5) + Physics(3) + Render(10) = 13.5ms
* Core 2: AI(8ms) parallel
* Wall time: 13.5ms (under 16.67ms budget)

Result: Consistent 13.5ms frame times, no stutter.

Why Better:
* Wakeup formula: lag_adjusted = lag × (new_weights / old_weights) maintains proportional fairness
* Deterministic deadline ordering via vruntime = avg - lag_adjusted
* Audio's 0.5ms slice → always earliest deadline
* Predictable → enables parallelization
* No buddy randomness

2.8 EEVDF Solutions to CFS Drawbacks

2.8.1 No Latency Expression (SOLVED)
CFS: VoIP needs <5ms latency + 10% CPU. Nice 0: high latency. Nice -5: wastes CPU.

EEVDF:
* VoIP: weight=1024 (nice 0), custom slice=2ms
* 8 Encoders: weight=1024, slice=3ms

VoIP wakes every 20ms:
```
After sleeping 20ms:
Current avg ≈ 5ms (encoders ran)
VoIP last vruntime = 2ms
VoIP's lag when it slept = 5 - 2 = 3 (positive from sleep)
Sum weights before VoIP = 8192, after = 9216
VoIP's lag on wakeup = 3 × (9216 / 8192) = 3.375
vruntime_VoIP = 5 - 3.375 = 1.625
VD_VoIP = 1.625 + 2 = 3.625

Encoders: vruntime ≈ 5, VD ≈ 8
3.625 < 8 → VoIP preempts
```

Latency: <0.5ms consistently.

CPU share (over 1s):
* VoIP: 50×2ms = 100ms = 10% ✓
* Encoders: 900ms / 8 = 112.5ms each ✓

Achieves both: Low latency + fair share.

2.8.2 Lack of Guarantees (SOLVED)
CFS: last_buddy causes corner case delays up to 1.5ms.

EEVDF:
* Task L: weight=1024, custom slice=1ms
* Tasks A-D: weight=1024, slice=6ms

L wakes after 100ms sleep:
```
A-D have been running: avg ≈ 25ms
L last vruntime = 1ms
L's lag when it slept = 25 - 1 = 24 (huge positive lag!)
Sum weights before L = 4096, after = 5120
L's lag on wakeup = 24 × (5120 / 4096) = 30
vruntime_L = 25 - 30 = -5
VD_L = -5 + 1 = -4

Batch: vruntime ≈ 25, VD ≈ 31
-4 < 31 → L preempts immediately
```

Latency: <0.1ms every time.

Why: Wakeup formula lag_adjusted = lag × (new_weights / old_weights) amplifies L's advantage, 1ms slice creates very early deadline.

2.8.3 Timeslice Rigidity (SOLVED)
CFS: Fixed 1.15ms slice, buddy deferrals cause 0.6-1.2ms latency.

EEVDF:
* Task S: weight=1024, custom slice=0.5ms
* Task X: weight=3277, slice=9.6ms

S wakes every 10ms:
```
After 10ms sleep:
Current avg ≈ 13ms
S last vruntime = 0.5ms
S's lag when it slept = 13 - 0.5 = 12.5 (very positive)
Sum weights before S = 3277, after = 4301
S's lag on wakeup = 12.5 × (4301 / 3277) = 16.4
vruntime_S = 13 - 16.4 = -3.4
VD_S = -3.4 + 0.5 = -2.9

X: vruntime ≈ 13, VD ≈ 22.6
-2.9 < 22.6 → S preempts
```

Latency: 0.1-0.3ms (well under 3ms requirement).

2.8.4 Heuristic Complexity (ELIMINATED)
CFS: Sleeper bonus + buddy + thresholds → unpredictable.

EEVDF:
```
if woken_VD < current_VD:
    preempt

where VD calculated deterministically:
lag_on_wakeup = lag_when_slept × (new_weights / old_weights)
vruntime = avg - lag_on_wakeup
VD = vruntime + slice
```

Single decision criterion. Deterministic. Predictable.

2.8.5 How to Guarantee Task Runs Every X ms
Question: Ensure task runs every 10ms?

Answer: Set small slice + maintain positive lag through sleep/wake behavior.

Example: Audio task, runs every 5ms, needs <0.5ms latency.

```c
attr.sched_runtime = 500000; // 500µs slice
```

What happens:
```
t=0: Audio runs 0.5ms vruntime, sleeps
     vruntime_Audio = 0.5
     lag when sleeping = -0.5 (slightly negative)

t=0-5: Others run, avg increases to ~5

t=5: Audio wakes
     Current avg = 5
     Audio's lag when it slept = 5 - 0.5 = 4.5 (positive from others running!)
     Sum weights before Audio = 3072, after = 4096
     Audio's lag on wakeup = 4.5 × (4096 / 3072) = 6.0
     vruntime_Audio = 5 - 6.0 = -1.0
     VD_Audio = -1.0 + 0.5 = -0.5
     
     Others: VD ≈ 8-15
     
     -0.5 < 8 → Audio runs immediately
     Latency: <0.1ms
```

Why guaranteed:
1. Short slice → early deadline
2. Sleep pattern → accumulates positive lag (avg increases while vruntime frozen)
3. Wakeup formula: lag_adjusted = lag × (new_weights / old_weights) maintains advantage
4. vruntime = avg - lag_adjusted places task competitively ahead
5. EEVDF always picks earliest deadline among eligible
6. Deterministic: same lag/vruntime state → same decision

For hard real-time: Use SCHED_DEADLINE.
EEVDF provides: Soft real-time with fairness.

2.9 EEVDF Performance Weakness

Low Latency + Long Slice Workload
EEVDF performs poorly for tasks requiring:
* Low scheduling latency AND
* Long time slices

Why: Deadline = vruntime + slice. Long slice → late deadline → scheduled late.

Example:
* Task needs to run every 10ms (low latency requirement)
* But needs 8ms of CPU time each invocation (long slice)
* slice=8ms → deadline is 8ms away → may wait if other tasks have earlier deadlines

Workaround:
* Break work into smaller chunks
* Use multiple 2ms slices instead of one 8ms slice
* Trade-off: More context switches

CFS comparison: CFS could prioritize via nice value, but sacrifices fairness. EEVDF maintains fairness but can't prioritize long-slice tasks for low latency.

2.10 Summary

Improvements Over CFS

```
Aspect              CFS                         EEVDF
Fairness            vruntime + heuristics       Lag + eligibility (mathematical)
Latency control     Implicit (nice)             Explicit (slice)
Preemption          Threshold                   Deadline
Heuristics          5+                          1 (lag decay)
Determinism         Low                         High
Per-task control    No                          Yes (sched_setattr)
```

Practical Benefits
1. Consistent low latency: Audio/VoIP <0.5ms (vs CFS 1-5ms)
2. Workload flexibility: HFT, games, mixed workloads
3. Simplified tuning: Per-app slice vs global parameters
4. Predictable behavior: No timing-dependent races

Trade-offs
EEVDF challenges:
* Applications should set custom slices for optimal behavior
* Learning curve for sched_setattr()
* Poor for "low latency + long slice" workloads

When CFS might be better:
* Legacy apps depending on CFS heuristics
* Workloads where CFS "just works"
* Preference for global tuning vs per-app config

Overall: EEVDF trades slight application complexity for much better latency control, predictability, and mathematical fairness.

Conclusion
EEVDF represents a fundamental shift from heuristic-based to deadline-based scheduling:
1. Explicit latency control via custom slice (sched_runtime)
2. Deterministic scheduling via deadline ordering
3. Mathematical fairness via lag-based eligibility
4. Reduced complexity via heuristic elimination

The key wakeup mechanism uses the formula: lag_on_wakeup = lag_when_slept × (Sum(task-weights) + task_weight) / Sum(task-weights), then vruntime = avg_vruntime - lag_on_wakeup. This ensures tasks that sleep while others run accumulate positive lag, giving them competitive vruntime positions on wakeup. Combined with small custom slices, this creates early deadlines for latency-sensitive tasks.

For latency-sensitive applications, EEVDF enables precise control without sacrificing fairness or requiring elevated privileges. The examples demonstrate 2-10x latency improvements over CFS.

Key takeaway: EEVDF provides two independent dimensions of control (nice for CPU share, slice for latency) where CFS provided only one (nice), enabling "low latency + fair share" combinations that were impossible before.
