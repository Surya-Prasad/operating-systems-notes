- We will be looking at why we don't think we should use locks
- Why don't we want to use locks?
	- Locks prevent other processes from using the resource
	- When the process crashes when holding the resource, it is not easy to transfer the lock.
	  Because there may be critical sections that should not be seen by those that the lock is being passed on to. 
	  Lot of overhead to undo progress
- Systems that use high concurrency do not desire locks
-
- # Desirables
	- Build data structures that can be accessed in:
		- Non-Blocking Fashion (Block Free)
			- SOME thread will complete the operation within a finite number of steps
			- Important from system POV
		- Wait-Free fashion
			- Each/EVERY thread will complete the opeation in a finite number of steps
			- Important from user-thread POV
	- Regardless of which thread is being pre-empted/halted/delayed, some thread can proceed into the critical section
	- How we do this is put a bound on the thread trying to access the critical section
		- Kinds of bounds
			- System Latency
				- Non-Blocking latency
				- Largest number of steps that the system can take without any one thread successfully completing an operation in a non-blocking implementation
				- So we assume that some thread will get it in system latency time
			- Process Latency
				- Largest number of steps that the system can take when all threads successfully complete an operation [CHECK THIS]
		- Pessimistic Concurrency Control:
			- When we get a lock before trying to do something
			- We assume everyone else is trying to do something and you try to get in before everyone else
			- Locks try to do this
		- Optimistic Concurrency Control:
			- We do stuff and then check for lock
			- So we access critical section and then worry about if someone else is using the lock
				- If they are, then rollback and restart
			- Transactions in databases do this
	- We will be covering block-free in this course, not wait-free
	-
	- ## Analogy to Databases
		- Analogy to Database: `Enclosed in block like this`
		- Operations
			- Tell the system you are `beginning a transaction`
			- Do loads/stores on the data structures (*Write to Copies*)
			- Tell the system that you are `ending a transaction`
		- If successful (no one committed between your begin and end)
			- Atomically reflect the changes `(Commit)`
			- We don't *Write in Place* but write them in shadow copies
				- `Reads` would not be a problem but `Write` would be rolled back
				- We should also write to the actual data ***Atomically***
		- If not successful (someone commits in between)
			- Undo and Retry `(Abort)`
	- ## Designing Block-Free Variants
		- ### Conventional (Lock-Based) Fetch&Add looks like this
		  ```c
		  int x;
		  
		  fetch&add(x, v) {
		    int old;
		    mutex_lock(L);
		    old = x;
		    x = old + v;
		    mutex_unlock(L);
		    return old;
		  }
		  ```
		- ### Block-Free Implementation
		  ```c
		  int x;
		  
		  fetch&add(x, v) {
		    int old;
		    int temp;
		    success = false;
		    while(!success) {
		      old = x;
		      temp = old + v;
		      success = compare&swap(x, new, old);
		    }
		    return old;
		  }
		  ```
			- This is not a wait-free implementation
			- Proof that this is a O(n) non-blocking implementation
				- The intuition is that if we fail, it means that someone else has succeeded
				- So we can bound this
-