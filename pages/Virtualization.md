## Illusion of Virtual Memory
	- Programs think that they are the only program that is running in the CPU
	- Each program thinks that the CPU, it's registers and memory belongs to it
	- Same thing for IO, etc
-
- ## Hardware Interface vs App Interface
	- There is a hardware with CPU, mem, storage on which OS runs
	- On the OS, Apps run
		- But the application does not actually run on the OS
		- Most of the instructions of the app directly run on the hardware
		- But there is a subset of things that the application does not directly run on the hardware, and needs the OS to run them on -> They are system calls
		- If you really look at it, there is an interface that is exposed by the hardware -> Called the ISA
	- The ISA is directly accessible by the hardware
	- But some of them, is not supposed to be accessed by the user/programmer
		- Like page table manipulation
	- Hardware does not know what the application is or the OS is
		- What it does understand is the bit that says if the instruction is supposed to be run in normal mode or privileged mode
		- TODO To implement any OS, what is the most fundamental feature that you need?
			- User mode and privileged mode at the very least
	- The Application set of instructions to be executed by the OS are called the Application Binary Interface. Which is the set of user instructions and the set of system calls that is allowed for the application
	- The system call is a trap -> Tells that the instruction has to be executed in a privileged mode
	- ### The Issue with this System
		- The problem with this is that the programmer has to obey the interface. There is dependency on the architecture, say x86 or ARM
		- Software needs to run across different hardware -> Needs to persist across changing architectures
		- InterOperability Problem
		  logseq.order-list-type:: number
			- Write-once, Run-anywhere
			  logseq.order-list-type:: number
			- Whether tailored for an OS or an ISA
			  logseq.order-list-type:: number
			- Backward compatability
			  logseq.order-list-type:: number
		- Inflexibility 
		  logseq.order-list-type:: number
			- Cannot change syscalls, OS or ISA
			  logseq.order-list-type:: number
			- Innovation opportunities lost 
			  logseq.order-list-type:: number
			- Let us take malloc for instance
			  logseq.order-list-type:: number
				- malloc has 1 argument - number of bytes, because at that point, we did not care about where the memory was allocated
				- But today we care about locality, proximity etc
				- We are losing flexibility to change, adapt, optimize
			- Multimedia is another example
			  logseq.order-list-type:: number
				- Multimedia relies on bitwise operations
				- Terrible on x86
				- Intel came up with an ISA called MMX
				- But it wasn't popular - because software was already running about it
				- People forgot about MMX
			- All this is because of interface standardization
			  logseq.order-list-type:: number
-
- ## Virtualization
	- We have an interface available -> Can we provide an engineering solution (not a clean one) that provides us with an interface that we want?
	-