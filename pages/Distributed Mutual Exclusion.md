- Looking at Sync across Distributed Systems
- The Problem
	- Resources must be released before it can be granted to another process
	- Different requests for resource must be granted in the order in which they are made
		- This satisfies the *fairness* criteria of resource sharing (The other being correctness)
		- Fairness prevents starvation queue
	- If every process which is granted the resource eventually releases it, then every request is eventually satisfied
		- Again, starvation is prevented here
-
- ## Centralized Solution
	- There is going to be a central server and multiple nodes
	- The central server grants and revokes resources from the nodes
		- This checks all the boxes, both fairness and correctness
		- But it is a single point of failure and it has severe scalability issues
-
- ## Fully Distributed Solution
	- All processes are symmetric
		- All nodes are treated the same
		- No special treatment for anyone
	- Completely distributed
	- Assumptions:
		- A process can send a message to every other process
			- Fault tolerant solutions are not the focus here
			- We don't worry about delivery, assume there is a transport protocol that guarantees
			- But we also assume that there is no bounded time for this
		- In-order delivery of messages between every pairs of processes
			- If A was sent before B and A also reaches before B
			- Or the case where A reached after B
			- This is not a violation of In-Order processes, since it is still between two processes only
			- However, if a different process C is sent from P1 and is received by P3 before B reaches P2, this is a violation of in-order delivery
		- Every message is eventually received
-
- ## Non-Trivial Problem
	- Let us take 3 processes P1, P2 and P3
	- P1 holds the critical section
	- In a distributed System, we do not know who has the critical section
	- So when the other 2 processes want the critical section, they broadcast and ask for the critical section
	- ### Case 1
		- P3 makes a request, and then P2 makes one
		- Now P2's request reaches P1 before P3. This is not a violation of in-order delivery.
		- It looks simple, but who should P1 service?
			- P1 will give it to P2 because P1 saw that first
			- And we cannot prove P1 otherwise (Lamport clock)
	- ### Case 2
		- If P3 reaches P2
	- ### Case 3
-
- ## Using Lamport Clocks
	- Firstly, we need to ensure that the messages are seen by everyone
		- The messages can take time and unbounded
		- So the receipt of a message is a reply. We wait for a response from the reply
	- ### Lamport's Mutex Algorithm
		- In Lamport clocks, every message has a timestamp
		- When required, a process sends a `REQUEST` message to every other process in the system along with the timestamp
		- When a process receives a `REQUEST` message, it places it on the local request queue and sends back a timestamped `ACK`
			- Each node has it's own queue, not a shared one
			- When a node receives a message, we add it to the queue in the order of the timestamp
			- The node sending the `REQUEST` is not going to be serviced with the critical section until I respond. So we know for sure if the `REQUEST` is serviced or not.
		- $P_i$ gets the resource when:
			- When should a process be serviced with the critical section?
				- The queue is based on timestamps
				- So there might be other `REQUEST`s who have a timestamp earlier
				- The node who made a request might get a response which might have a timestamp lower than the head of the queue
					- In this case, we should make sure that there are no pending request in transit that have already been seen
					- There can't be a service that has yet to be serviced.
					- To check if all the requests are received, we look if we have an ACK from everyone that is greater than the time of the request. By the concept of Lamport Clocks, we assume *in-order delivery*.
			- There is no request in local queue with a lower timestamp than Pi's time of request
			- $P_i$ has received a message from every other process timestamped greater than it's time of request
		- Dequeueing
			- When a process is done with resource, it sends a timestamped `RELEASE` message to everyone
			- When Pi receives a `RELEASE` From Pj, it removes Pj from it's local queue
	- ### Correctness
		- By contradiction
			- TODO Try it out
	- ### Fairness
		- Again by contradiction
		- Assume someone with a lower timestamp got it before someone else
	- This is a fully distributed and symmetric system
	- Complexity
		- In terms of messages, N-1 `REQUEST`, `ACK` and `RELEASE` messages for each time critical message is transferred, for all nodes except me
	- ### No Starvation
		- If it is FCFS, no starvation
	- ### Forward Progress
		- No case where all of them are trying and no one gets the resource
-
- A Centralized solution would have O(1) - One request, one ACK and one RELEASE
- Improving on this idea, can we do better in terms of messages?
	- Could we get rid of the `RELEASE` message?
	- Could we get rid of `ACK`s?
		- We use `ACK` to track if the receiver has seen the message or not.
		- If the sender and receiver both send `ACK`, that is inefficient
		- We can stagger the `ACK` response. Why should we blindly `ACK`? We can hold on to the `ACK` until the sender is done with the critical section and then send it with the `RELEASE` message
