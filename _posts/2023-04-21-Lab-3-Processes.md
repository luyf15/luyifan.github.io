---
layout: post
read_time: true
show_date: true
title:  JOS Lab 3 - Processes (User Environments)
date:   1997-01-09 22:48:20 +0800
description: JOS Lab 3 - Processes (User Environments)
img: posts/20210125/Perceptron.jpg
tags: [coding, jos, operation system, C]
author: Obsidian0215
github: obsidian0215/jos
mathjax: yes # leave empty or erase to prevent the mathjax javascript from loading
toc: yes # leave empty or erase for no TOC
---

# Lab 3: Processes (User Environments)

```c
divzero: OK (0.9s) 
softint: OK (1.0s) 
    (Old jos.out.softint failure log removed)
badsegment: OK (1.0s) 
Part A score: 30/30

faultread: OK (1.0s) 
faultreadkernel: OK (0.9s) 
faultwrite: OK (1.0s) 
faultwritekernel: OK (0.9s) 
breakpoint: OK (1.0s) 
    (Old jos.out.breakpoint failure log removed)
testbss: OK (1.1s) 
hello: OK (0.9s) 
buggyhello: OK (1.0s) 
buggyhello2: OK (0.9s) 
evilhello: OK (1.0s) 
Part B score: 50/50

Score: 80/80
```

| folder | filename | description |
| --- | --- | --- |
| inc/ | env.h | Public definitions for user-mode environments |
|  | trap.h | Public definitions for trap handling |
|  | syscall.h | Public definitions for system calls from user environments to the kernel |
|  | lib.h | Public definitions for the user-mode support library |
| kern/ | env.h | Kernel-private definitions for user-mode environments |
|  | env.c | Kernel code implementing user-mode environments |
|  | trap.h | Kernel-private trap handling definitions |
|  | trap.c | Trap handling code |
|  | trapentry.S | Assembly-language trap handler entry-points |
|  | syscall.h | Kernel-private definitions for system call handling |
|  | syscall.c | System call implementation code |
| lib/ | Makefrag | Makefile fragment to build user-mode library, obj/lib/libjos.a |
|  | entry.S | Assembly-language entry-point for user environments |
|  | libmain.c | User-mode library setup code called from entry.S |
|  | syscall.c | User-mode system call stub functions |
|  | console.c | User-mode implementations of putchar and getchar, providing console I/O |
|  | exit.c | User-mode implementation of exit |
|  | panic.c | User-mode implementation of panic |
| user/ | * | Various test programs to check kernel lab 3 code |

### Before Implementation:

When executing `make qemu-nox`, JOS crashed for the reason of OUT_OF_MEMORY in *boot_alloc*. It’s caused by the insufficent capacity of 4MB entrypgdir. To solve this issue, we expanded the entrypgdir to 8MB.

```c
__attribute__((__aligned__(PGSIZE)))
pde_t entry_pgdir[NPDENTRIES] = {
	// Map VA's [0, 4MB) to PA's [0, 4MB)
	[0]
		= ((uintptr_t)entry_pgtable - KERNBASE) + PTE_P,
	[1]
		= ((uintptr_t)entry_pgtable_2 - KERNBASE) + PTE_P,
	// Map VA's [KERNBASE, KERNBASE+4MB) to PA's [0, 4MB)
	[KERNBASE>>PDXSHIFT]
		= ((uintptr_t)entry_pgtable - KERNBASE) + PTE_P + PTE_W,
	[(KERNBASE>>PDXSHIFT) + 1]
		= ((uintptr_t)entry_pgtable_2 - KERNBASE) + PTE_P + PTE_W,
};
```

## Part A: User Environments and Exception Handling

In *kern/env.c*, the kernel maintains three main global variables pertaining to environments:

```c
struct Env *envs = NULL;		// All environments
struct Env *curenv = NULL;		// The current env
static struct Env *env_free_list;	// Free environment list
```

Once JOS gets running, the `envs` pointer points to an array of `Env` structures representing **all the environments** in the system. Current JOS kernel will support a maximum of `NENV`(constantly `#define` in *inc/env.h*) simultaneously active environments, although there will typically be far fewer running environments at any time. 

The JOS kernel keeps all of the **inactive** `Env` structures on the `env_free_list`. This design allows easy allocation and deallocation of environments, as they merely have to be added to or removed from the free list.

The kernel uses the `curenv` symbol to keep track of the ***currently executing*** environment at any given time. During boot up, before the first environment is run, `curenv` is initially set to `NULL`.

### Environment State

```c
struct Env {
	struct Trapframe env_tf;	// Saved registers
	struct Env *env_link;		// Next free Env
	envid_t env_id;			// Unique environment identifier
	envid_t env_parent_id;		// env_id of this env's parent
	enum EnvType env_type;		// Indicates special system environments
	unsigned env_status;		// Status of the environment
	uint32_t env_runs;		// Number of times environment has run

	// Address space
	pde_t *env_pgdir;		// Kernel virtual address of page dir

	//... (will expanded in future lab)
};
```

- **Current fields in *struct Env***
    
    **env_tf**: defined in inc/trap.h, holding the **saved register values** for the environment while that environment is *not* running: i.e., when the kernel or a different environment is running. The kernel saves these when switching from user to kernel mode, so that the environment can later be resumed where it left off.
    
    **env_link**: a link to the next `Env` on the `env_free_list`. `env_free_list` points to the first free environment on the list.
    
    **env_id**: The kernel stores here a value that uniquely identifiers the environment currently using this `Env` structure. After a user environment terminates, the kernel may re-allocate the same `Env` structure to a different environment - but the new environment will have a different `env_id` from the old one.
    
    **env_parent_id**: The kernel stores here the `env_id` of the environment that created this environment. In this way the environments can form a “family tree,” which will be useful for making security decisions about which environments are allowed to do what to whom.
    
    **env_type**: This is used to distinguish special environments. For most environments, it will be `ENV_TYPE_USER`. We'll introduce a few more types for special system service environments in later labs.
    
    - **env_status**: This variable holds one of the following values:
        
        `ENV_FREE`: Indicates that the `Env` structure is inactive, and therefore on the `env_free_list`.
        
        `ENV_RUNNABLE`: Indicates that the `Env` structure represents an environment that is waiting to run on the processor.
        
        `ENV_RUNNING`: Indicates that the `Env` structure represents the currently running environment.
        
        `ENV_NOT_RUNNABLE`: Indicates that the `Env` structure represents a currently active environment, but it is not currently ready to run: for example, waiting for an IPC from another environment.
        
        `ENV_DYING`(will not used until Lab 4): Indicates that the `Env` structure represents a zombie environment. A zombie environment will be freed the next time it traps to the kernel. 
        
    
    **env_pgdir**: This variable holds the kernel *virtual address* of this environment's page directory.
    

