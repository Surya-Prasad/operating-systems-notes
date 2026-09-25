- Implements Concurrency (Between users, activities of a user, etc)
- Insulates/Isolates one activity from another
	- Correctness of a process is completely insular from others
	- Processes can run safely without impacting each other by default
	- Time to switch between one process to another is harder
	- Inter Process Communication is harder
- Drawbacks of Processes
	- Higher scheduling costs (context switch)
		- Direct Costs: Switching Address Spaces
		- Indirect Costs: Cache Flushes etc
	- State Sharing of Process is a problem
- Aliasing Problem
	- Two processes generate the same virtual address, but they might not be the same address in each space
	- Intel purged this, they now attach a Process tag in front of the address space; This should resolve the addressing problem
- Threads solve this problem
-
- Threads decouple address spaces from processes
- Now allow for multiple activities within the same address space
- Threads within a process share (code, data, heap)
	- Code can be shared - There is no problem because different processes have different Program Counters (PC) -> So different parts of code can be tracked
	- Data can be shared - If you have data inconsistency, that is the program's problem, not the system's
	- Heap can be shared - Same as the above, multiple mallocs are possible
	- Stacks cannot be shared - The stacks have program states - lookup cactus stacks - StackOverflow
- Switching between threads only switches the stacks (and registers - Because Number of Registers are Smol) - None of the CodeSeg gang gets changed
	- Switching between processes is what would change CodeSeg gang
- Sharing is implicit
- There is no protection between processes in a thread - which is intended
-
- PThread Library
	- Types
		- pthread_create() - Create/Fork
		- pthread_exit() - Terminate a thread
		- pthread_join() - Wait for a thread to exit
		- pthread_self() - Returns thread ID
	- Library calls vs System Calls
		- Library routines are run as a function of the process
		- However, system calls are trapped into the operating system and run in the OS - Involves a protection privilege switch and runs in privileged mode
	- When multiple processes use the same library, rather than copying it, OSes just manipulate page tables to point to the same library, even though logically each process is separately calling it's own library
	- Can you create threads without actually using system calls?
		- If we think about it, we just need CodeSeg gang, not the resource of a process
		- We don't need a system call for it, we can do it using assembly and C
		- Try it
	- Multithreaded programs
		- Programs always start as a process with a single thread
		- Global elements are stored in the data segment (shared across all the threads)
		- We fork off the main thread
			- `pthread_create(&tid0, NULL, fn_ptr, (void *) 0);`
			- So we would have a parent and a child thread
			- We give the function pointer to pthread_create
			- Now, we have different PCs in both the threads
		- We now would be creating a new thread off the thread
			- `pthread_read_create(&tid1, NULL, fn_ptr, (void *) 1);`
		- Now looking at each thread
			- The find_max local variables are stored in their own stacks
			- Each thread tries to find maximum element in a subsection of the total array
			- We will return max
		- The parent waits for both threads to finish and find the max between both
			- ```C
			  pthread_join(tid0, (void **) &max0);
			  pthread_join(tid1, (void **) &max1);
			  max = max0;
			  if (max1 > max) max = max1;
			  return max
			  ```
-
- Thread Synchronization
	- Things are runtime-dependent, and if not careful, race-conditions
	- The math behind Synchronization
		- Synchronization is basically restricting the total number of combinations that we can use to combine
		- Not a permutation problem, because the order of instructions matter
		- If there are 3 threads, T1 with n1 instr, T2 with n2 instr, T3 with N3 instr, then total number of combinations are $$\frac{(n_1 + n_2 + n_3)!}{n_1! \cdot n_2! \cdot n_3!}$$
	- Synchronization
		- Locks
			- Used to guard sections of code where data is manipulated
			- Ordering is not as important (only exclusion)
			- Waiting for events to happen is not easy to implement
				- So locks alone are not sufficient
		- Condition Variables
			- The ordering constraint
				- Dictates what should happen after what
			- c_wait() and c_signal() operations
			- A thread blocked on c_wait() returns when another performs a c_signal()
			- What differentiates a conditional variable from a (boolean) semaphore?
				- Let us take the bounded buffer problem
				- Think of a circular buffer, a producer that keeps adding items and a consumer removing them
				- Let us say:
				- ```C
				  cond_t not_full, not_empty;
				  int count == 0;
				  
				  append() {
				    if (count == N) c_wait(not_full);
				    // Add to Buffer, Update count
				    c_signal(not_empty); //Update not_empty
				  }
				  
				  remove() {
				    if (count == 0) c_wait(not_empty);
				    // Remove from buffer, update count
				    c_signal(not_full); //Update not_full
				  }
				  ```
				- In conditionals (unlike semaphores), the signal might get lost if the receiver is not waiting for it
				- If signal is done before the wait, then signal is lost
				- If we were to work with locks
					- If we put lock and unlock at line 5, then we would have an infinite wait/deadlock
					- So where will we put the lock and unlock?
					- We need to lock inside the wait()
					  
					  ```C
					  cond_t not_full, not_empty;
					  int count == 0;
					  mutex_lock m;
					  
					  append() {
					    mutex_lock(m);
					    //Temporarily relinquish the lock while waiting
					    if (count == N) c_wait(not_full, m); 
					    // Add to Buffer, Update count
					    c_signal(not_empty); //Update not_empty
					    mutex_unlock(m);
					  }
					  
					  remove() {
					    mutex_lock(m);
					    if (count == 0) c_wait(not_empty, m); 
					    // Remove from buffer, update count
					    c_signal(not_full); //Update not_full
					    mutex_unlock(m);
					  }
					  ```
					- We temporarily relinquish the lock when we are waiting and then regain it when we get the signal
					-
			- Semaphores
				- Condition Variables with state (signals) preserved
				- They preserve the signal
				- They have a count variable which preserves the signal
				- Counting Semaphores (boolean semaphores) offer more flexibility
				- Operations
					- P() - Wait
					- V() - Signal
				- Invented by Dijkstra
					- He's Dutch, so wait and signal mean P and V in Dutch
	- Threads can be implemented on
		- Single core
		- Multicore/multisocket systems
		- Distributed / Shared memory processors
		- Networked / Distributed Workstations
	- Flavors of threads
		- User-Level Threads
			- Threads that are created and maintained entirely by the user-code/libraries
			- Kernel is unaware of their existence
			- Advantages:
				- Scheduling/Switching can be more efficient
				- They are more lightweight - Systemcalls are always more expensive, because every call involves a privilege switch, address space switch etc
		- Kernel-Level Threads
			- Kernel explicitly creates (by explicit syscalls) and manage the threads
			- Eg: Windows
			- Advantages
				- The OS just looks at User-Level threads as a single thread and if the thread needs more
				- Kernel threads have better resource allocation
					- If we have a 1000 threads and one of them is running the file read syscall, it will switch from a READY state to a LOCKED state; This will lock the remaining 999 threads;
					- If this was created by the kernel, it will allocate the appropriate amount
-
-
- Solaris Multithreaded Architecture
	- Sun Microsystems - Acquired by Oracle now
		- Tried to build an OS with both user and kernel level threads
		- Kernel level threads are synonymous with LightWeight Processes (LWP)
	- Two level threaded implementation
		- Missed this, need to brush up
	- User Level Scheduling
	-