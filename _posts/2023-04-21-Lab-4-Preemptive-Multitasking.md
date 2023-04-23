---
layout: post
read_time: true
show_date: true
categories: JOS
title:  JOS Lab 4 - Preemptive Multitasking
date:   2023-04-21 22:48:20 +0800
description: JOS Lab 4 - Preemptive Multitasking
tags: [coding, jos, operation system, C]
author: Obsidian0215
github: obsidian0215/jos
---

# Lab 4: Preemptive Multitasking

```c

```

## Introduction

- **implementation goals**: **preemptive multitasking among multiple simultaneously active user-mode environments**
    - add multiprocessor support, implement round-robin scheduling, and add environment management system calls (create/destroy environments, allocate/map memory).
    - implement a Unix-like `fork()`, allowing a user-mode environment to create copies of itself.
    - add support for inter-process communication (IPC), allowing different user-mode environments to communicate and synchronize with each other explicitly, additionly add support for hardware clock interrupts and preemption.

New source files in Lab4:

| FileName | Description |
| --- | --- |
| kern/cpu.h | Kernel-private definitions for multiprocessor support |
| kern/mpconfig.c | Code to read the multiprocessor configuration |
| kern/lapic.c | Kernel code driving the local APIC unit in each processor |
| kern/mpentry.S | Assembly-language entry code for non-boot CPUs |
| kern/spinlock.h | Kernel-private definitions for spin locks, including the big kernel lock |
| kern/spinlock.c | Kernel code implementing spin locks |
| kern/sched.c | Code skeleton of the scheduler that you are about to implement |
|  |  |

## Part A: Multiprocessor Support and Cooperative Multitasking

- extend JOS to run on a multiprocessor system
- implement some new JOS kernel system calls to allow user-level environments to create additional new environments
- implement *cooperative* round-robin scheduling, allowing the kernel to switch from one environment to another when the current environment voluntarily relinquishes the CPU

### Multiprocessor Support (SMP)

**SMP**(symmetric multiprocessing): all CPUs have equivalent access to system resources such as memory and I/O buses; while during bootstrap they can be divided into the bootstrap processor (**BSP**—initializing the system and booting the operating system) and the application processors (**APs**—activated by the BSP only after the operating system is up and running). 

In an SMP system, each CPU has an accompanying **Local-APIC (LAPIC)** unit which is responsible for delivering interrupts throughout the system. The LAPIC also provides its connected CPU with a **unique identifier**. some of the following basic functionality of the LAPIC unit (in `*kern/lapic.c*`):

- Reading the LAPIC ID to tell which CPU our code is currently running on: `cpunum()`
- Sending the `STARTUP` interprocessor interrupt (IPI) from the BSP to the APs to bring up other CPUs: `lapic_startap()`
- later program LAPIC's built-in timer to trigger clock interrupts to support preemptive multitasking: `apic_init()`

A processor accesses its LAPIC using **memory-mapped I/O (MMIO)**. In MMIO, a portion of *physical* memory is hardwired to the registers of some I/O devices so that `mov` instruction can be used to access these registers (IO hole example: the VGA display buffer at physical address 0xA0000). The LAPIC lives in a hole starting at physical address 0xFE000000 (32MB short of 4GB). The JOS virtual memory map leaves a 4MB gap at MMIOBASE to map devices. 

- **Ex1:** Implement `mmio_map_region` in kern/pmap.c. To see how this is used, look at the beginning of `lapic_init` in kern/lapic.c. You'll have to do the next exercise, too, before the tests for `mmio_map_region` will run.
    
    **In `*kern/pmap.c*`:** 
    
    ```c
    void *
    mmio_map_region(physaddr_t pa, size_t size)
    {
    	// Where to start the next region.
    	// just like nextfree in boot_alloc.
    	static uintptr_t base = MMIOBASE;
    
    	uintptr_t start;
    	start = base;
    	size = ROUNDUP(size, PGSIZE);
    	if (base + size > MMIOLIM) 
    		panic("invalid mmio mapping over MMIOLIM");
    	base += size;
    	boot_map_region(kern_pgdir, start, size, pa, PTE_W|PTE_PCD|PTE_PWT);
    	return (void *)start;
    }
    ```
    

### **Application Processors Bootstrap**

Before booting up APs, the BSP should first collect information about the multiprocessor system, such as the total number of CPUs, their APIC IDs and the MMIO address of the LAPIC unit. The `mp_init()` function in `*kern/mpconfig.c*` retrieves this information by reading the MP configuration table that resides in the BIOS's region of memory.

The `boot_aps()` function (in `*kern/init.c*`) drives the AP bootstrap process. APs start in real mode, much like how the bootloader started in `*boot/boot.S*`, so `boot_aps()` copies the AP entry code (`*kern/mpentry.S*`) to a memory location that is addressable in the real mode. Unlike with the bootloader, the place where the AP will start executing code can be controlled; the entry at 0x7000 (`MPENTRY_PADDR`), or any unused, page-aligned physical address below 640KB.

After that, `boot_aps()` activates APs one after another, by sending `STARTUP` IPIs to the LAPIC unit of the corresponding AP, along with an initial `CS:IP` address at which the AP should start running its entry code (`MPENTRY_PADDR` in this case). The entry code in `*kern/mpentry.S*` is quite similar to that of `*boot/boot.S`:* setup the AP into protected mode with paging enabled, and then calls `mp_main()` (in `*kern/init.c*`). `boot_aps()` waits for the AP to signal a `CPU_STARTED` flag in `cpu_status` field of its `struct CpuInfo` before going on to wake up the next one.

- **Ex2:** Read `boot_aps()` and `mp_main()` in `*kern/init.c*`, and the assemblys in `*kern/mpentry.S`* to understand the control flow transfer during the bootstrap of APs. Then modify `page_init()` in `*kern/pmap.c*` to avoid adding the page at `MPENTRY_PADDR` to the free list, so that we can safely copy and run AP bootstrap code at that physical address.
    
    **In `*kern/default_pmm.c*`:** 
    
    ```c
    static void
    default_page_init(void)
    {
    #define MARK_FREE(_i) do {\
        set_page_ref(&pages[_i], 0);\
        list_add_before(&page_free_list, &(pages[_i].pp_link));\
    		SetPageProperty(&pages[_i]);\
    		pages[_i].property = 0;\
    		nr_free++;\
    	} while(0)
    
    #define MARK_USE(_i) do {\
        set_page_ref(&pages[_i], 0);\
        SetPageReserved(&pages[_i]);\
        pages[_i].pp_link.next = &(pages[_i].pp_link);\
        pages[_i].pp_link.prev = &(pages[_i].pp_link);\
    	} while(0)
    
    #define MPCT MPENTRY_PADDR / PGSIZE
    	
    	size_t i;
    
      MARK_USE(0);
      for (i = 1; i < MPCT; ++i)
    		MARK_FREE(i);
    	pages[1].property = MPCT - 1;
    
    	// jump the physical page at MPENTRY_PADDR
    	MARK_USE(MPCT);
    	for (i = MPCT + 1; i < npages_basemem; ++i)
    		MARK_FREE(i);
    	pages[MPCT + 1].property = npages_basemem - 1 - MPCT;
    	// jump over the gap between Base(IO) and Extended
      for (i = IOPHYSMEM / PGSIZE; i < EXTPHYSMEM / PGSIZE; ++i)
    		MARK_USE(i);
    	
    	// kernel_base to last boot_alloc end
      for (i = EXTPHYSMEM / PGSIZE; i < boot_alloc_end / PGSIZE; ++i)
    		MARK_USE(i);
      for (i = boot_alloc_end / PGSIZE; i < npages; ++i)
        MARK_FREE(i);
    	pages[boot_alloc_end / PGSIZE].property = npages - boot_alloc_end / PGSIZE;
    
    #undef MPCT
    #undef MARK_USE
    #undef MARK_FREE
    }
    ```
    
    **In `*kern/buddy.c*`:** 
    
    ```c
    //buddy_init_memmap - build page_free_list for Page base follow  n continuing pages.
    static void
    buddy_init_memmap(struct Page *base, size_t n)
    {
    	static int zone_num = 0;
    	assert(n > 0 && zone_num < MAX_ZONE_NUM);
    	struct Page *p = base;
    	for (; p != base + n; p++) {
    		assert(PageReserved(p));
    		p->flags = p->property = 0;
    		p->zone_num = zone_num;
    		set_page_ref(p, 0);
    	}
    	p = zones[zone_num++].mem_base = base;
    	size_t order = MAX_ORDER, order_size = (1 << order);
    
    	while (n != 0) {
    		while (n >= order_size) {
    			p->property = order;
    			SetPageProperty(p);
    			// ClearPageReserved(p);
    			list_add(&page_free_list(order), &(p->pp_link));	//avoid access of unmapped high address while bootstrapping
    			n -= order_size, p += order_size;
    			nr_free(order)++;
    		}
    		order--;
    		order_size >>= 1;
    	}
    }
    
    static void
    buddy_page_init(void)
    {
      #define MARK_USE(_i) SetPageReserved(&pages[_i])
    	#define MPCT MPENTRY_PADDR / PGSIZE
    
    	size_t i;
    
    	//jump over the gap between Base(IO) and Extended
        for (i = 0; i < npages; i ++)
            MARK_USE(i);
        //[1, npages_basemem)
        buddy_init_memmap(&pages[1], MPCT - 1);
    	//[1, npages_basemem)
        buddy_init_memmap(&pages[MPCT+1], npages_basemem - MPCT - 1);
        //[boot_alloc_end / PGSIZE, npages)
        buddy_init_memmap((struct Page *)(pages + boot_alloc_end / PGSIZE), npages - boot_alloc_end / PGSIZE);
    
        #undef MARK_USE
    }
    ```
    

