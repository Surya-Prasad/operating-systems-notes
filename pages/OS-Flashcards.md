# Advanced OS algorithm flashcards

54 short front/back cards based on your exam notes and slides, with the technical corrections discussed in our review.

Click a card title to reveal its answer. These are study prompts, not questions requiring a reply.

Notation: CAS(location, expected, desired); CLEAR=0, BUSY=1; N=number of participating processes; CS=critical section. P in the Anderson lock is the number of slots, sufficient for outstanding acquisitions. FIFO assumptions apply to individual sender–receiver channels. Distributed-algorithm progress assumes eventual delivery, continued process execution, and eventual CS release. Historical Solaris cards refer to the older many-to-many model. Algorithms are conceptual; real implementations need atomics and memory ordering.

## CPU and execution

<details>
<summary>01. Instruction pipeline</summary>

Fetch instruction → decode/read operands → execute/address calculation → access data memory if needed → write register result if needed. Cue: F–D–E–M–W. Five stages are a teaching model.

</details>

<details>
<summary>02. Recoverable page fault</summary>

Save fault state → enter kernel → validate access → repair mapping → restore → retry faulting instruction. Invalid access may instead cause a signal. Cue: Repair, then retry.

</details>

<details>
<summary>03. Interrupt handling versus switching</summary>

Save return state → run handler → restore/resume, or invoke scheduling when needed. An interrupt does not necessarily switch threads. Cue: Handler first; switch only if needed.

</details>

<details>
<summary>04. Context switch</summary>

Save A’s execution state → select B → change address-space context if needed → restore B’s stack/registers → resume. Preserve the original resume address; do not copy code/heap.

</details>

## Threads and scheduling

<details>
<summary>05. Create a user-level thread without a syscall</summary>

Use existing memory for a stack and thread record → initialize its starting context → enqueue it → switch registers/stack to run it. Many-to-one threads cannot use multiple cores simultaneously.

</details>

<details>
<summary>06. Parallel maximum of an array</summary>

Split array → each thread finds a local maximum → join workers → take maximum of their results. Cue: Split–compute–join–combine. Return results using valid types, not int storage cast to void**.

</details>

<details>
<summary>07. Historical Solaris two-level scheduling</summary>

Library assigns runnable user threads to available LWPs → kernel schedules their associated kernel threads on CPUs. Cue: User thread → LWP/kernel thread → CPU. ACTIVE means assigned to an LWP.

</details>

<details>
<summary>08. Solaris: synchronization wait versus blocking syscall</summary>

User-level wait: save waiter and run another user thread on the LWP. Blocking syscall: blocks that LWP; other LWPs may continue. Cue: Library wait swaps a worker; syscall can stall its carrier.

</details>

<details>
<summary>09. Historical SIGWAITING</summary>

All LWPs enter indefinite kernel waits → kernel sends SIGWAITING → thread library handles it → may create an LWP for runnable work. Does not solve every CPU-bound shortage.

</details>

## Synchronization

<details>
<summary>10. Mutex-protected critical section</summary>

Acquire mutex → manipulate protected shared data → release mutex. Cue: One at a time. Usually no FIFO guarantee; a delayed owner can delay everyone.

</details>

<details>
<summary>11. Condition-variable wait</summary>

Hold mutex → while predicate false, wait → wait atomically releases mutex and blocks → reacquire mutex → recheck predicate. Cue: Release–wait–reacquire–recheck. Signal is not a stored permit.

</details>

<details>
<summary>12. Bounded-buffer producer</summary>

Lock → while full, wait(not_full, mutex) → insert and increment count → signal(not_empty) → unlock. Cue: Wait for space; announce data. Waiting releases the mutex.

</details>

<details>
<summary>13. Bounded-buffer consumer</summary>

Lock → while empty, wait(not_empty, mutex) → remove and decrement count → signal(not_full) → unlock. Cue: Wait for data; announce space. Use while, not if.

</details>

<details>
<summary>14. Semaphore P() and V()</summary>

P: consume an available permit, otherwise wait. V: release a permit, enabling a waiter or leaving availability for a future P. Cue: P takes; V gives. Binary: at most one available permit; counting: multiple.

</details>

## Atomic primitives

<details>
<summary>15. Test-and-set</summary>

Atomically return old x and set x to BUSY. Old CLEAR means acquired; old BUSY means retry. Cue: Claim and report what was there.

</details>

<details>
<summary>16. Fetch-and-add</summary>

Atomically return old x and replace x with old x + amount. Cue: Take the old number, advance the counter. Used to issue unique tickets.

</details>

<details>
<summary>17. Compare-and-swap (CAS)</summary>

CAS(location, expected, desired): replace only if current equals expected; report success/failure. Cue: Still what I read? Then commit. This deck consistently uses expected before desired; lecture notation may differ.

</details>

<details>
<summary>18. Atomic exchange</summary>

Replace a location and return its old value. Cue: Install me; tell me who was there. MCS uses this to append to the tail and obtain its predecessor.

</details>

## Spinlocks

<details>
<summary>19. Test-and-set lock (TAS)</summary>