-
- ## Ricart and Agrawala Algorithm
	- Cut down 1 round of messages and make it 2 * (N - 1) messages
	- Basic Idea is
		- Send a REQUEST to everyone and get an ACK for everyone and only when you do, send out critical section
		- But send ACK only when you are done with Critical Section
		- The queue is not an explicit data structure here, it is formed implicitly now
	- Algorithm
	  ```c
	  // Lamport clock implementation
	  // highest # chosen by a request from this node
	  int myseqno; 
	  
	  // Seen by this node
	  int highestseqno;
	  
	  // See whether it is requesting a CS
	  bool requesting_cs;
	  
	  enter_mutex() {
	    requesting_cs = true;
	    myseqno = highestseqno + 1;
	    send(REQUEST, myseqno, everyone);
	    // Wait for replies from everyone
	  }
	  
	  // A routine that gets called async whenever a request is invoked
	  recv_req(k, j) {
	    // Called when it gets msg from j with sequence number k
	    highestseqno = max(highestseqno, k);
	    // if myseqno < k, we defer and then reply
	    // But if myseqno = k, then we use lamport clock tiebreaker
	    // Make sure to use the same tiebreaker everywhere
	    if (requesting_cs) &&
	      ((k>myseqno) ||
	      ((k == myseqno) && (j > myid))) {
	      //Defer reply
	    } else {
	      send(REPLY, j)
	    }
	  }
	  
	  exit_mutex() {
	    requesting_cs = FALSE;
	    // for each node i, if you deferred REPLY, send(REPLY, i)
	  }
	  ```
	- ### Proof of Mutual Exclusion
		- Contradiction
	- ### Proof of No Starvation
		- Show FCFS
	- ### Proof of Forward Progress
		- Contradiction
	- The algorithm has a lower bound
		- You cannot do any better than 2 * (N - 1), with the assumption that the algo is distributed and symmetric
-
- We can do improvements based on network assumptions
	- If we have a bus, then all of the nodes have a broadcast medium, which means no of REQUEST  will be O(N)
- Assuming that the algorithm is no longer fully distributed and symmetric
-
- ## Suzuki and Kasami Algorithm
	- This is very similar to token ring networks
	- We relax the assumption that "No node possesses the critical section when it has not been requested"
		- In other words, N - 1 requests will still be going out, but one of the nodes will be doing something different than others
		- If we make this assumption, we can make 2 * (N - 1) to N messages
	- We will denote the node which can use the critical section as having `PRIVILEGE`
	- There are a lot of practical repercussions to this algorithm
		- A lot of networks have been designed with this in mind, we encounter something called 'Token Drain'
		- You get some privilege by a token before you transfer
	- As long as a node has `PRIVILEGE`, it can keep using critical section without asking others
	- Since we do not have any handshake mechanism, how to we ensure that a request coming in has already not been serviced?
		- We maintain a history of requests seen from other nodes, and also a history of requests serviced
	- Algorithm
	  ```c
	  // Both the Data Structures below only make sense on the node having PRIVILEGE
	  // But any node can have PRIVILEGE at any time
	  RN[ 1 ... N]; // holds highest req # seen from every other node
	  bool requesting_cs; 
	  bool have_privilege;
	  
	  // Data Structs passed with Privilege
	  LN[1 ... N]; // Last node that has been serviced by each of the nodes
	  Queue Q; // Of waiting requests
	  
	  // if RN[i] > LN[i], then there is a request that has not been serviced
	   
	  enter_cs() {
	    requesting_cs = true;
	    RN[myid]++;
	    send(REQUEST, RN[myid], everyone);
	    // Wait for PRIVILEGE message
	    have_privilege = true;
	  }
	  recv_req(j,n) {
	    // From node j, with req # n
	    RN[i] = max(RN[j], n);
	    if(have_privilege && !requesting_cs && RN[j] == LN[j] + 1) {
	      have_privilege = false;
	      send(PRIVILEGE, <Q, LN>, j);
	    }
	  }
	    
	  exit_cs() { 
	    LN[myid] = RN[myid];
	    for all i { 
	    if not in Q and RN[i] == LN[i] + 1: 
	    	Append i to Q;
	    }
	    if Q not empty {
	      has_privilege = false;
	      send(PRIVILEGE, <rest of Q, LN>, head(Q));
	    }
	    requesting_cs = false;
	  }
	  ```
	- ### Complexity
		- (N-1) requests + 1 `PRIVILEGE` message per critical section invocation
-
- We have been discussing Distributed System Theory which has some problems
	- Voting Mechanisms for Consensus algos
	- But on reaching Consensus, can the votes be trusted?
	- Can you reach consensus on Fault tolerant Algorithms?
	- Can you reach consensus on a faulty systems?
	- What happens when they lie? (Byzantine Failures?)
	- Unsolved Problems
		- Common Knowledge problems
			- Prisoner's Dilemmas
			- I know that you know that I know that you know that...
	- Read a book by Nancy Lynch