**Question**

- Compare `*kern/mpentry.S*` side by side with `*boot/boot.S*`. `*kern/mpentry.S*` is compiled and linked to run above `KERNBASE` just like everything else in the kernel, what is the purpose of macro `MPBOOTPHYS`? Why is it necessary in `*kern/mpentry.S*` but not in `*boot/boot.S*`? In other words, what could go wrong if it were omitted in kern/mpentry.S?
Hint: recall the differences between the link address and the load address that we have discussed in Lab 1.
    1. **The macro `MPBOOTPHYS` is used to get the physical address of specific symbol.**
    2. **Codes of `*kern/mpentry.S*` (compiled and linked above `KERNBASE`) need to be copied(memmove in boot_aps) to `MPENTRY_PADDR`, requiring re-located aka absolutely-addressing codes. While boot/boot.S is compiled and linked in the form of paddr.**

### **Per-CPU State and Initialization**

It is important to distinguish between *per-CPU state that is private to each processor*, and *global state that the whole system shares*. `*kern/cpu.h*` defines most of the per-CPU state and stores in `struct CpuInfo`. `cpunum()` always returns the ID of the CPU that calls it, which can be used as an index into some CPU-relative arrays like `cpus`. Alternatively, the macro `thiscpu` is shorthand for the current CPU's `struct CpuInfo`.

- **Some important per-CPU states:**
    - **Per-CPU kernel stack:** Because multiple CPUs can trap into the kernel simultaneously, each CPU needs a separate kernel stack to prevent from interfering with each other's execution. The array `percpu_kstacks[NCPU][KSTKSIZE]` reserves space for NCPU's kernel stacks.
        
        In this lab, you will map each CPU's kernel stack into this region with guard pages acting as a buffer between them. CPU 0's stack will still grow down from `KSTACKTOP`; CPU 1's stack will start `KSTKGAP` bytes below the bottom of CPU 0's stack, and `*inc/memlayout.h*` shows the mapping layout.
        
    - **Per-CPU TSS and TSS descriptor:** A per-CPU task-state segment (TSS) is also needed in order to specify where each CPU's kernel stack lives. The TSS for CPU *i* is stored in `cpus[i].cpu_ts`, and the corresponding TSS descriptor is defined in the GDT entry `gdt[(GD_TSS0 >> 3) + i]`. The global `ts` variable defined in kern/trap.c will no longer be useful.
    - **Per-CPU current environment pointer:** Since each CPU can run different user process simultaneously, the symbol `curenv` now refers to `cpus[cpunum()].cpu_env` (or `thiscpu->cpu_env`), which points to the environment *currently* executing on the *current* CPU.
    - **Per-CPU system registers:** All registers, including system registers, are private to a CPU. Therefore, instructions such as `lcr3()`, `ltr()`, `lgdt()`, `lidt()`, must be executed once on each CPU—functions `env_init_percpu()` and `trap_init_percpu()`.

In addition to this, any extra per-CPU state or additional CPU-specific initialization (e.g. new register bits enabled)  to challenge problems need to be  replicated on each CPU here!

- **Ex3:** Modify `mem_init_mp()` (in `*kern/pmap.c*`) to map per-CPU stacks starting at `KSTACKTOP`, as shown in `*inc/memlayout.h*`. Each stack is `KSTKSIZE` bytes plus `KSTKGAP` bytes of unmapped guard pages. After Ex3, new `check_kern_pgdir()` should be passed.
    
    **In `*kern/pmap.c*`:** 
    
    ```c
    static void
    mem_init_mp(void)
    {
    	size_t i;
    	for (i = 0; i < NCPU; i++)
    		boot_map_region(kern_pgdir,
    			KSTACKTOP - KSTKSIZE - i * (KSTKSIZE + KSTKGAP),
                KSTKSIZE,
                PADDR(percpu_kstacks[i]),
                PTE_W);
    }
    ```
    
    **Result: Test functions `check_mm_manager()` and `check_kern_pgdir()` passed**
    
    ![Untitled](./assets/img/posts/Lab4-Preemptive-Multitasking/Untitled.png)
    
- **Ex4:** The code in `trap_init_percpu()` (in `*kern/trap.c*`) initializes the TSS and TSS descriptor for the BSP,which is incorrect when running on APs. Change the code so that it can work on all CPUs. (**Note: New code should not use the global `ts` variable anymore.**)
    
    **In `*kern/trap.c*`:** 
    
    ```c
    void
    trap_init_percpu(void)
    {
    	struct Taskstate *ts;
    	size_t i;
    
    	// Setup a TSS so that we get the right stack
    	// when we trap to the kernel.
    	i = cpunum();
    	ts = &cpus[i].cpu_ts;
    	ts->ts_esp0 = (uintptr_t)percpu_kstacks[i] + KSTKSIZE;
    	ts->ts_ss0 = GD_KD;
    	ts->ts_iomb = sizeof(struct Taskstate);
    
    	// Initialize the TSS slot of the gdt.
    	gdt[(GD_TSS0 >> 3) + i] = SEG16(STS_T32A, (uint32_t)ts,
    					sizeof(struct Taskstate) - 1, 0);
    	gdt[(GD_TSS0 >> 3) + i].sd_s = 0;
    
    	// Load the TSS selector (like other segment selectors, the
    	// bottom three bits are special; we leave them 0)
    	ltr(GD_TSS0 + (i << 3));
    
    	// Load the IDT
    	lidt(&idt_pd);
    }
    ```
    
    **Result: run make qemu CPUS=8 and the output is as follows.**
    
    ![Untitled](./assets/img/posts/Lab4-Preemptive-Multitasking/Untitled%201.png)
    

### Locking

Before letting the AP get any further, it’s quitely ne to first address race conditions when multiple CPUs run kernel code simultaneously. The simplest way to achieve this is to use a *big kernel lock*. The big kernel lock is a single global lock that is held whenever an environment enters kernel mode, and is released when the environment returns to user mode—**Environments in user mode can run concurrently on any available CPUs, but no more than one environment can run in kernel mode; any other environments that try to enter kernel mode are forced to wait.**

- `*kern/spinlock.h*` declares the big kernel lock, `kernel_lock`, with interfaces to acquire and release the lock, `lock_kernel()` and `unlock_kernel()`. The big kernel lock applying location:
    - In `i386_init()`: acquire the lock before the BSP wakes up APs.
    - In `mp_main()`: acquire the lock after initializing the AP, then call `sched_yield()` to start running environments on this AP.
    - In `trap()`: acquire the lock when trapped from user mode. To determine whether a trap happened in user mode or in kernel mode, `trap()` check the low bits of the `tf_cs`.
    - In `env_run()`: release the lock *right before* switching to user mode. Do not do that too early or too late, otherwise you will experience race-conditions or deadlocks.
- **Ex5:** Apply the big kernel lock as described above—call `lock_kernel()`, `unlock_kernel()` at the proper locations.
    
    **In `*kern/init.c*`:** 
    
    ```c
    void
    i386_init(void)
    {
    	...
    	pic_init();
    	lock_kernel();
    	boot_aps();
    	...
    }
    
    void
    mp_main(void)
    {
    	...
    	lapic_init();
    	env_init_percpu();
    	trap_init_percpu();
    	xchg(&thiscpu->cpu_status, CPU_STARTED); // tell boot_aps() we're up
    
    	lock_kernel();
    	sched_yield();
    	for (;;);
    }
    ```
    
    **In `*kern/trap.c*`:** 
    
    ```c
    void
    trap(struct Trapframe *tf)
    {
    	...
    	if ((tf->tf_cs & 3) == 3) {
    		// Trapped from user mode.
    		// Acquire the big kernel lock before doing any
    		// serious kernel work.
    		assert(curenv);
    		lock_kernel();
    
    		// Garbage collect if current enviroment is a zombie
    		if (curenv->env_status == ENV_DYING) {
    			env_free(curenv);
    			curenv = NULL;
    			sched_yield();
    		}
    	...
    }
    ```
    
    **In `*kern/env.c*`:** 
    
    ```c
    void
    env_run(struct Env *e)
    {
    	...
    	lcr3(PADDR(e->env_pgdir));
    	unlock_kernel();
    	env_pop_tf(&e->env_tf);
    }
    ```
    

**Question**