Repeatedly TAS until old value is CLEAR → critical section → release-store CLEAR. Cue: Keep trying to claim. Weakness: failed attempts demand write ownership and cause coherence traffic.

</details>

<details>
<summary>20. Test-test-and-set lock (TTAS)</summary>

Read while BUSY → when apparently free, TAS → retry if lost → release-store CLEAR. Cue: Look before claiming. Fixes repeated write attempts; still causes a release-time rush.

</details>

<details>
<summary>21. Delay after noticing release</summary>

After contention, poll while BUSY → once release noticed, delay before retrying acquisition. Cue: Spread the rush. Weakness: polling itself may still cause costly reloads.

</details>

<details>
<summary>22. Delay between memory references</summary>

After each unsuccessful polling/acquisition iteration, delay → recheck. Cue: Check less often. Helps when polling traffic is expensive; adds detection latency and requires tuning.

</details>

<details>
<summary>23. Ticket lock</summary>

Initialize both counters to 0 → fetch-and-add next_ticket → wait for now_serving == my_ticket → enter → advance now_serving. Cue: Take a number; watch the display. FIFO, but everyone watches one line.

</details>

<details>
<summary>24. Anderson array lock</summary>

Fetch-and-add tail → slot = ticket mod P → wait on own flag → reset own flag for reuse → enter → enable flags[(slot+1) mod P]. Cue: Own slot; wake next slot. Need sufficient slots, padding/alignment.

</details>

<details>
<summary>25. MCS acquisition</summary>

Initialize node.next=NULL → exchange tail with own node → if no predecessor, enter → otherwise set locked=true before linking pred.next → spin on own locked flag. Cue: Queue once, wait locally.

</details>

<details>
<summary>26. MCS release</summary>

If successor linked, clear its locked flag. Otherwise CAS tail from me to NULL: success means empty; failure means wait for successor link, then hand off. Cue: Empty or unfinished arrival? CAS decides.

</details>

<details>
<summary>27. Why must MCS release wait after failed CAS?</summary>

Successor may have changed tail but not yet written my next pointer. Wait for that link before handoff. Clearing tail unconditionally could erase the successor. Cue: Joined tail ≠ linked next.

</details>

<details>
<summary>28. False sharing and padding</summary>

Different flags on one cache line interfere when one is written. Separate heavily contended flags/nodes onto different lines. Cue: Different variables, same line. Padding trades space for less traffic.

</details>

<details>
<summary>29. Acquire/release memory ordering</summary>

Successful acquisition must see prior owner's protected writes; release publishes this owner's writes. Cue: Ownership plus visibility. Atomic lock updates alone do not justify arbitrary memory reordering.

</details>

## Concurrent updates

<details>
<summary>30. Lock-based software fetch-and-add</summary>

Lock → read old → write old+v → unlock → return old. Cue: Exclude, then update. Weakness: progress depends on lock owner.

</details>

<details>
<summary>31. CAS-based software fetch-and-add</summary>

Read old → compute proposed=old+v → CAS(x, old, proposed) → retry on failure → return successful old. Cue: Read–prepare–commit–retry. Successful CAS is the linearization point.

</details>

<details>
<summary>32. Why CAS fetch-and-add is lock-free, not wait-free</summary>

With strong/non-spurious CAS and only these writers, failure indicates a competing update succeeded. System progresses, but one thread may lose forever. Cue: Someone wins, not necessarily me.

</details>

<details>
<summary>33. Optimistic copy-and-commit</summary>

Read state → prepare private changes → validate relevant state → atomically publish → retry if conflict. Cue: Prepare privately; publish together. Optimism alone does not guarantee lock freedom.

</details>

<details>
<summary>34. Lock-free versus wait-free; latency</summary>

Lock-free: system-wide progress. Wait-free: every continuing thread finishes within bounded own steps. System latency counts steps until some operation completes; process latency concerns one thread. Neither is a wall-clock guarantee.

</details>

## Logical clocks

<details>
<summary>35. Happens-before</summary>

Same-process order + send before corresponding receive + transitivity. Neither direction means concurrent events. Cue: Local–message–chain. Concurrent does not mean simultaneous.

</details>

<details>
<summary>36. Lamport clock update</summary>

Start C=0. Local/send: C=C+1; attach C on send. Receive timestamp T: C=max(C,T)+1 using pre-receive C. Cue: Take the larger, then add one.

</details>

<details>
<summary>37. Lamport total ordering</summary>

Compare timestamps, break ties with one consistent process-ID rule. Happens-before implies smaller timestamp; converse fails. Cue: Number, then name. Ordering labels alone do not ensure safe delivery/execution.

</details>

## Distributed mutual exclusion

<details>
<summary>38. Centralized mutex</summary>

Send REQUEST to coordinator → wait for GRANT → enter → send RELEASE → coordinator grants next queued requester. Cue: One referee. 3 messages per entry; coordinator is bottleneck/failure point.

</details>

<details>
<summary>39. Lamport mutex: request and entry</summary>

Enqueue own timestamped request → broadcast REQUEST → receivers enqueue and ACK → enter when own request is first AND a later-timestamped message arrived from every other process. Cue: First in queue + everyone heard.

