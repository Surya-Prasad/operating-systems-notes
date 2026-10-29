## Solaris 2-level Scheduling Scheme
	- LWP State, User level Thread state
	- (E, _) -> LWP running, no user level thread -> F
		- IF nothing is running, why should it be running?
		- SHould be blocked
	- (R, _) -> Same -> F
	- (B, _) -> Waiting for work -> T
	- (_, E) -> No LWP, user T running -> F
	- (E, E) -> valid
	- (R, E) -> User level thread running -> Valid
	- (B, E) -> Possible
		- When some kind of blocking call is there, user thread thinks it is running
	- (_, R) -> Ready, no LWP -> Valid
		- Ready, waiting to be picked up
	- (E,R) -> Executing, Ready
	- (R, R) -> F
	- (B,R) -> Ready, but LWP is blocked
		- Cannot happen
		- If ready, LWP has picked it up
		- F
	- (_, B) -> Valid
		- If ULT is blocked, LWT should not be wasted on thread
	- (E, B) -> ULT is blocked but LWP running
		- F
		- Does not make sense, why running
	- (R, B) -> F
		- Why should it be ready to run
	- (B, B) -> If it is blocked, the LWP should also not be there
		- But marks were given for T or F
		- Somewhere in paper, there were some optimizations where they block it preemptively for fast wakeup
-
- ## Thread Mutex Lock
	- availservice()
	- Scenario
		- If one thread comes and puts it in use and takes a while
		- If another thread comes, it will come and block
		- If the first thread finishes and calls `pthread_cond_signal(c)`
			- It will wake up the thread that is blocked
			- But that other thread has not yet left the CS because the other thread is still executing
		- A third thread thread can come in, gets the thread, checks in_use which is false, does not do a cant_wait, gets into service, takes time
		- The thread earlier can come out, says in_use = TRUE
		- Goes into CS -> Two threads are there
	- Basically, the `if` should be a `while`
-
- ## Possible Executions of code
	- Possible number of ways that these can interleave to get one serial order
	- Part 1
		- $\frac{(m1 + n1 + n1 + n2)!}{(n1 + n2)! (m1+ m2)!}$
	- Part 2
		- 2
		- You can either have one or the other execute before the other
	- Part 3
		- Either block's L1 has to come first
		- Say T1's L1 comes first, how many possible ways can the remaining blocks can execute? -> Is the framing of this question
			- We calculate the number of ways that T1's L2 can interleave with T2's L1, which is
			  $$\frac{(m1 + n2)!}{m1! n2!} + 1$$
			- If T2's L1 comes first,
			  $$\frac{(n1 + m2)!}{n1! m2!} + 1$$
		- These are the only ways this can happen, so final answer is sum of these two cases
-
- ## Purposes of CallID sent by the RPC
	- Demultiplexing
	- Avoiding replaying the call -> For removing duplicate calls
-
- ## Binding Info
	- Network Address
	- Row Number of table index
	- Machine unique ID
-
- ## Spinlocks
	- Everyone is looking at `now_serving`
	- Write propagation is everywhere, lot of overhead
	- We need to only send message to next in-line for them to invalidate
-
- ## Converse of Causality
	- This is asking if a caused b
	- We can make the converse true
		- Problem is, when we see an event at b, we do not know what the P1's clock value is at a
		- The way to make the converse true is to make it a tuple of (??, clock value of b)
		- Now we have a vector -> We can now say unless C(a) < C(b) for each value, a did not cause b
		- TODO Distributed timestamps in Lamport Clocks
-
- ## In-Order Delivery
	- Ask in OH
-
- ## Block Free
	- ```c
	  bool compare-and-swap(a, b, c): 
	  {
	   while !success {
	    temp = load-linked(A, pid)
	    if (temp != c) return false
	    else
	    {
	      success = Store-Conditional(A, B, pid)
	        b = temp;
	    }
	    return true 
	  }
	  ```
	- Doing it for two variables would lose atomicity
		- We can combat this by putting it in one structure and having it in one structure
		- But inefficient - When you create a shadow copy, we create the copy of the structure and then update it -> Doing lot of copies and then garbage collection overhead -> Garbage collection by itself requires mutex -> We're just moving the problem elsewhere
		-