- It seems that using the big kernel lock guarantees that only one CPU can run the kernel code at a time. Why do we still need separate kernel stacks for each CPU? Describe a scenario in which using a shared kernel stack will go wrong, even with the protection of the big kernel lock.
    
    **Since the big kernel lock is always required after entering the kernel, the CPU states will be pushed into the kernel stack. Without separating, another CPU trying to seize the kernel will push its states to the only kernel stack(by hardware), which can make a big mess when `iret`.**  
    

***Challenge: fine-grained locking***

- The big kernel lock is simple and easy to use. Nevertheless, it eliminates all concurrency in kernel mode. Most modern operating systems use different locks to protect different parts of their shared state, an approach called *fine-grained locking*. *Fine-grained locking* can increase (concurrency) performance significantly, but is more difficult to implement and error-prone. If you are brave enough, drop the big kernel lock and embrace concurrency in JOS!
    
    Following shared components in the JOS kernel can use spinlocks to ensure exclusive access: 
    **environment** / **page allocator** / **console driver** / **scheduler** / **inter-process communication (IPC) state**.
    
    **Definition in `*kern/spinlock.h*`:** 
    
    ```c
    struct spinlock env_lock = {
    #ifdef DEBUG_SPINLOCK
    	.name = "env_lock"
    #endif
    };
    
    struct spinlock page_lock = {
    #ifdef DEBUG_SPINLOCK
    	.name = "page_lock"
    #endif
    };
    
    struct spinlock console_lock = {
    #ifdef DEBUG_SPINLOCK
    	.name = "console_lock"
    #endif
    };
    
    struct spinlock ipc_lock = {
    #ifdef DEBUG_SPINLOCK
    	.name = "ipc_lock"
    #endif
    };
    
    struct spinlock scheduler_lock = {
    #ifdef DEBUG_SPINLOCK
    	.name = "scheduler_lock"
    #endif
    };
    ```
    
    1. **Page Allocator lock:** 
        
        **临界区：`Struct page`—`*pages*`, `(struct) free_area_t`—`*free_area[11]*`(buddy), `*free_area*`(first_fit)**
        
        **调用关系图：**
        
    2. **Environment lock:** 
        
        **临界区：`Struct Env`—`*envs*`: env_id, env_status, env_pgfault_upcall, env_pgdir, env_ipc_recving, env_ipc_dstva, env_ipc_value, env_ipc_from, env_ipc_perm, env_priority; `*env_free_list*`**
        
        **调用关系图：**
        
    3. **Console driver lock:**
        
        **临界区：**
        
        **调用关系图：**
        
    4. **IPC lock:** 
        
        
    5. **Scheduler lock:** 
        
        目前还未实现通用的调度器类，
        

**Some interesting issues**

- **indirect call? how to call mp_main for APs**
    
    **In `*kern/mpentry.S*`, the JOS kernel uses the indirect call to mp_main to startup the APs.** 
    
    ```c
    movl    $mp_main, %eax
    call    *%eax
    ```
    
    **However, if I try to use direct call (`call mp_main`), the kernel will fail and reset.**
    
    **While using the following call methods will not get this issue.**
    
    ```c
    push	$spin   ;return addr(while should't go to)
    push	$mp_main
    ret
    ```
    
    ```c
    lcall $PROT_MODE_CSEG, $mp_main  ;PROT_MODE_CSEG=0x08
    ```
    
    **A possible reason is that direct call get the jump destination by calculating the offset, while in this case it’s much too far (`mp_main`=0xe01001e0, `direct call`=0xe010789c). Other calls directly use the v-addr of `mp_main`(0xe01001e0) to jump into, so that they avoid the issue.**
    
- **`Fast_syscall` caused kernel crash**
    
    **As dericribed above, when the processor trap into kernel, it should acquire a big *kernel spinlock*. On the contrary, the processor should release the kernel spinlock right before returning to user process. In lab3, the `fast_syscall` using `sysenter/sysexit` can invoke the corresponding syscall handler. However, as this call path jumped over the trapentry and trap(dispatch) and didn’t save user states, some syscall will make  some problems(e.g., fast path didn’t acquire the kernel lock, when `sys_yield` runs an *env*, the unlock operation in `env_run` will cause a zero vaddr *page fault*)**
    
    **For now, the simplest way is using the *`int`* path for some lock-relevant functions.**
    

### **Round-Robin Scheduling**

Your next task in this lab is to change the JOS kernel so that it can alternate between multiple environments in "round-robin" fashion. Round-robin scheduling in JOS works as follows:

- The function `sched_yield()` in `*kern/sched.c*` is responsible for selecting a new environment to run. It searches sequentially through the `envs[]` array in circular fashion, starting just after the previously running environment (or at the beginning of the array if there was no previously running environment), picks the first environment it finds with a status of `ENV_RUNNABLE` (see inc/env.h), and calls `env_run()` to jump into that environment.
- `sched_yield()` must never run the same environment on two CPUs at the same time. It can tell that an environment is currently running on some CPU (possibly the current CPU) because that environment's status will be `ENV_RUNNING`.
- We have implemented a new system call for you, `sys_yield()`, which user environments can call to invoke the kernel's `sched_yield()` function and thereby voluntarily give up the CPU to a different environment.
- **Ex6:** Implement round-robin scheduling in `sched_yield()`(also in `mp_main`) as described above. Don't forget to modify `syscall()` to dispatch `sys_yield()`. Modify `*kern/init.c*` to create three more environments that all run the program `*user/yield.c*`. After the yield programs exit (no runnable environment), the scheduler should invoke the JOS kmonitor.
    
    **In `*kern/sched.c*`:**
    
    ```c
    void
    sched_yield(void)
    {
    	struct Env *idle;
    	idle = NULL;
    
    	size_t i = 0;
    	if (curenv) {
    		i = ENVX(curenv->env_id) + 1;
    		while (1) {
    			idle = &envs[i];
    			if (idle == curenv) {
    				// context switch to running current env(still runnable)
    				if (curenv->env_status == ENV_RUNNING)
    					env_run(idle);
    				// no runnable env found
    				break;
    			}
    			// context switch to a runnable env found
    			if (idle->env_status == ENV_RUNNABLE)
    				env_run(idle);
    			i = (i + 1) & (NENV - 1);
    		}
    	} else {
    		for (i = 0; i < NENV; ++i) {
                if (envs[i].env_status == ENV_RUNNABLE) {
                    idle = &envs[i];
                    env_run(idle);
                }
    		}
    	}
    }
    ```
    
    **In `*kern/syscall.c*`:**
    
    ```c
    int32_t
    syscall(uint32_t syscallno, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
    {
    	switch (syscallno) {
    		...
    		case SYS_yield: 
    			sys_yield();
    			return 0;
    		...
    	}
    }
    ```
    
    **In `*kern/init.c*`:**
    
    ```c
    void
    i386_init(void)
    {
    	...
    	pic_init();
    	lock_kernel();
    	boot_aps();
    	...
    }
    
    void
    mp_main(void)
    {
    	...
    	lapic_init();
    	env_init_percpu();
    	trap_init_percpu();
    	xchg(&thiscpu->cpu_status, CPU_STARTED); // tell boot_aps() we're up
    
    	lock_kernel();
    	sched_yield();
    	// for (;;);
    }
    ```
    
    **Result: Run `user_yield`, and `sys_yield` works well.**
    
    ![Untitled](./assets/img/posts/Lab4-Preemptive-Multitasking/Untitled%202.png)
    

**Question**

- `env_run()` should have called `lcr3()`. Before and after the call to `lcr3()`, it makes references (at least it should) to the argument `e`. Upon loading the `%cr3` register, the addressing context used by the MMU is instantly changed. But a virtual address (namely `e`) has meaning relative to a given address context--the physical address to which the virtual address maps. Why can the pointer `e` be dereferenced both before and after the addressing switch?
    
    **Because the pointer e points to a kernel data array(envs), with idential kernel mappings within all the user envs.**
    
- Whenever the kernel switches from one environment to another, it must ensure the old environment's registers are saved so they can be restored properly later. Why? Where does this happen?
    
    **The environment’s contexts can only be saved as a member of PCB (`struct Env`), as there is not any seperated kernel stack for each process. The point of saving contexts is at `trap` (in `*kern/trap.c*`):**
    
    ```c
    void
    trap(struct Trapframe *tf)
    {
    	...
    
    	if ((tf->tf_cs & 3) == 3) {
    		...
    		// Copy trap frame (which is currently on the stack)
    		// into 'curenv->env_tf', so that running the environment
    		// will restart at the trap point.
    		curenv->env_tf = *tf;
    		// The trapframe on the stack should be ignored from here on.
    		tf = &curenv->env_tf;
    	}
    	...
    }
    ```
    

***Challenge:*** 

- Add a less trivial scheduling policy to the kernel, such as a ***fixed-priority scheduler*** that allows each environment to be assigned a priority and ensures that higher-priority environments are always chosen in preference to lower-priority environments. As an adventure, try implementing a Unix-style ***adjustable-priority scheduler*** or even a ***lottery or stride scheduler.*** Write a test program or two that verifies that your scheduling algorithm is working correctly (i.e., the right environments get run in the right order).
    
    
- The JOS kernel currently does not allow applications to use the x86 processor's **x87 floating-point unit (FPU)**, **MMX instructions**, or **Streaming SIMD Extensions (SSE)**. Extend the `Env` structure to provide a save area for the processor's floating point state, and extend the context switching code to save and restore this state properly when switching from one environment to another. The `FXSAVE` and `FXRSTOR` instructions may be useful, but note that they are available in more recent processors. Write a user-level test program that does something cool with floating-point.
    
    