</details>

<details>
<summary>40. Lamport mutex: release and proof cue</summary>

Exit → broadcast RELEASE → everyone removes your request. 3(N−1) messages. Safety: two entrants cannot both have the earliest outstanding request. Progress needs eventual delivery, execution and release.

</details>

<details>
<summary>41. Why Lamport mutex needs FIFO</summary>

A later message from sender j cannot overtake j's earlier request to the same recipient. Thus earlier requests cannot remain hidden behind the later message. Cue: Same channel, same order. Clocks do not provide FIFO.

</details>

<details>
<summary>42. Ricart–Agrawala: request/reply rule</summary>

Choose request number above highest seen → broadcast REQUEST → wait for all replies. Defer while inside CS, or while requesting with a smaller (number,ID); otherwise reply immediately. Cue: Reply means you may go before me.

</details>

<details>
<summary>43. Ricart–Agrawala: exit and proof cue</summary>

Exit → clear requesting state → send all deferred replies. 2(N−1) messages; no separate RELEASE. Safety: smaller-priority request withholds permission from the larger. Earliest outstanding request can collect replies.

</details>

<details>
<summary>44. Suzuki–Kasami: bookkeeping</summary>

Every node has RN[j]=highest request number seen from j. Unique token carries LN[j]=last served request number from j and queue Q. Pending request: RN[j]=LN[j]+1. Cue: Seen versus served.

</details>

<details>
<summary>45. Suzuki–Kasami: request and receive</summary>

If holding token, enter locally. Otherwise increment own RN → broadcast REQUEST → wait for token. On receiving (j,n), set RN[j]=max(RN[j],n); idle token holder sends token if j has a pending request.

</details>

<details>
<summary>46. Suzuki–Kasami: release</summary>

Set LN[me]=RN[me] → append pending nodes not already queued → if Q nonempty, pop head and send token with remaining Q and LN; otherwise retain token. Cue: Mark served–queue pending–pass token.

</details>

<details>
<summary>47. Suzuki–Kasami: safety and cost</summary>

Only the unique token holder enters. Remote acquisition: N−1 request messages + 1 token = N; already-held token: 0 messages. Cue: One token, one owner. Token loss/holder failure needs recovery outside this basic model.

</details>

## RPC

<details>
<summary>48. Stub generation</summary>

Describe procedures, parameter types and results in an interface → RPC compiler generates client/server stubs. Cue: Define once; generate both sides. Pointer parameters also need serialization/length rules.

</details>

<details>
<summary>49. Server export / registration</summary>

Server stub exports (type,instance,dispatcher) → runtime records export → name service registers availability. Cue: Advertise the service.

</details>

<details>
<summary>50. Client import / binding</summary>

Ask name service for exporter → contact server runtime → obtain export identifier and table index → retain (address,ID,index). Cue: Find–verify–remember. Binding is not automatically authorization.

</details>

<details>
<summary>51. RPC invocation and reply</summary>

Caller → client stub packs → client runtime sends → server runtime dispatches → server stub unpacks → procedure runs; result follows reverse path. Cue: Pack–send–dispatch–execute–return.

</details>

<details>
<summary>52. RPC timeout and retransmission</summary>

No reply before timeout → retry according to protocol, using same invocation ID. Timeout cannot tell whether execution occurred. Cue: Missing reply ≠ missing execution. Retry must account for duplicate effects.

</details>

<details>
<summary>53. RPC duplicate detection</summary>

Track activity ID + sequence number at server → recognize repeated/old requests → avoid re-execution; retransmit cached result when appropriate. Cue: Same call ID, same invocation. Guarantees depend on retained state.

</details>

<details>
<summary>54. RPC acknowledgement optimization</summary>

Result packet acknowledges call; next call may acknowledge preceding result. Use explicit acknowledgement/timeout handling when needed. Cue: Useful packets do double duty. This is transport-specific, not proof TCP is always inefficient.

</details>

## Fast recall

- Spinlocks: claim → read first → delay → tickets → slots → nodes.
- Updates: exclude then update, or prepare then CAS.
- Lamport clock: max, then +1. Order: timestamp, then ID.
- Distributed mutex: referee → ordered queues → deferred permissions → unique token.
- RPC: define → export → import → pack → send → execute → reply → deduplicate retries.

## Sources and corrections

Based on all ten updated Markdown note files supplied on October 13, 2026, and the fifteen lecture slide decks already reviewed in this conversation. The cards use corrected semantics rather than preserving typographical or conceptual errors in the notes. Main corrections: CV waits use while; semaphore P consumes a permit only when one is available; ordinary context switching preserves a resume address and does not copy all memory; CAS/exchange differ; Anderson indexing is (slot+1) mod P; MCS uses atomic tail exchange; RN exists at every Suzuki–Kasami participant; FIFO is per channel; RPC binding is separate from authorization. The CAS retry loop is not a bounded per-thread O(N) algorithm simply because failed attempts imply competing updates. Modern hardware multiplication and later Solaris models are outside these algorithm cards.

