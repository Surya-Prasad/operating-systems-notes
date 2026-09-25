## Spinlocks on Shared Memory Multiprocessors
	- This is an implementation of locks on multiprocessors
	- Two common techniques for implementing synchronization
		- Blocking: Give up the CPU and Context Switch to another activity
		- Busy Waiting: Spin Locks
	- Busy Waiting:
		- The thread trying to acquire lock is continually polling availability in a tight loop.
		- Thread is ACTIVE and scheduled on the CPU, consuming cycles but not doing meaningful work
	- On Multiprocessors, you would prefer busy waiting, since
		- The sections that are trying to procure the lock are not that large
	- Assumed Hardware Support
		- Atomic Instructions (Eg: Fetch and Add, Test and Set, etc)
			- If something takes 2 or 3 instructions, a single instruction is provided
			- It is not required, but without them, it becomes excessively complex
			- Atomic test & set
			  
			  ```c
			  atomic test&set(x) {
			    int temp = x;
			    x = BUSY;
			    return(temp);
			  }
			  ```
			- Atomic Fetch and Add
			  
			  ```c
			  atomic fetch&add(x) { 
			  int temp = x;
			  x = x + y;
			  return(temp);
			  }
			  ```
			- If they do not exist atomically and exist as two separate instructions, it would get much harder to synchronize them as a unit
-
- ## Bus-Based Shared Memory Multiprocessors
	- Simplest version of the Shared Memory Multiprocessors
	- What it is
	- /Bus gets accessed when cache is missed
	- Instruction Flow
		- Say Process P1 runs `load A, R0`
			- Cache miss, so P1 pulls A from Memory
			- And then Stores it in R0
			- Updates Cache
		- Say we have `store R1, A`
			- Paradigm: It is very important that every component of the system has the same, latest value of A
			- Two ways to do this:
				- Update A everywhere everytime
				- Wipe all other copies of A and ask everything else to reference that one copy that stays
				- Each copy goes on the bus everytime cache misses
				- In a bus, update does not do better than wipe the rest because we would still involve the bus
			-