### **System Calls for Environment Creation**

So far JOS kernel is still limited to running environments that the *kernel* initially set up: it’s necessary to implement some JOS syscalls to allow *user* environments to create and start other new user environments.

Unix’s *process creation primitive* `fork()` syscall copies the entire address space of calling process (the parent) to create a new process (the child). The only differences from user space are their PIDs and parent PIDs ( `getpid` / `getppid`). In the parent, `fork()` returns the child's process ID, while in the child, `fork()` returns 0. By default, each process gets its own private address space, and neither process's modifications to memory are visible to the other.

There is a different, more primitive set of JOS system calls for creating new user environments, which can be a Unix-like `fork()`. **For all of the following syscalls that accept env IDs, the JOS kernel supports the convention that 0 value means "the current environment.", implemented by `envid2env()` in `*kern/env.c*`.**

- **The new syscalls for user envs creation in JOS:**
    - `sys_exofork`: This system call creates a new environment with an almost blank slate: nothing is mapped in the user portion of its address space, and it’s not runnable. The new environment will have the same register state as the parent environment at the time of the `sys_exofork` call. In the parent, `sys_exofork` will return the `envid_t` of the newly created environment (or a negative error code if the environment allocation failed). In the child, however, it will return 0. (Since the child starts out marked as not runnable, `sys_exofork` will not actually return in the child until the parent has explicitly allowed this by marking the child runnable)
    - `sys_env_set_status`: Sets the status of a specified environment to `ENV_RUNNABLE` or `ENV_NOT_RUNNABLE`. This system call is typically used to mark a new environment ready to run, once its address space and register state has been fully initialized.
    - `sys_page_alloc`: Allocates a page of physical memory and maps it at a given virtual address in a given environment's address space.
    - `sys_page_map`: Copy a page mapping (*not* the contents of a page!) from one environment's address space to another, leaving a memory sharing arrangement in place so that the new and the old mappings both refer to the same page of physical memory.
    - `sys_page_unmap`: Unmap a page mapped at a given virtual address in a given environment.
- **Ex7:** Implement the system calls described above in `*kern/syscall.c*` and make sure `syscall()` calls them. Use various functions in `*kern/pmap.c*` and `*kern/env.c*`, particularly `envid2env()`. For now, whenever you call `envid2env()`, pass 1 in the parameter `checkperm`. Be sure to check for any invalid syscall arguments, returning `-E_INVAL`. Test with `*user/dumbfork*` and make sure it works before proceeding.
    
    **In `*kern/syscall.c*`:**
    
    ```c
    // Dispatches to the correct kernel function, passing the arguments.
    int32_t
    syscall(uint32_t syscallno, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
    {
    	// Call the function corresponding to the 'syscallno' parameter.
    	// Return any appropriate return value.
    	// LAB 3: Your code here.
    	switch (syscallno) {
    		...
    		case SYS_page_alloc:
    			return sys_page_alloc(a1, (void *)a2, a3);
    		case SYS_page_map:
    			return sys_page_map(a1, (void *)a2, a3, (void *)a4, a5);
    		case SYS_page_unmap:
    			return sys_page_unmap(a1, (void *)a2);
    		case SYS_exofork:
    			return sys_exofork();
    		case SYS_env_set_status:
    			return sys_env_set_status(a1, a2);
    		...
    	}
    }
    
    static envid_t
    sys_exofork(void)
    {
    struct Env *e;
    	int r;
    
    	if ((r = env_alloc(&e, curenv->env_id, curenv->env_priority)) < 0)
    		return r;
    
    	e->env_status = ENV_NOT_RUNNABLE;
    	e->env_tf = curenv->env_tf;
    	// child process will return 0
    	e->env_tf.tf_regs.reg_eax = 0;
    
    	return e->env_id;
    }
    
    static int
    sys_env_set_status(envid_t envid, int status)
    {
    	if (status < 0 || status >= ENV_INVALID_STATUS)
    		return -E_INVAL;
    	struct Env *e;
    	int r = envid2env(envid, &e, 1);
    	if (r < 0)
    		return -E_BAD_ENV;
    	e->env_status = status;
    	return 0;
    }
    
    #define CHECK_PGALIGN_UVA(_va) do {	\
    	if ((uintptr_t)_va >= UTOP	\
    		|| ((uintptr_t)_va & (PGSIZE - 1)))	\
    		return -E_INVAL;	\
    } while (0)
    
    #define CHECK_SYSCALL_PERM(_perm) do {	\
    	if (!(_perm & PTE_P)	\
    		|| !(_perm & PTE_U)	\
    		|| (_perm & ~PTE_SYSCALL))	\
    		return -E_INVAL;	\
    } while (0)
    
    static int
    sys_page_alloc(envid_t envid, void *va, int perm)
    {
    	struct Env *e;
    	struct Page *pp;
    	pte_t *pte;
    
    	CHECK_PGALIGN_UVA(va);
    	CHECK_SYSCALL_PERM(perm);
    	if (envid2env(envid, &e, 1) < 0)
    		return -E_BAD_ENV;
    	
    	if (!(pp = alloc_page(ALLOC_ZERO)))
    		return -E_NO_MEM;
    	
    	int err;
    	if ((err = page_insert(e->env_pgdir, pp, va, perm)) < 0) {
    		free_page(pp);
    		return err;
    	}
    	return 0;
    }
    
    static int
    sys_page_map(envid_t srcenvid, void *srcva, envid_t dstenvid, void *dstva, int perm)
    {
    	struct Env *srce,*dste;
    	pte_t *pte;
    	Struct Page *pp;
    	int err;
    
    	CHECK_PGALIGN_UVA(srcva);
    	CHECK_PGALIGN_UVA(dstva);
    	CHECK_SYSCALL_PERM(perm);
    	if ((envid2env(srcenvid, &srce, 1)) < 0)
    		return -E_BAD_ENV;
    	if ((envid2env(dstenvid, &dste, 1)) < 0)
    		return -E_BAD_ENV;
    	if (!(pp = page_lookup(srce->env_pgdir, srcva, &pte)))
    		return -E_INVAL;
    	if ((~(*pte) & PTE_W) && (perm & PTE_W))
    		return -E_INVAL;
    	if ((err = page_insert(dste->env_pgdir, pp, dstva, perm)) < 0)
    		return err;
    	return 0;
    }
    
    static int
    sys_page_unmap(envid_t envid, void *va)
    {
    	int err;
    	struct Env *e;
    
    	CHECK_PGALIGN_UVA(va);
    	if ((err = envid2env(envid, &e, 1)) < 0)
    		return err;
    	page_remove(ppdste->env_pgdir, va);
    	
    	return 0;
    }
    #undef CHECK_PGALIGN_UVA
    #undef CHECK_SYSCALL_PERM
    ```
    
    **Result: run `*user/dumbfork*`**
    
    ```
    SMP: CPU 0 found 2 CPU(s)
    enabled interrupts: 1 2
    SMP: CPU 1 starting
    [00000000] new env 00001000
    [00001000] new env 00001001
    0: I am the parent!
    0: I am the child!
    1: I am the parent!
    1: I am the child!
    2: I am the parent!
    2: I am the child!
    3: I am the parent!
    3: I am the child!
    4: I am the parent!
    4: I am the child!
    5: I am the parent!
    5: I am the child!
    6: I am the parent!
    6: I am the child!
    7: I am the parent!
    7: I am the child!
    8: I am the parent!
    8: I am the child!
    9: I am the parent!
    9: I am the child!
    [00001000] exiting gracefully
    [00001000] free env 00001000
    10: I am the child!
    11: I am the child!
    12: I am the child!
    13: I am the child!
    14: I am the child!
    15: I am the child!
    16: I am the child!
    17: I am the child!
    18: I am the child!
    19: I am the child!
    [00001001] exiting gracefully
    [00001001] free env 00001001
    No runnable environments in the system!
    ```
    

***Challenge:*** 