A JOS environment couples the concepts of "thread" and "address space". The thread is defined primarily by the saved registers (the `env_tf` field), and the address space is defined by the page directory and page tables pointed to by `env_pgdir`. 

**In JOS, individual environments do not have their own kernel stacks as processes do in xv6.** There can be only one JOS environment active in the kernel at a time, so JOS needs only a *single* kernel stack.

### Allocating the Environments Array

In lab 2, you allocated memory in `mem_init()` for the `pages[]` array, which is a table the kernel uses to keep track of which pages are free and which are not. You will now need to modify `mem_init()` further to allocate a similar array of `Env` structures, called `envs`.

- **Ex 1:** Modify `mem_init()` in kern/pmap.c to allocate and map the `envs` array. This array consists of exactly `NENV` instances of the `Env` structure allocated much like how you allocated the `pages` array. Also like the `pages` array, the memory backing `envs` should also be mapped user read-only at `UENVS` (defined in inc/memlayout.h) so user processes can read from this array. You should run your code and make`check_kern_pgdir()` succeed.
    
    ```c
    //kern/pmap.c
    envs = (struct Env *)boot_alloc(sizeof(struct Env) * NENV);
    memset(envs, 0, sizeof(struct Env) * NENV);
    cprintf("envs start at: %.8x\n", envs);
    cprintf("envs end at: %.8x\n", ((char*)envs) + (sizeof(struct Env) * NENV));
    ...
    boot_map_region(kern_pgdir, UENVS, PTSIZE, PADDR(envs), PTE_U);
    ```
    
    **Current issue**: after merging mm_branch with origin/lab3, qemu will crash while running JOS kernel, unless *envs* initialized or not——Possibly caused by the implemention of buddy system.
    
    The following experiments will base on default(firtst-fit) pmm manager.
    

### Creating and Running Environments

You will now write the code in *kern/env.c* necessary to run a user environment. Because we do not yet have a *filesystem*, we will set up the kernel to **load a static binary image that is *embedded within the kernel itself***. **JOS embeds this binary in the kernel as a ELF executable image.**

The Lab 3 *GNUmakefile* generates a number of binary images in the *obj/user/* directory. If you look at *kern/Makefrag*, you will notice some magic that "links" these binaries directly into the kernel executable——**The `-b` binary option on the linker command line causes these files to be linked in as "raw" uninterpreted binary files** rather than as regular .o files produced by the compiler. (As far as the linker is concerned, these files do not have to be ELF images at all - they could be anything! ) If you look at *obj/kern/kernel.sym* after building the kernel, you will notice that the linker has "magically" produced a number of funny symbols with names like `_binary_obj_user_hello_start`, `_binary_obj_user_hello_end`, `_binary_obj_user_hello_size`. The linker generates these symbol names by mangling the file names of the binary files; the symbols provide the regular kernel code with a way to reference the embedded binary files.

```bash
//kern/Makefrag
# Binary program images to embed within the kernel.
# Binary files for LAB3
**KERN_BINFILES** :=	user/hello \
			user/buggyhello \
			user/buggyhello2 \
			user/evilhello \
			user/testbss \
			user/divzero \
			user/breakpoint \
			user/softint \
			user/badsegment \
			user/faultread \
			user/faultreadkernel \
			user/faultwrite \
			user/faultwritekernel

# How to build the kernel itself
$(OBJDIR)/kern/kernel: $(KERN_OBJFILES) $(KERN_BINFILES) kern/kernel.ld \
	  $(OBJDIR)/.vars.KERN_LDFLAGS
	@echo + ld $@
	$(V)$(LD) -o $@ $(KERN_LDFLAGS) $(KERN_OBJFILES) $(GCC_LIB) **-b binary $(KERN_BINFILES)**
	$(V)$(OBJDUMP) -S $@ > $@.asm
	$(V)$(NM) -n $@ > $@.sym
```

In `i386_init()` in kern/init.c you'll see code to run one of these binary images in an environment. However, the critical functions to set up user environments need to be complete.

**Ex 2:** In the file *kern/env.c*, finish coding the following functions:

```c
env_init()
//Initialize all of the Env structures in the envs array and add them to the env_free_list. 
env_setup_vm()
//Allocate a page directory for a new environment and initialize the kernel portion of the new environment's address space.
region_alloc()
//Allocates and maps physical memory for an environment
load_icode()
//Parse an ELF binary image, much like the boot loader already does, 
//and load its contents into the user address space of a new environment.
env_create()
//Allocate an environment with env_alloc and call load_icode to load an ELF binary into it.
env_run()
//Start a given environment running in user mode.
```

The new cprintf verb %e prints a description corresponding to an error code. 

```c
r = -E_NO_MEM;
panic("env_alloc: %e", r);
```

will panic with the message "env_alloc: out of memory".

Below is a call graph of the code up to the point where the user code is invoked. 

- `start` (*kern/entry.S*)
- `i386_init` (*kern/init.c*): `cons_init`, `mem_init`, `env_init`, `trap_init` (still incomplete at this point), `env_create`, `env_run`, `env_pop_tf`

**Result:** 

**In `*env.c*`:** 

```c
void
env_init(void)
{
	  // Set up envs array
    int i;
    struct Env *e;

    env_free_list = NULL;
    for (i = NENV - 1; i >= 0; --i) {
        e = &envs[i];
        e->env_id = 0;
        e->env_status = ENV_FREE;
				// the environments in the free_list should be in the same order in envs,
				// so that the first call to env_alloc() returns envs[0].
        e->env_link = env_free_list;
        env_free_list = e;
    }
	// Per-CPU part of the initialization
	env_init_percpu();
}

static int
env_setup_vm(struct Env *e)
{
int i;
	struct Page *p = NULL;

	// Allocate a page for the page directory
	if (!(p = alloc_page(ALLOC_ZERO)))
		return -E_NO_MEM;

	//  The VA space of all envs is identical above UTOP (except at UVPT)
	//  Only env_pgdir's pp_ref need to be incremented for env_free to work correctly.
	e->env_pgdir = page2kva(p);
	page_ref_inc(p);

	for (size_t i = PDX(UTOP); i < NPDENTRIES; ++i)
        e->env_pgdir[i] = kern_pgdir[i];
	// UVPT maps the env's own page table read-only: kernel R, user R
	e->env_pgdir[PDX(UVPT)] = PADDR(e->env_pgdir) | PTE_P | PTE_U;

	return 0;
}

static void
region_alloc(struct Env *e, void *va, size_t len)
{
	// Watch out for corner-cases
	if ((uintptr_t)va >= UTOP)
	    panic("region_alloc(1): Unavailable virtual address for user environment");

	uintptr_t vstart, vend;
  struct Page *p;
  int err;

	//  'va' and 'len' values that are not page-aligned should be allowed.
	//  You should round va down, and round (va + len) up.
  vstart = ROUNDDOWN((uintptr_t)va, PGSIZE);
  vend = ROUNDUP((uintptr_t)va + len, PGSIZE);

  for (; vstart < vend; vstart += PGSIZE) {
      if (!(p = alloc_page(ALLOC_ZERO)))
          panic("region_alloc(2): page allocation failed");

		  // Pages should be writable by user and kernel.
      if ((err = page_insert(e->env_pgdir, p, (void*)vstart, PTE_W | PTE_U)) < 0)
          panic("region_alloc(3): %e", err);
    }
}

static void
load_icode(struct Env *e, uint8_t *binary)
{
	struct Elf *eh = (struct Elf *)binary;
	if (eh->e_magic != ELF_MAGIC)
	    panic("load_icode(1): valid ELF magic number");

	lcr3(PADDR(e->env_pgdir));
	struct Proghdr *ph = (struct Proghdr *)(binary + eh->e_phoff), *ph_end = ph + eh->e_phnum;
	for (; ph < ph_end; ++ph) {
        if (ph->p_type != ELF_PROG_LOAD)
            continue;
		
        if (ph->p_memsz < ph->p_filesz)
            panic("load_icode(2): ELF illegal file and mem size");

        // firstly allocate space (mapping)
        region_alloc(e, (void *)ph->p_va, ph->p_memsz);
        // copy to virtual address and set the rest to 0
        memcpy((void *)ph->p_va, binary + ph->p_offset, ph->p_filesz);
        memset((void *)(ph->p_va + ph->p_filesz), 0, ph->p_memsz - ph->p_filesz);
  }
  e->env_status = ENV_RUNNABLE;
	// set executing entry(in trapframe)
	e->env_tf.tf_eip = eh->e_entry;

	// Now map one page for the program's initial stack
	// at virtual address USTACKTOP - PGSIZE.
	region_alloc(e, (void *)(USTACKTOP - PGSIZE), PGSIZE);
	// switch back to kernel address mapping
  lcr3(PADDR(kern_pgdir));
}

void
env_create(uint8_t *binary, enum EnvType type)
{
	struct Env *e;
	int err;
	if ((err = env_alloc(&e,0)) < 0)
		panic("env_create: %e", err);

	load_icode(e, binary);
	e->env_type = type;
}

void
env_pop_tf(struct Trapframe *tf)
{
	asm volatile(
		"\tmovl %0,%%esp\n"
		"\tpopal\n"
		"\tpopl %%es\n"
		"\tpopl %%ds\n"
		"\taddl $0x8,%%esp\n" /* skip tf_trapno and tf_errcode */
		"\tiret\n"			/* ip,cs,flags,sp,ss */
		: : "g" (tf) : "memory");
	panic("iret failed");  /* mostly to placate the compiler */
}

