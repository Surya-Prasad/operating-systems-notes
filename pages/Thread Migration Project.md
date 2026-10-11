- Parent thread is thread executing pthread_create
- Child Thread is executing foo
- Pointer References:
	- PC
		- Next instruction to be executed
		- Points to LoC
	- Stack Pointer
		- End of active stack
		- Somewhere inside child thread's stack
	- Base Pointer / Frame Pointer
		- Points to address of frame
		- Stable reference point for stack frame
- ucontext_t - snapshot of stack registers and values
- [#A] So we need to find a code base of reference and a stack base of reference
-
- ```bash
  gcc -D_GNU_SOURCE -Wall -Wextra -g app1.c -o app1 -pthread
  
  cp ../cse-511-26fall-thread-migration-group-3/app1 ../client-folder/
  cp ../cse-511-26fall-thread-migration-group-3/app1 ../server-folder/
  
  scp app1 skp6318@e5-cse-135-04.cse.psu.edu:/home/grads/skp6318/socket-test
  
  ./app1 e5-cse-135-03.cse.psu.edu 1
  
  ```
- ## Stack Snapshot
- Source Stack
	- ```bash
	  skp6318@e5-cse-135-03:/scratch/skp6318/operating-systems/client-folder$ ./app1 e5-cse-135-04.cse.psu.edu 0
	  Hostname: e5-cse-135-03.cse.psu.edu
	  Foo: Entry
	  psu_thread_setup_init: sent Stack Pointer: 7f83bdd14e70
	  psu_thread_setup_init: sent PC: 401382
	  psu_thread_setup_init: sent Base Pointer: 7f83bdd14ed0
	  sizeof(ucontext_t) = 968
	  Source stack low: 0x7f83bd516000
	  Source stack size: 8388608
	  Source stack high: 0x7f83bdd16000
	  ```
- Destination Stack
	- ```bash
	  e5-cse-135-04 3$ ./app1 e5-cse-135-03.cse.psu.edu 1
	  psu_thread_setup_init: Started and listening
	  psu_thread_setup_init: Stack Pointer: 7f68a9e26eb0
	  psu_thread_setup_init: PC: 401382
	  psu_thread_setup_init: Base Pointer: 7f68a9e26ed0
	  Destination stack low address: 0x7ffc7b597000
	  Destination stack size: 8380416
	  Destination stack high address: 0x7ffc7bd95000
	  Destination RSP: 7ffc7bd93f50
	  Destination RBP: 7ffc7bd94760
	  ```
- We had difficulty understanding the boundaries of the stack frame and if the return addresses came before or after the top of the frame
-