- Add the additional system calls necessary to ***read* all of the vital state of an existing environment as well as set it up**. Then implement a user mode program that forks off a child environment, runs it for a while (e.g., a few iterations of `sys_yield()`), then takes a complete **snapshot** or ***checkpoint*** of the child environment, runs the child for a while longer, and finally **restores** the child environment to the state it was in at the checkpoint and continues it from there. Thus, you are effectively "replaying" the execution of the child environment from an intermediate state. Make the child environment perform some interaction with the user using `sys_cgetc()` or `readline()` so that the user can view and mutate its internal state, and verify that with your checkpoint/**restart** you can give the child environment a case of selective amnesia, making it **"forget"** everything that happened beyond a certain point.
    
    

## **Part B: Copy-on-Write Fork**

As mentioned earlier, Unix provides the `fork()` system call as its primary process creation primitive. The `fork()` system call copies the address space of the calling process (the parent) to create a new process (the child).

Primitive `fork()`(like `dumbfork()`) copys all data from the parent's pages into new pages allocated for the child, which is the most expensive part of the `fork()` operation. However, a `fork()` call is frequently followed almost immediately by `exec()` in the child process, which replaces the child's memory with a new program. In this case, the time spent copying the parent's address space is largely wasted.

Newer `fork()` took advantage of virtual memory hardware to allow the parent and child to *share* the memory mapped into their respective address spaces until one of the processes actually modifies it——**COW*(copy-on-write)***. To do this, on `fork()` the kernel would only copy **the address space *mappings*** from the parent to the child, and at the same time mark the **now-shared pages read-only**. When one of the two processes tries to write to one of these shared pages, the process takes a **page fault**. At this point, the kernel realizes that the page was a "copy-on-write" copy, and so it makes a new, *private*, *writable copy* of the page for the faulting process. In this way, the contents of individual pages aren't actually copied until they are actually written to. This optimization makes a `fork()` followed by an `exec()` in the child much cheaper: the child will probably only need to copy **one page (the current page of its stack)** before it calls `exec()`.

Implementing `fork()` and *copy-on-write support* in ***user space library*** has the benefit that the kernel remains much simpler and thus more likely to be correct. It also lets individual user-mode programs define their *own semantics* for `fork()`. A program that wants a slightly different implementation (e.g. the expensive always-copy version like `dumbfork()`, or one in which the parent and child actually share memory afterward) can easily provide its own.

### User-level page fault handling

A user-level copy-on-write `fork()` needs to know about page faults on *write-protected pages* firstly. *Copy-on-write* is only one of many possible uses for user-level page fault handling.

It's common to set up an address space so that page faults indicate when some action needs to take place. For example, most Unix kernels initially map only a single page in a new process's stack region, and allocate and map additional stack pages later "on demand" as the process's stack consumption increases and causes page faults on stack addresses that are not yet mapped. A typical Unix kernel must keep track of what action to take when a page fault occurs in each region of a process's space. For example, **a fault in the stack region** will typically allocate and map new page of physical memory. **A fault in the program's BSS region** will typically allocate a new page, fill it with zeroes, and map it. In systems with *demand-paged executables*, **a fault in the text region** will read the corresponding page of the binary off of disk and then map it.

This is a lot of information for the kernel to keep track of. Instead of taking the traditional Unix approach, you will decide what to do about each page fault in user space, where bugs are less damaging. This design has the added benefit of allowing programs great **flexibility** in defining their memory regions; you'll use user-level page fault handling later for mapping and accessing files on a disk-based file system.

### Setting the Page Fault Handler

In order to handle its own page faults, a user environment will need to register a *page fault handler entrypoint* with the JOS kernel. The user environment registers its page fault entrypoint via the new `sys_env_set_pgfault_upcall` system call. We have added a new member to the `Env` structure, `env_pgfault_upcall`, to record this information.

- **Ex8:** Implement the `sys_env_set_pgfault_upcall` system call. Be sure to enable permission checking when looking up the env ID of the target environment, since this is a "dangerous" system call.
    
    **In `*kern/syscall.c*`:**
    
    ```c
    static int
    sys_env_set_pgfault_upcall(envid_t envid, void *func)
    {
    	struct Env *e;
    	if (envid2env(envid, &e, 1) < 0)
    		return -E_BAD_ENV;
    	e->env_pgfault_upcall = func;
    	return 0;
    }
    ```
    

### **Normal and Exception Stacks in User Environments**

During normal execution, a user environment in JOS will run on the *normal* user stack: its ESP register starts out pointing at `USTACKTOP`, and the stack data it pushes resides on the page between `USTACKTOP-PGSIZE` and `USTACKTOP-1` inclusive. **When a page fault occurs in user mode, however, the kernel will restart the user environment running a designated user-level page fault handler on a different stack, namely the *user exception* stack**. In essence, we will make the JOS kernel implement automatic "stack switching" on behalf of the user environment, in much the same way that the x86 *processor* already implements stack switching on behalf of JOS when transferring from user mode to kernel mode!

The JOS user exception stack is also one page-size, and its top is defined to be at virtual address `UXSTACKTOP`, so the valid bytes of the user exception stack are from `UXSTACKTOP-PGSIZE` through `UXSTACKTOP-1` inclusive. While running on this exception stack, the user-level page fault handler can use JOS's regular system calls to map new pages or adjust mappings so as to fix whatever problem originally caused the page fault. Then the user-level page fault handler returns, via an assembly language stub, to the faulting code on the original stack.

Each user environment that wants to support user-level page fault handling will need to allocate memory for its own exception stack, using the `sys_page_alloc()` system call.

### Invoking the User Page Fault Handler

You will now need to change the page fault handling code in kern/trap.c to handle page faults from user mode as follows. We will call the state of the user environment at the time of the fault the *trap-time* state.

If there is no page fault handler registered, the JOS kernel destroys the user environment with a message as before. Otherwise, the kernel sets up a `struct UTrapframe`(from `*inc/trap.h*`) on the exception stack:

```c
										<-- UXSTACKTOP
trap-time esp       esp+0x30
trap-time eflags    esp+0x2c
trap-time eip       esp+0x28
trap-time eax       esp+0x24 -- start of struct PushRegs
trap-time ecx       esp+0x20
trap-time edx       esp+0x1c
trap-time ebx       esp+0x18
trap-time esp       esp+0x14
trap-time ebp       esp+0x10
trap-time esi       esp+0xc
trap-time edi       esp+0x8 -- end of struct PushRegs
tf_err (error code) esp+0x4
fault_va            <-- %esp when handler is run
```

The kernel then arranges for the user environment to resume execution with the page fault handler running on the exception stack with this stack frame; you must figure out how to make this happen. The `fault_va` is the virtual address that caused the page fault.

If the user environment is *already* running on the user exception stack when an exception occurs, then the page fault handler itself has faulted. In this case, you should start the new stack frame just under the current `tf->tf_esp` rather than at `UXSTACKTOP`. You should first push an empty 32-bit word, then a `struct UTrapframe`. To test whether `tf->tf_esp` is already on the user exception stack, check whether it is in the range between `UXSTACKTOP-PGSIZE` and `UXSTACKTOP-1`, inclusive.

- **Ex9:** Implement the code in `page_fault_handler` in `*kern/trap.c*` required to dispatch page faults to the user-mode handler. Be sure to take appropriate precautions when writing into the exception stack. (if the user environment runs out of exception stack)
    
    **In `*kern/trap.c*`:**
    
    ```c
    void
    page_fault_handler(struct Trapframe *tf)
    {
    	uint32_t fault_va;
    
    	// Read processor's CR2 register to find the faulting address
    	fault_va = rcr2();
    
    	// Handle kernel-mode page faults.
    	// LAB 3: Your code here.
    	if(!(tf->tf_cs & 0x3)) {
            print_trapframe(tf);
            panic("[%08x] kernel fault va %08x ip %08x\n",
    		    curenv->env_id, fault_va, tf->tf_eip);
        }
    
    	// NULL function potiner check
    	if (!curenv->env_pgfault_upcall) {
    		log("null upcall pointer");
    		goto bad;
    	}
    
    	// check whether upcall and xstack are in user spaces
    	user_mem_assert(curenv, (void *)curenv->env_pgfault_upcall, 0, PTE_U);
      user_mem_assert(curenv, (void *)(UXSTACKTOP - 1), 0, PTE_U | PTE_W);
    
    	// check uxstack overflow (pgfault within stack gap)
    	if (fault_va < UXSTACKTOP - PGSIZE
    		&& fault_va >= UXSTACKTOP - PGSIZE * 2) {
    		log("uxstack overflow");
    		goto bad;
    	}
    	
    	// create a user trapframe
    	uintptr_t xesp;
    	struct UTrapframe *utf;
    
    	// if there is a recursive pgfault
    	// when trap_time esp is [UXSTACKTOP-PGSIZE,UXSTACKTOP)
    	if (tf->tf_esp >= (UXSTACKTOP - PGSIZE) 
    				&& tf->tf_esp < UXSTACKTOP)
    		xesp = tf->tf_esp - (sizeof(struct UTrapframe) + 4);
    	else
    		xesp = UXSTACKTOP - sizeof(struct UTrapframe);
    	
    	// check uxstack overflow (pgfault within stack gap)
    	if (xesp < UXSTACKTOP - PGSIZE) {
    		log("uxstack overflow");
    		goto bad;
    	}
    
    	utf = (struct UTrapframe *)xesp;
    	utf->utf_fault_va = fault_va;
    	utf->utf_err = tf->tf_err;
    	utf->utf_eip = tf->tf_eip;
    	utf->utf_esp = tf->tf_esp;
    	utf->utf_regs = tf->tf_regs;
    	utf->utf_eflags = tf->tf_eflags;
    
    	// modify tf_esp and tf_eip to go into upcall entry
    	tf->tf_esp = xesp;
    	tf->tf_eip = (uintptr_t)curenv->env_pgfault_upcall;
    	return ;
    
    bad: 
    	// Destroy the environment that caused the fault.
    	cprintf("[%08x] user fault va %08x ip %08x\n",
    		curenv->env_id, fault_va, tf->tf_eip);
    	print_trapframe(tf);
    	env_destroy(curenv);
    }
    ```
    

### User-mode Page Fault Entrypoint

Next, you need to implement *the assembly routine* that will take care of *calling the C page fault handler* and *resume execution at the original faulting instruction*. This assembly routine is the handler that will be registered with the kernel using `sys_env_set_pgfault_upcall()`.

- **Ex10:** Implement the `_pgfault_upcall` routine in `*lib/pfentry.S`--directly* returning to the original *user code point* that caused the page fault, without going back through the kernel; simultaneously *switching stacks* and *re-loading the EIP*.
    
    **In `*lib/pfentry.S*`:**
    
    ```
    _pgfault_upcall:
    	// Call the C page fault handler.
    	pushl %esp			// function argument: pointer to UTF
    	movl _pgfault_handler, %eax
    	call *%eax
    	addl $4, %esp			// pop function argument
    	
    	movl 0x30(%esp), %eax	     # trap-time esp
    	movl 0x28(%esp), %ebx	     # trap-time eip
    	sub $0x4, %eax	           # update trap-time esp to save trap-time eip(return addr)
    	movl %ebx, (%eax)	         # save return addr in trap-time esp
    	movl %eax, 0x30(%esp)	     # update trap-time esp to swtich
    	add $0x8, %esp	           # ignore fault_va and error code
    
    	// Restore the trap-time registers.
    	popal
    	add $0x4, %esp	           # ignore eip
    
    	// Restore eflags from the stack.
    	popfl
    
    	// Switch back to the adjusted trap-time stack.
    	popl %esp	#switch to user stack, dest.esp->trap-time eip
    
    	// Return to re-execute the instruction that faulted.
    	ret
    ```
    

Finally, you need to implement the C user library side of the user-level page fault handling mechanism.

- **Ex11:** Finish `set_pgfault_handler()` in lib/pgfault.c.
    
    **In `*lib/pgfault.c*`:**
    
    ```c
    void
    set_pgfault_handler(void (*handler)(struct UTrapframe *utf))
    {
    	int r;
    
    	if (_pgfault_handler == 0) {
    		// allocate user exception stack
    		if ((r = sys_page_alloc()) < 0)
    			panic("set_pgfault_handler: %e", r);
    		
    		// register curenv's page fault entrypoint
    		if ((r = sys_env_set_pgfault_upcall(0, _pgfault_upcall)) < 0)
    			panic("set_pgfault_handler: %e", r);
    	}
    
    	// Save handler pointer for assembly to call.
    	_pgfault_handler = handler;
    }
    ```
    

### Testing Result

Run `*user/faultread*` (**make run-faultread**): 

```
...
[00000000] new env 00001000
[00001000] user fault va 00000000 ip 0080003a
TRAP frame ...
[00001000] free env 00001000

```

Run `*user/faultdie*`:

```
...
[00000000] new env 00001000
i faulted at va deadbeef, err 6
[00001000] exiting gracefully
[00001000] free env 00001000

```

Run `*user/faultalloc*`: If only the first "this string" line seen, recursive pgfault handling is not implemented properly.

```
...
[00000000] new env 00001000
fault deadbeef
this string was faulted in at deadbeef
fault cafebffe
fault cafec000
this string was faulted in at cafebffe
[00001000] exiting gracefully
[00001000] free env 00001000

```

Run `*user/faultallocbad*`:

```
...
[00000000] new env 00001000
[00001000] user_mem_check assertion failure for va deadbeef
[00001000] free env 00001000

```

Why `*user/faultalloc*` and `*user/faultallocbad*` behave differently: **In `*user/faultalloc*`, the pgfault is handled by *the registered user handler* as expected. However in `*user/faultallocbad*`, the pgfault is detected firstly by kernel function `sys_cputs`, which will do `user_mem_check` while the *va* is not present.**

***Challenge:***

- Extend JOS kernel so that ***all* types of processor exceptions that user-mode code can generate**, can be redirected to a *user-mode exception handler*. Test user-mode handling of exceptions such as *divide-by-zero*, *general protection fault*, and *illegal opcode*.
    
    

### Implementing Copy-on-Write Fork

Now, JOS has the kernel facilities to implement copy-on-write `fork()` entirely in user space.

There is already a skeleton for `fork()` in *`lib/fork.c`*. Like `dumbfork()`, `fork()` should create a new environment, then scan through the parent environment's address space and set up corresponding page mappings in the child. *The key difference* is that,  `fork()` will initially only copy *page mappings,* while `dumbfork()` copied *pages*. `fork()` will copy each page only when one of the environments tries to write it.

**The basic control flow for `fork()`:**

1. The parent installs `pgfault()` as the C-level page fault handler, using the `set_pgfault_handler()` function.
2. The parent calls `sys_exofork()` to create a child environment.
3. For each writable or copy-on-write page in its address space below UTOP, the parent calls `duppage`, which should *firstly* map the page copy-on-write into the address space of the *child* and then *remap* the page copy-on-write in its own address space. `duppage` sets both PTEs so that the page is not writeable, and to contain `PTE_COW` in the "avail" field to distinguish copy-on-write pages from genuine read-only pages. The exception stack is *not* remapped this way, however. Instead you need to allocate a fresh page in the child for the exception stack. Since the page fault handler will be doing the actual copying and the page fault handler runs on the exception stack, the exception stack cannot be made copy-on-write. `fork()` also needs to handle pages that are present, but not writable or copy-on-write.
4. The parent sets the user page fault entrypoint for the child to look like its own.
5. The child is now ready to run, so the parent marks it runnable.

Each time one of the environments writes a copy-on-write page that it hasn't yet written, it will take a page fault. 

**The control flow for the user page fault handler:**

1. The kernel propagates the page fault to `_pgfault_upcall`, which calls `fork()`'s `pgfault()` handler.
2. `pgfault()` checks that the fault is a write (check for `FEC_WR` in `err`) and that the PTE for the page is marked `PTE_COW`. If not, `panic`.
3. `pgfault()` allocates a new page mapped at a temporary location and copies the contents of the faulting page into it. Then the fault handler maps the new page at the appropriate address with *read/write* permissions, in place of the old read-only mapping.

The user-level *`lib/fork.c`* code must consult the environment's page tables for several of the operations above (e.g. the PTE for a page is marked `PTE_COW`). The kernel maps the environment's page tables at `UVPT` exactly for this purpose. `UVPT` makes it easy to lookup PTEs for user code. `*lib/entry.S*` sets up `uvpt` and `uvpd` so that you can easily lookup page-table information in `*lib/fork.c*`.

- **Ex12:** Implement `fork`, `duppage` and `pgfault` in `lib/fork.c`.
    
    **In `*lib/fork.c*`:**
    
    ```c
    static void
    pgfault(struct UTrapframe *utf)
    {
    	void *addr = (void *) utf->utf_fault_va;
    	uint32_t err = utf->utf_err;
    	int r;
    
    	addr = ROUNDDOWN(addr, PGSIZE);
    	if (!(	(uvpd[PDX(addr)] & PTE_P) && 
                (uvpt[PGNUM(addr)] & PTE_P) && 
                (uvpt[PGNUM(addr)] & PTE_U) && 
                (uvpt[PGNUM(addr)] & PTE_COW) && 
                (err & FEC_WR)))
            panic(
                "[0x%08x] user page fault va 0x%08x ip 0x%08x: "
                "[%s, %s, %s]",
                sys_getenvid(),
                utf->utf_fault_va,
                utf->utf_eip,
                err & 4 ? "user" : "kernel",
                err & 2 ? "write" : "read",
                err & 1 ? "protection" : "not-present"
            );
    
        if (r = sys_page_alloc(0, PFTEMP, PTE_P | PTE_U | PTE_W), r < 0)
            panic("page fault handler: %e", r);
        memmove(PFTEMP, addr, PGSIZE);
        if (r = sys_page_map(0, PFTEMP, 0, addr, PTE_P | PTE_U | PTE_W), r < 0)
            panic("page fault handler: %e", r);
        if (r = sys_page_unmap(0, PFTEMP), r < 0)
            panic("page fault handler: %e", r);
        return;
    }
    
    static int
    duppage(envid_t envid, unsigned pn)
    {
    	int r;
    	void *addr = (void *)(pn * PGSIZE);
    	pte_t pte = uvpt[pn];
    
    	assert((pte & PTE_P) && (pte & PTE_U));
    	if ((pte & PTE_W) || (pte & PTE_COW)) {
    		if ((r = sys_page_map(0, addr, envid, addr, PTE_P | PTE_U | PTE_COW)) < 0)
    			return r;
    		if ((r = sys_page_map(0, addr, 0, addr, PTE_P | PTE_U | PTE_COW)) < 0)
    			return r;
    	} else {
    		if ((r = sys_page_map(0, addr, envid, addr, PTE_P | PTE_U)) < 0)
    			return r;
    	}
    	return 0;
    }
    
    envid_t
    fork(void)
    {
    	int r;
    	envid_t child_eid;
    
    	set_pgfault_handler(pgfault);
    
    	if (!(child_eid = sys_exofork()))
    		thisenv = &envs[ENVX(sys_getenvid())];
    	else if (child_eid > 0) {
    		// don't copy the UXSTACK(aka pn<=UTOP/PGSIZE-1)
    		for (unsigned pn = 0; pn < UTOP / PGSIZE - 1;) {
    			pde_t pde = uvpd[pn / NPDENTRIES];
    			if (!(pde & PTE_P))
    				pn += NPDENTRIES;
    			else {
    				unsigned next = MIN(UTOP / PGSIZE - 1, pn + NPDENTRIES);
                    for (; pn < next; ++pn) {
                        pte_t pte = uvpt[pn];
                        if ((pte & PTE_P) && (pte & PTE_U))
                            if (r = duppage(child_eid, pn), r < 0)
                                panic("fork: %e", r);
    				}
    			}
    		}
            if (r = sys_page_alloc(child_eid, (void*)(UXSTACKTOP - PGSIZE), PTE_P | PTE_U | PTE_W), r < 0)
                panic("%s:%s fork: %e", __FILE__, __LINE__, r);
            if (r = sys_env_set_pgfault_upcall(child_eid, _pgfault_upcall), r < 0)
                panic("%s:%s fork: %e", __FILE__, __LINE__, r);
            if (r = sys_env_set_status(child_eid, ENV_RUNNABLE), r < 0)
                panic("%s:%s fork: %e", __FILE__, __LINE__, r);
    	}
    	return child_eid;
    }
    ```
    
    **Result: `*user/forktree`--*produce the following messages, with interspersed 'new env', 'free env', and 'exiting gracefully' messages.**
    
    ```
    [00000000] new env 00001000
    1000: I am ''
    [00001000] new env 00001001
    [00001000] new env 00001002
    [00001000] exiting gracefully
    [00001000] free env 00001000
    1001: I am '0'
    [00001001] new env 00002000
    [00001001] new env 00001003
    [00001001] exiting gracefully
    [00001001] free env 00001001
    2000: I am '00'
    [00002000] new env 00002001
    [00002000] new env 00001004
    [00002000] exiting gracefully
    [00002000] free env 00002000
    2001: I am '000'
    [00002001] exiting gracefully
    [00002001] free env 00002001
    1002: I am '1'
    [00001002] new env 00003001
    [00001002] new env 00003000
    [00001002] exiting gracefully
    [00001002] free env 00001002
    3000: I am '11'
    [00003000] new env 00002002
    [00003000] new env 00001005
    [00003000] exiting gracefully
    [00003000] free env 00003000
    3001: I am '10'
    [00003001] new env 00004000
    [00003001] new env 00001006
    [00003001] exiting gracefully
    [00003001] free env 00003001
    4000: I am '100'
    [00004000] exiting gracefully
    [00004000] free env 00004000
    2002: I am '110'
    [00002002] exiting gracefully
    [00002002] free env 00002002
    1003: I am '01'
    [00001003] new env 00003002
    [00001003] new env 00005000
    [00001003] exiting gracefully
    [00001003] free env 00001003
    5000: I am '011'
    [00005000] exiting gracefully
    [00005000] free env 00005000
    3002: I am '010'
    [00003002] exiting gracefully
    [00003002] free env 00003002
    1004: I am '001'
    [00001004] exiting gracefully
    [00001004] free env 00001004
    1005: I am '111'
    [00001005] exiting gracefully
    [00001005] free env 00001005
    1006: I am '101'
    [00001006] exiting gracefully
    [00001006] free env 00001006
    No runnable environments in the system!
    ```
    

***Challenge:***

- Implement a shared-memory `fork()` called `sfork()`. This version should have the parent and child *share* all their memory pages (so writes in one environment appear in the other) except for pages in the stack area, which should be treated in the usual copy-on-write manner. Modify `*user/forktree.c*` to use `sfork()` instead of regular `fork()`. Also, once you have finished implementing IPC in part C, use your `sfork()` to run `*user/pingpongs*`. You will have to find a new way to provide the functionality of the global `thisenv` pointer.
    
    
- Current implementation of `fork` makes a huge number of system calls. Switching into the kernel using interrupts has non-trivial cost. Augment the system call interface so that it is possible to send a batch of system calls at once. Then change `fork` to use this interface. Try to measure how faster the new `fork` can make——
1) (roughly) using analytical arguments to estimate how much of an improvement batching system calls will make to the performance of `fork`: How expensive is an `int 0x30`? How many executed `int 0x30` in `fork`? Accessing the TSS stack switch will be expensive? ...
2) Alternatively, you can *really* benchmark your code on real hardwate. See the `RDTSC` (read time-stamp counter) instruction, which counts the number of clock cycles that have elapsed since the last processor reset.
    
**In `*kern/syscall.c*`:**
    
```c
    static struct Env *
    fork_exofork(void)
    {
    	struct Env *e;
    
    	if(env_alloc(&e, curenv->env_id, curenv->env_priority + 1) < 0)
    		return NULL;
    
    	e->env_tf = curenv->env_tf;
    	e->env_tf.tf_regs.reg_eax = 0;
    	e->env_pgfault_upcall = curenv->env_pgfault_upcall;
    	
    	return e;
    }
    
    static int
    fork_page_map(struct Env *dstenv, void *srcva, void *dstva, int perm)
    {
    	struct Page *pp;
    	pte_t *pte;
    	int r;
    	
    	CHECK_PGALIGN_UVA(srcva);
    	CHECK_PGALIGN_UVA(dstva);
    	CHECK_SYSCALL_PERM(perm);
    
    	if(!(pp = page_lookup(curenv->env_pgdir, srcva, &pte)))
    		return -E_INVAL;
    	
    	if((r = page_insert(dstenv->env_pgdir, pp, dstva, perm)) < 0)
    		return r;
    	return 0;
    }
    
    static int
    fork_duppage(struct Env *childenv, void *va)
    {
    	pte_t pte = *pgdir_walk(curenv->env_pgdir, va, 0);
    	int r;
    
    	if (!(pte & PTE_P) || !(pte & PTE_U))
    		return 0;
    	if ((pte & PTE_W) || (pte & PTE_COW)) {
    		if ((r = fork_page_map(childenv, va, va, PTE_P | PTE_U | PTE_COW))<0)
    			return r;
    		if ((r = fork_page_map(curenv, va, va, PTE_P | PTE_U | PTE_COW))<0)
    			return r;
    	} else {
    		if ((r = fork_page_map(childenv, va, va, PTE_P | PTE_U)) < 0)
    			return r;
    	}
    	return 0;
    }
    
    static int
    fork_page_alloc(struct Env *e, void *va, int perm)
    {
    	struct Page* pp;
    	int r;
    
    	CHECK_PGALIGN_UVA(va);
    	CHECK_SYSCALL_PERM(perm);
    
    	if(!(pp = alloc_page(ALLOC_ZERO)))
    		return -E_NO_MEM;
    	
    	if((r = page_insert(e->env_pgdir, pp, va, perm)) < 0){
    		free_page(pp);
    		return r;
    	}
    	return 0;
    }
    
    static envid_t
    sys_fork(unsigned char end[])
    {
    	uintptr_t va;
    	int r;
    	struct Env *childenv;
    
    	if (!(childenv = fork_exofork())) {
    		return -E_NO_MEM;
    	}
    	
    	fork_duppage(childenv,
    		 (void *)ROUNDDOWN(curenv->env_tf.tf_esp, PGSIZE));
    	
    	if((r = fork_page_alloc(curenv,
    			 (void *)PFTEMP, PTE_W | PTE_U | PTE_P)) < 0){
    		return r;
    	}
    
    	memmove((void *)PFTEMP, (void *)(UXSTACKTOP - PGSIZE), PGSIZE);
    
    	if((r = fork_page_map(childenv, (void *)PFTEMP, 
    			(void *)(UXSTACKTOP - PGSIZE), PTE_P | PTE_U | PTE_W)) < 0){
    		return r;
    	}
    
    	page_remove(curenv->env_pgdir, (void*)PFTEMP);
    
    	for(va = 0; va < (uintptr_t)end; va += PGSIZE)
    		fork_duppage(childenv, (void *)va);
    	
    	childenv->env_status = ENV_RUNNABLE;
    
    	return childenv->env_id;
    }
```
    
**In `*lib/fork.c*`:**
    
```c
    // BATCH_SYSCALL_FORK defined in inc/.config
    envid_t
    fork(void)
    {
    	int r;
    	envid_t child_eid;
    
    	set_pgfault_handler(pgfault);
    
    #ifdef BATCH_SYSCALL_FORK
    
    	extern unsigned char end[];
    	set_pgfault_handler(pgfault);
    	if (!(child_eid = sys_fork(end)))
    		thisenv = &envs[ENVX(sys_getenvid())];
    
    #else
    
    	if (!(child_eid = sys_exofork()))
    		thisenv = &envs[ENVX(sys_getenvid())];
    	else if (child_eid > 0) {
    		// assume UTOP == UXSTACKTOP
    		// don't copy the UXSTACK(aka pn<=UTOP/PGSIZE-1)
    		for (unsigned pn = 0; pn < UTOP / PGSIZE - 1;) {
    			pde_t pde = uvpd[pn / NPDENTRIES];
    			if (!(pde & PTE_P))
    				pn += NPDENTRIES;
    			else {
    				unsigned next = MIN(UTOP / PGSIZE - 1, pn + NPDENTRIES);
                    for (; pn < next; ++pn) {
                        pte_t pte = uvpt[pn];
                        if ((pte & PTE_P) && (pte & PTE_U))
                            if (r = duppage(child_eid, pn), r < 0)
                                panic("fork: %e", r);
    				}
    			}
    		}
    
            if (r = sys_page_alloc(child_eid, (void*)(UXSTACKTOP - PGSIZE), PTE_P | PTE_U | PTE_W), r < 0)
                panic("fork: %e", r);
            if (r = sys_env_set_pgfault_upcall(child_eid, _pgfault_upcall), r < 0)
                panic("fork: %e", r);
            if (r = sys_env_set_status(child_eid, ENV_RUNNABLE), r < 0)
                panic("fork: %e", r);
    	}
    
    #endif
    
    	return child_eid;
    }
```
    
    **Result: same with Ex12 (`*user/forktree*`)**
    

## **Part C: Preemptive Multitasking and Inter-Process communication (IPC)**

preempt uncooperative environments /  allow environments to pass messages to each other explicitly.

### Clock Interrupts and Preemption

Run the *`user/spin`* test program. This test program forks off a child environment, which simply spins forever in a tight loop once it receives control of the CPU. Neither the parent environment nor the kernel ever regains the CPU. This is obviously not good for protecting the system from bugs or malicious code in user-mode, since any user-mode environment can bring the whole system to a *halt* simply by getting into an infinite loop and never giving back the CPU. In order to allow the kernel to *preempt* a running environment, forcefully retaking control of the CPU from it, we must extend the JOS kernel to support external hardware interrupts from the clock hardware.

### Interrupt discipline

External interrupts (i.e., device interrupts) are referred to as IRQs. There are 16 possible IRQs, numbered 0 through 15. The mapping from IRQ number to IDT entry is not fixed. `pic_init` in picirq.c maps IRQs 0-15 to IDT entries `IRQ_OFFSET` through `IRQ_OFFSET+15`.

In inc/trap.h, `IRQ_OFFSET` is defined to be decimal 32. Thus the IDT entries 32-47 correspond to the IRQs 0-15. For example, the clock interrupt is IRQ 0. Thus, IDT[IRQ_OFFSET+0] (i.e., IDT[32]) contains the address of the clock's interrupt handler routine in the kernel. This `IRQ_OFFSET` is chosen so that the device interrupts do not overlap with the processor exceptions, which could obviously cause confusion.

In JOS, we make a key simplification compared to xv6 Unix. **External device interrupts are *always* disabled when in the kernel (and, like xv6, enabled when in user space).** External interrupts are controlled by the `FL_IF` flag bit of the `%eflags` register. When this bit is set, external interrupts are enabled. While the bit can be modified in several ways, because of our simplification, we will handle it solely through the process of saving and restoring `%eflags` register as we enter and leave user mode.

You will have to ensure that the `FL_IF` flag is set in user environments when they run so that when an interrupt arrives, it gets passed through to the processor and handled by your interrupt code. Otherwise, interrupts are *masked*, or ignored until interrupts are re-enabled. We masked interrupts with the very first instruction of the bootloader, and so far we have never gotten around to re-enabling them.

- **Ex13:** Modify `*kern/trapentry.S*` and `*kern/trap.c*` to initialize the appropriate entries in the IDT and provide handlers for IRQs 0 through 15. Then modify the code in `env_alloc()` in kern/env.c to ensure that user environments are always run with interrupts enabled. Also uncomment the sti instruction in sched_halt() so that idle CPUs unmask interrupts.
    
    **In `*kern/trapentry.S*`: in `sysenter_handler`, the `sti` instruction is needed right before `sysexit`.**
    
    ```c
    #define TH(n) TRAPHANDLER_NOEC(handler##n, n)
    #define THE(n) TRAPHANDLER(handler##n, n)
    
    #include <kern/trapvector.inc>
    /*
    // IRQ_OFFSET+n
    TH(32) // TIMER
    TH(33) // KBD
    TH(34)
    TH(35)
    TH(36) // SERIAL
    TH(37)
    TH(38)
    TH(39) // SPURIOUS
    TH(40)
    TH(41)
    TH(42)
    TH(43)
    TH(44)
    TH(45)
    TH(46) // IDE
    TH(47)
    TH(48) // SYSCALL
    // Other interrupt
    TH(51) // ERROR
    */
    
    .globl sysenter_handler
     sysenter_handler:
    	...
    	# before returning to userspace
    	# enable the interrupt
    	sti
    	sysexit
    ```
    
    **In `*kern/trap.c*`:**
    
    ```c
    void
    trap_init(void)
    {
    	...
    	for (int i = IRQ_OFFSET; i < IRQ_OFFSET + 16; ++i)
            SETGATE(idt[i], 0, GD_KT, handlers[i], 0);
    	...
    }
    ```
    
    **In `*kern/env.c*`:**
    
    ```c
    int
    env_alloc(struct Env **newenv_store, envid_t parent_id, uint8_t priority)
    {
    	...
    	e->env_tf.tf_eflags |= FL_IF;
    	...
    }
    ```
    
    **Result: with `sti` in `sched_halt()`, running `*user/spin*` will output the hardware interrupt’s trapframe.**
    
    ![Untitled](./assets/img/posts/Lab4-Preemptive-Multitasking/Untitled%203.png)
    

### Handling Clock Interrupts

In the `*user/spin*` program, after the child environment was first run, it just spun in a loop, and the kernel never got control back. We need to program the hardware to generate clock interrupts periodically, which will force control back to the kernel where we can switch control to a different user environment. The calls to `lapic_init` and `pic_init` (from `i386_init` in `*init.c*`) set up the clock and the interrupt controller to generate interrupts. You now need to write the code to handle these interrupts.

- **Ex14:** Modify the kernel's `trap_dispatch()` function so that it calls `sched_yield()` to find and run a different environment whenever a clock interrupt takes place.
    
    **In `*kern/trap.c*`:**
    
    ```c
    static void
    trap_dispatch(struct Trapframe *tf)
    {
    	...
    	if (tf->tf_trapno == IRQ_OFFSET + IRQ_TIMER) {
    		lapic_eoi();
    		sched_yield();
    		return;
    	}
    	...
    }
    ```
    
    **Result: Now in `*user/spin`,* the parent environment should `fork` off the child, `sys_yield()` to it a couple times but in each case regain control of the CPU after one time slice, and finally kill the child environment and terminate gracefully.**
    
    ![Untitled](./assets/img/posts/Lab4-Preemptive-Multitasking/Untitled%204.png)
    
    **Now, `make CPUS=2 grade` will succeed in the part of `stresssched`.**
    

### Inter-Process communication (IPC)

So far the OS focus on the isolation, which provides *the illusion that each program has a machine all to itself*. Another important service of an operating system is to *allow programs to communicate with each other when they want to*. It can be quite powerful to let programs interact with other programs. There are many models for interprocess communication (e.g., Unix `pipe` model). Even today there are still debates about which models are best. 

### IPC in JOS

You will implement a few additional JOS kernel system calls that collectively provide a simple interprocess communication mechanism—— including two system calls, `sys_ipc_recv` and `sys_ipc_try_send`, and two library wrappers `ipc_recv` and `ipc_send`.

The "messages" that user environments can send to each other using JOS's IPC mechanism consist of two components: *a single 32-bit value*, and *optionally* *a single page mapping*. Allowing environments to pass page mappings in messages provides an efficient way to transfer more data than will fit into a single 32-bit integer, and also allows environments to set up shared memory arrangements easily.

### Sending and Receiving Messages

To receive a message, an environment calls `sys_ipc_recv`. This system call de-schedules the current environment and does not run it again until a message has been received. When an environment is waiting to receive a message, *any* other environment can send it a message - not just a particular environment or environments that have a parent/child arrangement with the receiving environment. In other words, the previous permission checking will not apply to IPC, since the IPC system calls are carefully designed so as to be *"safe"*: an environment cannot cause another environment to malfunction simply by sending it messages (unless the target environment is also buggy).

To try to send a value, an environment calls `sys_ipc_try_send` with both the receiver's environment id and the value to be sent. If the named environment is actually receiving (it has called `sys_ipc_recv` and not gotten a value yet), then the send delivers the message and returns 0. Otherwise the send returns `-E_IPC_NOT_RECV` to indicate that the target environment is not currently expecting to receive a value.

A library function `ipc_recv` in user space will take care of calling `sys_ipc_recv` and then looking up the information about the received values in the current environment's `struct Env`.

Similarly, a library function `ipc_send` will take care of repeatedly calling `sys_ipc_try_send` until the send succeeds.