void
env_run(struct Env *e)
{
	if (curenv && curenv->env_status == ENV_RUNNING)
		curenv->env_status = ENV_RUNNABLE;
	curenv = e;
	curenv->env_status = ENV_RUNNING;
	curenv->env_runs += 1;
	lcr3(PADDR(curenv->env_pgdir));
	env_pop_tf(&curenv->env_tf);
}
```

**So far, `i386_init` can execute *user_hello* in user environment, invoking a *triple fault* caused by the incomplete trap(syscall) handler(later implemented in `trap_init`): After `*int $0x30*` in `sys_cputs`(in *hello*) executed, $eip will jump to an invalid address 0xe05b.**

![Untitled](./assets/img/posts/Lab3-Processes/Untitled.png)

### Handling Interrupts and Exceptions

The first `*int $0x30*`syscall instruction in user space is a dead end so far: once the processor gets into user mode, there is no way to get back out. You will now need to implement basic exception and system call handling, so that it is possible for the kernel to recover control of the processor from user-mode code. (The x86 interrupt and exception mechanism is [here](Debug-JOS.html).)

### Protected Control Transfer

Some events which cause the processor to switch from user to kernel mode (CPL=0) without giving the user-mode code any opportunity to interfere with the functioning of the kernel or other environments——**interrupt** (caused by external(devices) asynchronous events) / **exception** (synchronous events while executing code)

On the x86, two mechanisms work together to provide the PCT’s protection:

- **IDT**: 256 (interrupt and exception) **handler entrypoints** determined by the kernel. Index 0-255 interrupt vector: the ***entrypoint virt-address*** of each handler + the value to load into ***CS***.
- **TSS**: the old processer state(SS, esp, EFLAGS, CS, eip...) will be stored in a protected memory space before handling PCT, and the new processer state(SS, esp) can be loaded from TSS to protect this procedure from unexpected user-mode codes.
JOS kernel only uses TSS to store the ***kernel stack***(when the processer switch from user to kernel mode)——only ***ESP0***, ***SS0*** fields in TSS be used.

### Types of Exceptions and Interrupts

All of the synchronous exceptions that the x86 processor can generate internally use interrupt vectors between 0 and 31, and therefore map to IDT entries 0-31. 
e.g. page fault: vector 14; software interrupts(`*int*`) and external interrupts: vectors greater than 31.

- **synchronous exceptions example: divided by 0**
    1. The processor switches to the kernel stack defined by the SS0 and ESP0 fields in TSS, which in JOS will hold the values GD_KD and KSTACKTOP, respectively.
    2. The processor pushes the exception frame on the kernel stack, starting at address KSTACKTOP:
    
    ```
                         +--------------------+ KSTACKTOP
                         | 0x00000 | old SS   |     " - 4
                         |      old ESP       |     " - 8
                         |     old EFLAGS     |     " - 12
                         | 0x00000 | old CS   |     " - 16
                         |      old EIP       |     " - 20 <---- ESP
    										(|     error code     |     " - 24 <---- ESP) ;e.g.page fault
                         +--------------------+
    ```
    
    1. Since a divide error being handled, which is interrupt vector 0 on the x86, the processor reads IDT entry 0 and sets CS:EIP to point to the handler function described by the entry.
    2. The handler function takes control and handles the exception.(e.g. terminate the user process)
- **Nested Exceptions and Interrupts**
    
    If the processor is already in kernel mode when the interrupt or exception occurs, then the pushed values will be on the same kernel stack. (nested exceptions that can provide protection in syscall)
    
    If the processor is already in kernel mode and takes a nested exception, since it does not need to switch stacks, it does not save the old SS or ESP registers. For exception types that do not push an error code, the kernel stack therefore looks like the following on entry to the exception handler:
    
    ```
    											+--------------------+ <---- old ESP
                          |     old EFLAGS     |     " - 4
                          | 0x00000 | old CS   |     " - 8
                          |      old EIP       |     " - 12
    										 (|     error code     |     " - 16 <---- ESP) ;e.g.page fault
                          +--------------------+
    ```
    
    **A possible fatal error**: Several nested exceptions can make the kernel stack lack of space, then there is nothing the processor can do to recover, so the kernel simply resets. Needless to say, the kernel should be designed so that this can't happen.
    

### Setting Up the IDT

The header files `*inc/trap.h*` and *`kern/trap.h`* contain important definitions related to interrupts and exceptions that you will need to become familiar with. The file `*kern/trap.h*` contains definitions that are strictly private to the **kernel**, while `*inc/trap.h*` contains definitions that may also be useful to **user-level programs and libraries**.

Now the IDT can be setting up and thus JOS can handle exceptions. Set up the IDT to handle **interrupt vectors 0-31 (the processor exceptions, some of them are reserved by Intel)**. We'll handle **system call interrupts** later in this lab and add **interrupts 32-47 (the device IRQs)** in a later lab.

The overall flow of exception control: 

```c
IDT                          trapentry.S         trap.c    
+----------------+           (handler entry)     (handle or dispatch traps)         
|   &handler1    |---------> handler1:          trap (struct Trapframe *tf) 
|                |             // do stuff      { 
|                |             call trap          // handle the exception/interrupt 
|                |             // ...           } 
+----------------+ 
       ...
+----------------+ 
|   &handlerX    |--------> handlerX: 
|                |             // do stuff 
|                |             call trap 
|                |             // ...
+----------------+
```

**Ex4 & Challenge**: Edit `*trapentry.S*` and `*trap.c*` and implement the features described above. The macros `TRAPHANDLER` and `TRAPHANDLER_NOEC` in trapentry.S should help you, as well as the T_* defines in `*inc/trap.h*`. You will need to add an entry point in `*trapentry.S*` (using those macros) for each trap defined in `*inc/trap.h*`, and you'll have to provide `_alltraps` which the `TRAPHANDLER` macros refer to. You will also need to modify `trap_init()` to initialize the `idt` to point to each of these entry points defined in `*trapentry.S`.*

`_alltraps` should do as follows:

1. push values to make the stack look like a `struct Trapframe`
2. load `GD_KD` into `%ds` and `%es`
3. `pushl %esp` to pass a pointer to the Trapframe as an argument to `trap()`
4. `call trap` (`trap` WON’T return)

Using the `pushal` instruction fits nicely with the layout of the `struct Trapframe`.

Test trap handlers by using some of the *user* programs that cause exceptions before making any system calls, e.g. `*user/divzero*`. You should be able to get **make grade** to succeed on the divzero, softint, and badsegment tests at this point.

*Challenge!* Change the macros in `*trapentry.S*` to automatically generate a table for `*trap.c*` to use. Note that you can switch between laying down code and data in the assembler by using the directives `.text` and `.data`.

**Add `*kern/trapvector.inc*`, including all the implementations of trap handlers as a flexible macro.**

```c
TH(0)
TH(1)
TH(2)
TH(3)
TH(4)
TH(5)
TH(6)
TH(7)
THE(8)
THE(9)
THE(10)
THE(11)
THE(12)
THE(13)
THE(14)
TH(15)
TH(16)
THE(17)
TH(18)
TH(19)
...
TH(48)
```

**In `*trapentry.S*`: (Reference—ucore_os_plus)**

```
#define TH(n) TRAPHANDLER_NOEC(handler##n, n)
#define THE(n) TRAPHANDLER(handler##n, n)
# entrypoints of each interrupt vector(handler)

#include <kern/trapvector.inc>

_alltraps:
	# push registers to build a trap frame 
	# therefore make the stack look like a struct trapframe
	pushl %ds
	pushl %es
	pushal

	# load GD_KD into %ds, %es
	movw $(GD_KD), %ax
	movw %ax, %ds
	movw %ax, %es

	# pass a pointer to the trap frame for function trap  
	pushl %esp
	call trap
spin_trap:
	jmp spin_trap
```

**In `*trap.c*`:** 

```c
#define TH(n) extern void handler##n (void);
#define THE(n) TH(n)

#include <kern/trapvector.inc>

#undef THE
#undef TH

#define TH(n) [n] = handler##n,
#define THE(n) TH(n)

static void (*handlers[256])(void) = {
#include <kern/trapvector.inc>
};

#undef THE
#undef TH
void
trap_init(void)
{
	extern struct Segdesc gdt[];

	// LAB 3: Your code here.
	for (int i = 0; i < 32; ++i)    
		SETGATE(idt[i], 0, GD_KT, handlers[i], 0);
	SETGATE(idt[T_BRKPT], 0, GD_KT, handlers[T_BRKPT], 3);
	SETGATE(idt[T_SYSCALL], 0, GD_KT, handlers[T_SYSCALL], 3);
	// Per-CPU setup 
	trap_init_percpu();
}
```

**Result: When the user program is user_divzero, user_badsegment, which can make kernel invoke an exception, we can get the following output.**

![Untitled](./assets/img/posts/Lab3-Processes/Untitled%201.png)

**In the output snapshot, we can know that the user_divzero triggered a divide-zero exception. The exception handler eventually destroyed the user environment.**

- Questions:
    1. What is the purpose of having an individual handler function for each exception/interrupt? (i.e., if all exceptions/interrupts were delivered to the same handler, what feature that exists in the current implementation could not be provided?)
    **To provide a uniform struct TrapFrame since some exceptions will push an extra error code.**
    2. Did you have to do anything to make the `*user/softint*` program behave correctly? The grade script expects it to produce a general protection fault (trap 13), but `*softint*`'s code says `int $14`. *Why* should this produce interrupt vector 13? What happens if the kernel actually allows `*softint*`'s `int $14` instruction to invoke the kernel's page fault handler (which is interrupt vector 14)?
    **The interrupt vector 14 have the DPL of 0, but we executed `int $14` in the user-mode(CPL=3). If `int $14` can be invoked by softint, the pagetable will be difficult to maintain by casually being modified in user-mode.**

## Part B: Page Faults, Breakpoints Exceptions, and System Calls

### Handling Page Faults

The page fault exception, interrupt vector 14 (T_PGFLT): When the processor takes a page fault, it stores the linear (virtual) address that caused the fault in CR2. In *`trap.c`* we have provided the beginnings of a special function, page_fault_handler(), to handle page fault exceptions.

- **Ex5**: Modify `trap_dispatch()` to dispatch page fault exceptions to `page_fault_handler()`. You should now be able to get make grade to succeed on the `*faultread*`, `*faultreadkernel*`, `*faultwrite*`, and `*faultwritekernel*` tests. If any of them don't work, figure out why and fix them. (JOS can boot into a particular user program using `make run-x` or `make run-x-nox`now)
    
    **In trap.c:** 
    
    ```c
    //trap_dispatch()
    switch (tf->tf_trapno){
    	case T_PGFLT:
    		page_fault_handler(tf);
    		return;
    }
    ```
    
    **Result: 4 user programs above can all be trapped into kernel, with correct fault reasons.**
    
    ![Untitled](./assets/img/posts/Lab3-Processes/Untitled%202.png)
    

### The Breakpoint Exception

The breakpoint exception, interrupt vector 3 (T_BRKPT), is normally used to allow debuggers to insert breakpoints in a program's code by temporarily replacing the relevant program instruction with **`0xCC`**. In JOS we will abuse this exception slightly by turning it into a primitive pseudo-system call that any user environment can use to invoke the JOS kernel monitor (as a primitive debugger). E.g. the user-mode `panic()` in `*lib/panic.c*`, performs an int3 after displaying the message.

- **Ex6**: Modify `trap_dispatch()` to make breakpoint exceptions invoke the kernel monitor. You should now be able to get `make grade` to succeed on the breakpoint test.
    
    **In `*trap.c*`:** 
    
    ```c
    //trap_dispatch()
    switch (tf->tf_trapno){
    		case T_DEBUG:
    		case T_BRKPT:
    			breakpoint_handler(tf);
    			return;
    }
    ```
    
    **Result: The breakpoint test passed.**
    
    ![Untitled](./assets/img/posts/Lab3-Processes/Untitled%203.png)
    
- *Challenge!* Modify the JOS kernel monitor so that you can 'continue' execution from the current location (e.g., after the int3, if the kernel monitor was invoked via the breakpoint exception), and so that you can single-step one instruction at a time. You will need to understand certain bits of the EFLAGS register in order to implement single-stepping.
    
    **E/RFLAGS.TF flag can make the processer run in single-step mode.**
    
    **In `*monitor.c*`:** 
    
    ```c
    static struct Command commands[] = {
    	{ "step", "", mon_step},
    	{ "s", "", mon_step},
    	{ "continue", "", mon_continue},
    	{ "c", "", mon_continue},
    };
    
    int
    mon_step(int argc, char **argv, struct Trapframe *tf)
    {
    	// if not user-source bp/db interrupt, do nothing
    	if (!(tf && (tf->tf_trapno == T_DEBUG || tf->tf_trapno == T_BRKPT) && 
              ((tf->tf_cs & 3) == 3)))
            return 0;
        tf->tf_eflags |= FL_TF;
        return -1;
    }
    
    int
    mon_continue(int argc, char **argv, struct Trapframe *tf)
    {
    	// if not user-source bp/db interrupt, do nothing
    	if (!(tf && (tf->tf_trapno == T_DEBUG || tf->tf_trapno == T_BRKPT) && 
              ((tf->tf_cs & 3) == 3)))
            return 0;
        tf->tf_eflags &= ~FL_TF;
        return -1;
    }
    ```
    
    **Result:** 
    
    - **If the kmonitor executes step/s: bit 8 of EFLAGS (TF) is set, the next instruction is `ret` which will cause the processor back to kernel.**
        
        ![Untitled](./assets/img/posts/Lab3-Processes/Untitled%204.png)
        
    - **If the kmonitor executes continue/c: bit 8 of EFLAGS (TF) is clear. The kmonitor exits and the control flow will be back to kernel until the user process destroyed.**
        
        ![Untitled](./assets/img/posts/Lab3-Processes/Untitled%205.png)
        
    
    *****When testing the `backtrace` command containing user stack, the kernel will catch a `page-fault` exception with `ebp=0xdebfdff0`(within `*the user stack:top=0xdebfe000*`). This issue is caused by the implementation of arguments displaying in mon_backtrace function:** 
    
    ```c
    int
    mon_backtrace(int argc, char **argv, struct Trapframe *tf)
    {
    	uint32_t ebp = read_ebp();
    	while (ebp){
    		**int *args = (int *)ebp + 2;
    		for (int i = 0; i < 4; i++)**
    			cprintf(" %08.x ", args[i]);
    		ebp = *(uint32_t *)ebp;
    		//count++ ;
    	}
    }
    ```
    
    **When `mon_backtrace` walks through the user stack, it will access the address over the USTACKTOP, which is settled as an unmapped guard-page. Thus a `page-fault`exception will be throwed out by the processor.**
    
    ```c
    //inc/memlayout.h
    
    // Top of user-accessible VM, 0xDEC00000
    #define UTOP		UENVS
    // Top of one-page user exception stack
    #define UXSTACKTOP	UTOP
    // Next page left invalid to guard against exception stack overflow; then:
    // Top of normal user stack, 0xDEBFE000
    #define USTACKTOP	(UTOP - 2*PGSIZE)
    ```
    

*Optional*: Find some x86 **disassembler** source code - e.g., QEMU, or GNU binutils, or just write it yourself - and extend the JOS kernel monitor to be able to disassemble and display instructions as you are stepping through them. Combined with **the symbol table** from lab1, this is the stuff of which real kernel debuggers are made.

- Questions:
    1. The breakpoint test case will either generate a #BP exception or a general protection fault depending on how you initialized the breakpoint entry in the IDT (i.e. SETGATE calls in `trap_init`). Why? How do you need to set it up in order to get the breakpoint exception to work as specified above and what incorrect setup would cause it to trigger a general protection fault?
        
        **The breakpoint user program uses `int $3` to invoke a #BP exception, which needs the  no.3 vector’s DPL=3. Otherwise the instruction will throw out a #GP exception.**
        
    2. What do you think is the point of these mechanisms, particularly in light of what the user/softint test program does?
        
        **To export some interfaces for users to use system services and meantime prevent unexpected users from abusing exceptions.**
        

### System calls

User processes ask the kernel to do things for them by invoking system calls. When the user process invokes a system call, the processor enters kernel mode, the processor and the kernel cooperate to save the user process's state, the kernel executes appropriate code in order to carry out the system call, and then resumes the user process. 

The JOS kernel will use the `int` instruction(`int $0x30`), which causes a processor interrupt, as the system call interrupt. **Note that interrupt 0x30 cannot be generated by hardware, so there is no ambiguity caused by allowing user code to generate it.——set int vector 0x30 DPL=3**

The caller will pass the system call number and the system call arguments in registers, which avoids  grub around within the user environment. The system call number will go in `%eax`, and the arguments (up to five of them) will go in `%edx`, `%ecx`, `%ebx`, `%edi`, and `%esi`, respectively. The kernel passes the return value back in `%eax`. The assembly code to invoke a system call has been written for you, in `syscall()` in lib/syscall.c. You should read through it and make sure you understand what is going on.

- **Ex7**: Add a handler in the kernel for interrupt vector T_SYSCALL. 
Functions needed to edit: `*kern/trapentry.S`,* `*kern/trap.c*`'s `trap_init(),trap_dispatch()`handle the system call interrupt by `syscall()` in `*kern/syscall.c*` with the appropriate arguments, with return value in `%eax` to be passed back to the user process. 
Make sure `syscall()` returns `**-E_INVAL**` if the system call number is invalid. You should read and understand the system call interface in `*lib/syscall.c*`. Handle all the system calls listed in `*inc/syscall.h*` by invoking the corresponding kernel function for each call.
Test case: Run the `*user/hello*` program under your kernel (make run-hello), which will print "hello, world" and then cause a page fault in user mode. Besides,`*testbss*`test now can succeed.
    
**In `*trap.c*`:**
    
```c
    //trap_init
    SETGATE(idt[T_SYSCALL], 0, GD_KT, handlers[T_SYSCALL], 3);
    
    //trap_dispatch()
    switch(tf->tf_trapno) {
    	case T_SYSCALL:
    	// eax, edx, ecx, ebx, edi, esi
    		struct PushRegs *r = &tf->tf_regs;
    		r->reg_eax = syscall(r->reg_eax,
    											r->reg_edx,
    											r->reg_ecx,
    											r->reg_ebx,
    											r->reg_edi,
    											r->reg_esi);
    		return;
    }
```
    
**In `*kern/syscall.c*`:**
    
```c
    int32_t
    syscall(uint32_t syscallno, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
    {
    	switch (syscallno) {
    		case SYS_cputs: {
    			sys_cputs((const char *)a1, (size_t)a2);
    			return 0;
        	}
    		case SYS_cgetc:
    			return sys_cgetc();
    		case SYS_env_destroy:
    			return sys_env_destroy((envid_t)a1);
    		case SYS_getenvid:
    			return sys_getenvid();
    		case NSYSCALLS:
    		default:
    			return -E_INVAL;
    	}
    }
```
    
**Result: The user page fault of following user programs  is caused by illegal access to kernel variables or functions.**
    
- **run `*user/hello*`: the “hello, world” string has been printed on the console.**
        
![Untitled](./assets/img/posts/Lab3-Processes/Untitled%206.png)
        
- **run `*user/testbss*`:**
        
    **I got a great series of page-fault exception, and then the kernel was stucked. When I debugged the user-environment initialization, I found that when the kernel tried to allocate memories of the .data segment, the kernel would get page-faults at va 0x00000048 ?? Deeping into the gva2gpa mapping got unmapped pa ≥0xa0000(`IOPHYSMEM`). I speculated there are some mistakes in the initial memmap and I figured out that the reserved kernel memory got a wrong range which would make the envs array accidentally modified by users. After correcting the `init_memmap`, `*user/testbss*` can print its messages normally.**
        
    ![Untitled](./assets/img/posts/Lab3-Processes/Untitled%207.png)
        
- ***Challenge!*** Implement system calls using `sysenter` and `sysexit` instructions (faster) instead of using `int 0x30` and `iret`. 
	When executing a `sysenter` instruction, the processor does not save state information for the user code (e.g., e/rip), and neither `sysenter` nor `sysexit` supports passing parameters on the stack, instead by registers——
    
    ```
    	eax                - syscall number
    	edx, ecx, ebx, edi - arg1, arg2, arg3, arg4(arg5 is dropped)
    	esi                - return pc
    	ebp                - return esp
    	esp                - trashed by sysenter
    ```
    
    Note: 1) The old method of system calls support 5 arguments while fast ways only need 4 args. 2) Because this fast path doesn't update the current environment's trap frame, it won't be suitable for all of the system calls. 3) It’s necessary to enable interrupts when returning to the user process, which `sysexit` doesn't do it.
    
    **Firstly, to use sysenter correctly, MSRs IA32_SYSENTER_CS/EIP/ESP must be set to the new `sysenter_handler` in `*kern/trapentry.S*`. Add `wrmsr/rdmsr` support in `*inc/x86.h*` and initialize relevant MSRs in `*kern/init.c*`.**
    
    **In `*kern/trapentry.S*`:** 
    
    ```c
    .globl sysenter_handler
     sysenter_handler:
    
    	# prepare arguments for syscall
    	pushl $0x0 # only four parameters supported, 0x0 for the fifth
    	pushl %edi # arg4
    	pushl %ebx # arg3
    	pushl %ecx # arg2
    	pushl %edx # arg1
    	pushl %eax # sysno
    	call syscall
    
    	# pepare user space information to restore
    	movl %esi,%edx # return eip
    	movl %ebp,%ecx # user space esp
    	sysexit
    ```
    
    **In `*inc/x86.h*`:** 
    
    ```c
    /* for fast syscall */
    #define IA32_SYSENTER_CS (0x174)
    #define IA32_SYSENTER_EIP (0x176)
    #define IA32_SYSENTER_ESP (0x175)
    
    #define rdmsr(msr,d_val,a_val) \
    	__asm__ __volatile__("rdmsr" \
    	: "=a" (a_val), "=d" (d_val) \
    	: "c" (msr))
    
    #define wrmsr(msr,d_val,a_val) \
    	__asm__ __volatile__("wrmsr" \
    	: /* no outputs */ \
    	: "c" (msr), "a" (a_val), "d" (d_val))
    ```
    
    **In `*kern/init.c*`:**
    
    ```c
    static void msr_init(){
    	extern void sysenter_handler();
    	uint32_t cs;
    	asm volatile("movl %%cs,%0":"=r"(cs));
    	wrmsr(IA32_SYSENTER_CS,0x0,cs);
    	wrmsr(IA32_SYSENTER_EIP,0x0,sysenter_handler);
    	wrmsr(IA32_SYSENTER_ESP,0x0,KSTACKTOP);
    }
    
    void
    i386_init(void)
    {
    	...
    	env_init();
    	trap_init();
    	msr_init();
    	... //user_process init
    }
    ```
    
    **In `*lib/syscall.c*`:**
    
    ```c
    // shouldn't define as an inline function
    static int32_t
    fast_syscall(int num, int check, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
    {
    	int32_t ret;
    
    	// move arguments into registers before %ebp modified
    	// as with static identifier, num, check and a1 are stroed
    	// in %eax, %edx and %ecx respectively, while a3, a4 and a5
    	// are addressed via 0xx(%ebp)
    	asm volatile("movl %0,%%edx"::"S"(a1):"%edx");
    	asm volatile("movl %0,%%ecx"::"S"(a2):"%ecx");
    	asm volatile("movl %0,%%ebx"::"S"(a3):"%ebx");
    	asm volatile("movl %0,%%edi"::"S"(a4):"%ebx");
    
    	// save userspace %esp in %ebp passed into sysenter_handler
    	asm volatile("pushl %ebp");
    	asm volatile("movl %esp,%ebp");
    
    	// save userspace %eip in %esi passed into sysenter_handler
    	asm volatile("leal .after_sysenter_label,%%esi":::"%esi");
    	asm volatile("sysenter \n\t"
    				 ".after_sysenter_label:"
    			:: "a" (num)
    			: "memory");
    	
    	// retrieve return value
    	asm volatile("movl %%eax,%0":"=r"(ret));
    	
    	// restore %ebp
    	asm volatile("popl %ebp");
    
    	if(check && ret > 0)
    		panic("syscall %d returned %d (> 0)", num, ret);
    
    	return ret;
    }
    
    sys_* functions call fast_syscall rather than syscall
    ```
    
    **Issue: In `fast_syscall`, if the `%esp` saving operation is before args movement, the `fast_syscall` will perform a wrong result, resulting from args being read from stacks(`%ebp`) while the value of `%ebp` been changed.**  
    
    ```c
    	800c84:	55                   	push   **%ebp**
      800c85:	89 e5                	**mov    %esp,%ebp**
      800c87:	57                   	push   %edi
      800c88:	56                   	push   %esi
      800c89:	53                   	push   %ebx
      800c8a:	83 ec 1c             	sub    $0x1c,%esp
      800c8d:	e8 e6 f3 ff ff       	call   800078 <__x86.get_pc_thunk.bx>
      800c92:	81 c3 6e 13 00 00    	add    $0x136e,%ebx
      800c98:	89 5d e4             	mov    %ebx,-0x1c(%ebp)
      800c9b:	89 d7                	mov    %edx,%edi
      800c9d:	89 ce                	mov    %ecx,%esi
    	// The "volatile" tells the assembler not to optimize
    	// this instruction away just because we don't use the
    	// return value.
    
    	// save user space %esp in %ebp passed into sysenter_handler
    	asm volatile("pushl %ebp");
      800c9f:	55                   	push   **%ebp**
    	asm volatile("movl %esp,%ebp");
      800ca0:	89 e5                	**mov    %esp,%ebp**
    
    	// move arguments into registers before %ebp modified
    	// as with static identifier, num, check and a1 are stroed
    	// in %eax, %edx and %ecx respectively, while a3, a4 and a5
    	// are addressed via 0xx(%ebp)
    	asm volatile("movl %0,%%edx"::"S"(a1):"%edx");
      800ca2:	89 f2                	mov    %esi,%edx
    	asm volatile("movl %0,%%ecx"::"S"(a2):"%ecx");
      800ca4:	8b 75 08             	mov    **0x8(%ebp)**,%esi
      800ca7:	89 f1                	mov    %esi,%ecx
    	asm volatile("movl %0,%%ebx"::"S"(a3):"%ebx");
      800ca9:	8b 75 0c             	mov    **0xc(%ebp)**,%esi
      800cac:	89 f3                	mov    %esi,%ebx
    	asm volatile("movl %0,%%edi"::"S"(a4):"%ebx");
      800cae:	8b 75 10             	mov    **0x10(%ebp)**,%esi
      800cb1:	89 f7                	mov    %esi,%edi
    ```
    

### **User-mode startup**

A user program starts running at the top of `*lib/entry.S*`. After some setup, this code calls `libmain()`, in `*lib/libmain.c*`. `libmain()` then calls `umain`. You should modify libmain() to initialize the global pointer `thisenv` to point at this environment's `struct Env` in the `envs[]` array. 

- **Ex8:** Add the required code to the user library by modifying `libmain()` to initialize the global pointer `thisenv` to point at this environment's `struct Env` in the `envs[]` array. . You should see `*user/hello print*` "hello, world" and then print "i am environment 00001000". user/hello then attempts to "exit" by calling `sys_env_destroy()` (see lib/libmain.c and lib/exit.c). Since the kernel currently only supports one user environment, it should report that it has destroyed the only environment and then drop into the kernel monitor. You should be able to get **make grade** to succeed on the hello test.Hint: look in inc/env.h and use `sys_getenvid`.
    
    **In `*lib/libmain.c*`:** 
    
    ```c
    void
    libmain(int argc, char **argv)
    {
    	// set thisenv to point at our Env structure in envs[].
    	thisenv = &envs[ENVX(sys_getenvid())];
    	...
    }
    ```
    
    **Result: user_hello now can print its PID.**
    
    ![Untitled](./assets/img/posts/Lab3-Processes/Untitled%208.png)
    

### Page faults and memory protection

Memory protection is a crucial feature of an operating system, ensuring that bugs in one program cannot corrupt other programs or corrupt the operating system itself.

The OS maintains the page tables to inform the processor about which virtual addresses are valid and which are not. When a program tries to access an invalid address or one for which it has no permissions, the processor throws out a page fault exception to the kernel. If the fault is fixable, the kernel can fix it and let the program continue running. If the fault is not fixable, then the program cannot continue, since it will never get past the instruction causing the fault.

A fixable fault example——**automatically extended stack:** the kernel initially allocates a single-page user stack, and then if a program faults accessing pages further down the stack, the kernel will allocate those pages automatically and let the program continue. (allocate as need transparently)

- **Memory protection in syscalls**: Most system call interfaces let user programs pass pointers(usually pointed to user buffer) to the kernel. The syscall handler will dereference these pointers, producing two problems:
    1. A page fault in the kernel is potentially a lot more serious than a page fault in a user program. If the kernel page-faults while manipulating its own data structures, that's a kernel bug, and the fault handler should panic the kernel (and hence the whole system). But when the kernel is dereferencing pointers given to it by the user program, it needs a way to remember that any page faults these dereferences cause are actually on behalf of the user program.
    2. The kernel typically has more memory permissions than the user program. The user program might pass a pointer to a system call that points to memory that the kernel can read or write but that the program cannot. The kernel must be careful not to be tricked into dereferencing such a pointer, since that might reveal private information or destroy the integrity of the kernel.
    
    So the kernel must carefully scrutinizes all pointers passed from userspace into the kernel when handling syscalls: When a program passes the kernel a pointer, the kernel should check that the address is in userspace, and that the page table would allow the memory operation. Otherwise, the kernel will never suffer a page fault due to dereferencing a user-supplied pointer——it should panic and terminate when a page fault gradually happens.
    
- **Ex9 & 10:** Change `kern/trap.c` to panic if a page fault happens in kernel mode, and when running `*user/evilhello*`, the environment should be destroyed, without `panic` in kernel.
Hint: Check the low bits of the `tf_cs`to determine whether a fault happened in user mode or kernel mode. Implement `user_mem_check` in `*kern/pmap.c*`. Change `*kern/syscall.c*` to sanity check arguments to sycalls. Change `debuginfo_eip` in `*kern/kdebug.c*` to call `user_mem_check` on `usd`, `stabs`, and `stabstr`. If you now run `*user/breakpoint*`, you should be able to run **backtrace** from the kernel monitor and see the backtrace traverse into `*lib/libmain.c*` before the kernel panics with a page fault. What causes this page fault? You don't need to fix it, but you should understand why it happens.
    
**In `*kern/trap.c*`:** 
    
```c
    void
    page_fault_handler(struct Trapframe *tf)
    {
    	...
    	// Handle kernel-mode page faults.
    	if(!(tf->tf_cs & 0x3))
    		panic("trap_handler: Kernel page fault");
    	...
    }
```
    
**In `*kern/pmap.c*`:** 
    
```c
    int
    user_mem_check(struct Env *env, const void *va, size_t len, int perm)
    {
    	// LAB 3: Your code here.
    	pte_t *pte;
    	uintptr_t vstart, vend;
    
    	vstart = ROUNDDOWN((uintptr_t)va, PGSIZE);
        vend = ROUNDUP((uintptr_t)va + len, PGSIZE);
    
    	if (vend > ULIM || vstart >= ULIM) {
    		user_mem_check_addr = MAX((uintptr_t)ULIM, (uintptr_t)va);
    		return -E_FAULT;
    	}
    
    	for (; vstart < vend; vstart += PGSIZE) {
    		pte = pgdir_walk(env->env_pgdir, (const void *)vstart, 0);
    		if (!pte || ((~*pte) & (perm | PTE_P))) {
    			user_mem_check_addr = MAX((uintptr_t)va, vstart);
    			return -E_FAULT;
    		}
    	}
    	return 0;
    }
```
    
**In `*kern/syscall.c*`: Only `sys_cputs` handler will dereference the pointer passed from user env——add `user_mem_check`.**
    
```c
    static void
    sys_cputs(const char *s, size_t len)
    {
    	// Check that the user has permission to read memory [s, s+len).
    	// Destroy the environment if not.
    	user_mem_assert(curenv, s, len, PTE_U);
    
    	// Print the string supplied by the user.
    	cprintf("%.*s", len, s);
    }
```
    
**In `*kern/kdebug.c*`:** 
    
```c
    int
    debuginfo_eip(uintptr_t addr, struct Eipdebuginfo *info)
    {
    		const struct UserStabData *usd = (const struct UserStabData *) USTABDATA;
    
    		// Make sure this UserStabData memory is valid.
    		// Return -1 if it is not.
    		if (curenv && 
    		(user_mem_check(curenv, (const void *)usd,
    					 sizeof(struct UserStabData), PTE_U) < 0))
    			return -1;
    
    		stabs = usd->stabs;
    		stab_end = usd->stab_end;
    		stabstr = usd->stabstr;
    		stabstr_end = usd->stabstr_end;
    
    		// Make sure the STABS and string table memory is valid.
    		if (curenv && 
    		((user_mem_check(curenv, (const void *)stabs,
    					 (uintptr_t)stab_end - (uintptr_t)stabs, PTE_U) < 0) ||
    		(user_mem_check(curenv, (const void *)stabstr,
    					 (uintptr_t)stabstr_end - (uintptr_t)stabstr, PTE_U) < 0)))
    			return -1;
    }
```
    
Results: 
    
1. **Run `*user/breakpoint*`, then run `backtrace`: (The analysis is in former Exercise 7.)**
    
    ![Untitled](./assets/img/posts/Lab3-Processes/Untitled%209.png)
    
2. **Run `*user/buggyhello(2)*`: They both caused a user_mem_failure as expected.**
    
    ![Untitled](./assets/img/posts/Lab3-Processes/Untitled%2010.png)
    
    **(Try to read one btye from an unmapped memory(0x01))**
    
    ![Untitled](./assets/img/posts/Lab3-Processes/Untitled%2011.png)
    
    **(Try to write to a const value in .data segment(0x803000))**
    
3. **Run `*user/evilhello*`: The attempt to abuse kernel memory is banned.**
    
    ![Untitled](./assets/img/posts/Lab3-Processes/Untitled%2012.png)
    
    **(Try to print the kernel entry point as a string)**
    

## Extra: fix bugs in buddy system

So far, the kernel only succeeds on default *first-fit physical memory manager*: when using *buddy system*, the OS will go into a restart loop. After debugging the procedure and result of mem_init, I figured out that the bootstrap `kern_pgdir` only mapped 8MB kernel space, while the buddy blocks could be accessed at higher address (in `pgdir_walk`).

To solve this issue, the implementation of buddy system(especially page allocation) should be modified, as well as `mem_init`, especially `boot_map_region`.

**In `*pmap.c/pmap.h*`:** 

```c
enum {
	// For page_alloc, zero the returned physical page.
	ALLOC_ZERO = 1<<0,
	// for buddy_system, only allocate blocks below 8MB
	BUDDY_MEM_INIT = 1<<1,
};

static void
boot_map_region(pde_t *pgdir, uintptr_t va, size_t size, physaddr_t pa, int perm)
{
	...
	for (i = 0; i < size; i += PGSIZE){
		if (!(pte = pgdir_walk_for_init(pgdir, (void*)(va + i), 1)))
			panic("failed to allocate a pagetable");
		*pte = (pa + i) | perm | PTE_P;		//permissions in p should be strict
	}
}

// special pgdir_walk only for mem_init
// pass some extra flags to specific pmm
static pte_t *
pgdir_walk_for_init(pde_t *pgdir, const void *va, int create)
{
	// Fill this function in
	pde_t *pde;
	pte_t *pgt;
	struct Page * pi;

	pde = &pgdir[PDX(va)]; 
	if (!(*pde & PTE_P)){	//pde not present
		if (!create || !(pi = alloc_page(ALLOC_ZERO|BUDDY_MEM_INIT)))	//won't create or fail to allocate a pagetable
			return NULL;
		*pde = page2pa(pi) | PTE_P | PTE_W | PTE_U | PTE_PWT | PTE_PCD | PTE_G;
		set_page_ref(pi, 1);
	}
	pgt = KADDR(PTE_ADDR(*pde));	//kva of the page table
	return (pgt + PTX(va));
}
```

**In `*buddy.c*`:**

```c
static inline struct Page *
buddy_alloc_pages_sub(size_t order, int alloc_flags)
{
	...
			list_entry_t *le = list_next(&page_free_list(cur_order));
			struct Page *page = le2page(le, pp_link);
			// when mem_init, buddy system use 8MB mapping by the bootstrap kern_pgdir
			if (alloc_flags & BUDDY_MEM_INIT) {
				if (page->zone_num > 0 
					&& (uintptr_t)page2kva(page) >= KERNBASE +BOOT_KERN_MAP_SIZE)
					continue;
			}
	...
}
```