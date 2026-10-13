## *Scheduler Activations: Effective Kernel Support for the User-Level Management of Parallelism” (1992)*, by Anderson, Bershad, Lazowska, and Levy.
	- Source: ![1-kernel-user.pdf](../assets/1-kernel-user_1791928307039_0.pdf)
	- Core Idea:
		- Kernel gives CPUs
		- User runtime decides which threads execute
		- Notifications keep them synchronized
	- User threads have quick switching and execution, but kernel cannot see them
	- Kernel has blocking and preemption but overhead is high
	- So *Scheduler Activations* provide execution context to execute threads on, a way for kernel to notify user runtime, storage for an interrupted thread's state
	- | Kernel | User runtime |
	  |---|---|
	  | Allocates CPUs among applications | Selects user threads for those CPUs |
	  | Handles blocking, faults, and preemption | Maintains ready queues and synchronization |
	  | Reports relevant changes | Decides which interrupted threads resume |
	- Calls
		- System Call: Application -> Kernel
		- Upcall: Kernel -> User space handler
			- Processor added
			- activation preempted
			- activation blocked
			- activation unblocked.
	- When a runtime with the critical section is interrupted
		- Examine the saved state
		- Detect if it has a critical section
		- temporarily resume to finish section
		- Return control to upcall handler
	- They avoid slowing every critical section by creating copies of critical section code. Execution resumes in the duplicate.
-
- ## *“The Performance of Spin Lock Alternatives for Shared-Memory Multiprocessors”* (1990), Thomas Anderson et al
	- Spinning favorable when time spent spinning is small compared to blocking and resuming, or there is no other useful work to run
	- ### Efficiency of a lock:
		- Uncontended latency: Acquisition cost when no one else wants the lock
		- Waiting Traffic: Network interference generated when lock is busy
		- Handoff Delay: Time between release and next acquisition
	- ### Basic Spinlocks
		- TAS and TTAS
		- Shortcomings
			- TAS: Failed attempts compete with useful attempts, bus pollution
			- TTAS: Once write is done, all other copies also attempt TAS, which means that every copy then is invalidated again - Time for network to settle down when another process gets the lock - Quiesce time
	- ### Delay based Spinlocks
		- After release: Delay after release
		- On reference: Dynamic backoff after failed acquisition
	- ### Anderson's algo
		- Assign each request a position
-
- ## Maurice Herlihy’s *“A Methodology for Implementing Highly Concurrent Data Structures” (1990)*
	- Core Idea: Read current version -> Write to a shadow copy -> Wait to acquire critical section -> Write in-place with Compare&Swap -> help other processes finish
	- Why block free but not wait free?
		- if CAS fails, it means some other process has updated it
		- Which means the current process is starving.
		- The current process could starve indefinitely
	- ### Terms
		- A **benevolent side effect** changes the representation without changing the abstract contents.
		- A **checkpoint** publishes such an improvement using CAS:
			- Success: the improvement survives even if later work fails.
			- Failure: restart the operation.
		- A **future** represents a computation postponed until needed.
	- **What is linearizability?**
	  Operations appear atomic and preserve the order of non-overlapping calls.
-
- ## Leslie Lamport’s *“Time, Clocks, and the Ordering of Events in a Distributed System” (1978)*.
	- Core idea: we can order distributed events consistently without knowing the exact physical time.
	- 1. Why is ordering difficult?
		- Each system only sees it's own events and messages
		- The arrival order of messages can differ from the order that it is sent
	- 2. Causality
		- a -> b when
			- Same Process: Timestamp(a) < timestamp(b)
			- Message: Sending a message happens before receiving response
			- Transitivity: a -> b && b -> c => a -> c
		- If you cannot ascertain order, events are concurrent (not simultaneous)
	- 3. Update
		- C <- max(C, T) + 1
	- a -> b => C(a) < C(b), not other way around
	- Partial to Total ordering
		- Arrange them by (timestamp, Process_ID) and then compare timestamps first, then use process ID to break ties
	- **What worsens the physical-clock bound?**
	  Larger diameter, greater rate error, longer synchronization intervals, and more delay uncertainty.
	- **Distinguish drift and skew.**
	  Drift concerns clock rate; skew concerns differences between readings.
-
- ## Ricart and Agrawala’s *“An Optimal Algorithm for Mutual Exclusion in Computer Networks” (1981)*.
- Core Idea: Ask everyone for permission, each node replies immediately or defers until your turn
- Information Kept: 
  
  | Variable | Meaning |
  |---|---|
  | `Highest_Sequence_Number` | Highest request number seen or chosen |
  | `Our_Sequence_Number` | Number assigned to our current request |
  | `Requesting_Critical_Section` | True while requesting **and while inside** the critical section |
  | `Outstanding_Reply_Count` | Number of permissions still missing |
  | `Reply_Deferred[j]` | Whether we owe node \(j\) a reply |
- ### Algorithm
	- While Requesting:
	  ```
	  Choose our req_no as highest_req_no + 1
	  Mark ourselves as requesting
	  Send REQUEST(req_no, node_ID) to everyone else
	  Enter CS after receiving all N - 1 replies
	  ```
	- While receiving request, 
	  
	  ```
	  Update highest number seen
	  If neither waiting nor inside cs : Reply immediately
	  If inside CS : Defer till done before replying
	  If waiting, incoming request has priority : Reply immediately
	  If Waiting but our request has priority : Defer
	  ```
	- When leaving:
	  ```
	  Stop requesting
	  Send all deferred replies
	  ```
-
-