- # Implementing Spinlocks
	- Continuous Checks
	- Bunch of processes with shared memory
	- Spinning comes with performance compromises
	- ### Spin on test&set
		- We declare a shared variable (boolean) x and continuously put a loop to see if it is true or false
		- x is on DataSegment
		- Everything is going to read and access x
		- Consider this
			- ```c
			  x = CLEAR;
			  lock() {
			    while(test&set(x) == BUSY); //wait when test&set are busy
			  }
			  unlock() {
			    x = CLEAR;
			  }
			  ```
			- Say there is a cache miss, and every instance is going to try a load and store operation on the bus, all of them are going to try and wipe all other instances
			- So this is going to be faulty, needs to be optimized
		- If several people do this, the first person would get a false and exit while the others are stuck in the loop
		- Each attempt creates network traffic (unnecessary).
		- Unlock also contends with trying for a lock
			- Should ideally not do this
			- Why would you try and run a test&set on something that is already locked?
	- ### Spin on test-test&set / Spin on Read
		- Test for lock before trying to run test&set
		- Code
		  ```c
		  x = CLEAR;
		  lock() {
		    while(x == BUSY || test&set(x) == BUSY); //wait when test&set are busy
		  }
		  unlock() {
		    x = CLEAR;
		  }
		  ```
		- This is substantially better than the previous implementation
		- But when P1 unlocks and does x = CLEAR, it wipes out all caches.
			- So all the other processes see that it is unlocked
			- Then the subsequent process comes and checks the cache, it will invalidate the next one and the next process does the next and so on
			- We will have a quadratic number of invalidations
			- Each of these count as network traffic
		- Gap between detecting the lock is clear and trying for it allows more people to try for it than required
		- The invalidations will eventually stop, when another process gets the lock and the cache gets synchronized (Quiesce Time)
			- To prevent this, the Critical Section has to be >> Quiesce time
			- Because if the process that gets the lock unlocks before all the cache gets synced, then that stable state will never be reached
		- There is a similar problem in Networks
			- Ethernet, which is a bus
			- So, we borrow the idea, and back off and retry later, hoping it is random enough that I don't collide
	- ### Delay after Spinning Processor Notices Lock is released
		- Code
		  ```c
		  x = CLEAR;
		  lock() {
		    while(x == BUSY || test&set(x) == BUSY) {
		      while(x == BUSY); //As long as this is busy, we wait
		      Delay(); //When it is free, don't try. Delay further
		    } //The next loop will try and get it
		  }
		  unlock() {
		    x = CLEAR;
		  }
		  ```
		- Problems
			- Delay based solutions are not necessarily the best solutions
			- In invalidation based schemes, the local busy wait may still incur misses (even if unsuccessful) because of other attempts
				- We assume that it is still unlocked right after exiting the inner loop.
				- But it might still incur a cache miss
	- ### Delay between each memory reference
		- For invalidation based schemes
		- Code (missed out)
		  ```c
		  ```
		- When do you think this would perform better than the earlier mechanism?
			- Possible exam question
			- It is possible that the frequency of subsequent reads is lesser than
	- Approaches 3 and 4 are highly conservative and not really something we want to have
	- ## Spinlocks: Queueing
		- In the approaches before this, everyone has an equal chance to get the lock. So everyone tries for the lock at once
		- We can just make one of them try and get the lock at a time
		- We can assign the priority on FCFS basis
		- So we can just use a queue
		- ## Ticket Lock
			- The comments create an analogy with a ticketing machine
			  ```c
			  int next_ticket, now_serving = 0; // Denotes the ticket machine and the display
			  
			  Lock() {
			    int myticket;
			    // We need to use Fetch&Add because +1 would create race conditions when
			    // 2 processes tug at the same ticket
			    myticket = Fetch&Add(next_ticket, 1);
			    while(myticket != now_serving);
			  }
			  
			  Unlock() { 
			    // We do not need to use fetch&add because now_serving would only be 
			    // updated by the process holding the ticket. And only one process can 
			    // hold the ticket at a time
			    now_serving++;
			  }
			  ```
				- I get invalidated only by the process unlocking
					- But they invalidate everyone by doing so
					- Eventually they will incur a cache miss and go get the value, but it is unnecessary.
				- Everyone reading and writing next_ticket, now_serving (Particularly the latter)
		- ## Array-based Queueing (Anderson Lock)
			- Circular Array of Booleans
			- Code
			  ```c
			  bool flags[P-1] // flags[0] = HAS_LOCK and the rest are MUST_WAIT
			  int tail = 0; // To find out where the end of the queue is for the process to join
			  
			  Lock() {
			    int myplace = Fetch&Add(tail, 1);
			    while(flags[myplace % P] == MUST_WAIT);
			    
			    flags[myplace % P] = MUST_WAIT;
			    return(myplace)
			  }
			  
			  Unlock(myplace) {
			    // We only invalidate (unlock) the array index that we want to address
			    flags[myplace + 1 % P] = HAS_LOCK;
			  }
			  ```
			- We are still trying to update the array addresses
				- Which would take *only one cache line*
				- All these variables are on the same cache line
				- This is called false sharing. The variables are not shared, but since they share the same cache line, they are actually shared
			- One fix is to pad the array
				- One element would be used while the others would be padded with some value
				- This is a waste of space though
			- You have no control over which position you will wait on  On a system without caches,
		- ## Queueing with Linked Lists (MCS Lock)
			- Named after people
			- Queueing structure
			  ```c
			  type queue {
			    next: ^queue;
			    locked: bool
			  }
			  
			  queue ^tail;
			  ```
			- Lock routine will add us to the tail of the queue
			- Normally, 
			  ```c
			  queue = node;
			  tail->next = node;
			  tail = node;
			  ```
			  But there is no atomicity for this 
			  So there can be inconsistency issues
			- test&set or fetch&add won't help here.
			- Many other hardware supports another atomic instruction called `compare&swap(a,b,c)` where:
				- if `a == c`, swap a and b returning `TRUE`
				- or return `FALSE`
				- ```c
				  if a == c {
				    temp = b;
				    b = a;
				    a = temp; 
				    return true;
				  }
				  else {
				    return false;
				  }
				  ```
			- We don't add a dummy node to an empty queue because head would contain the critical section
			- ```c
			  Lock() { 
			    new queue: mynode;
			    pred = mynode; // keep a predecessor node
			    compare&swap(tail, pred, tail); // This is just a swap operation
			    // Tail is now either null or the last element of the linkedList
			    if (pred != null) {
			      tail->next = mynode; // mynode instead of tail because tail can change in this time
			      pred->next = mynode;
			      while(mynode->locked) // We wait until the resource is unlocked
			        ;
			    }
			    /*else: We don't really need to do anything because the queue is null otherwise
			      and the tail would be pointing to us anyways because of compare&swap */  
			  }
			  
			  Unlock(mynode) { 
			    /* The next node could either be null (Either no one is waiting for me or 
			    There is someone who have done compare&swap but have not done Line 8 - in
			    which case, mynode->next would point to null) So we check for mynode->next->locked
			    */
			    if (mynode->next != NULL) {
			      mynode->next->locked = false;
			    }
			    else {
			      /* The case where mynode->next is NULL, we wait for the other node to attach themselves
			      to the tail of the node so we can attach after them */
			      if(compare&swap(tail, NULL, mynode)) {
			        return;
			      }
			      else {
			        while(mynode->next == NULL)
			          ;
			      }
			      mynode->next->locked = false;
			    }
			  }
			  ```
			- This is good because
				- You only look out for your node
				- No false sharing problem