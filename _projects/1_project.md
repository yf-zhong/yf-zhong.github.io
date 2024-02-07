---
layout: page
title: PintOS
description: Operating System Framework for x86 Architecture
img: assets/img/os-fig.webp
importance: 1
category: Project
related_publications: false
---

## Overview

----

- Designed a lightweight operating system for the x86 instruction set architecture in C with support for argument passing, process control and file operation system calls, and floating-point operations.
- Implemented and integrated a multi-threading system for user programs, featuring an efficient alarm clock and strict priority scheduler with priority donation for kernel threads to prevent deadlock.
- Developed an extensible file system for optimized concurrent disk operations, featuring a write-back buffer cache utilizing NRU replacement policy and a trilevel indexed inode file structure for fast random file access.

## User Program

### Argument Passing

#### Data Structures and Functions

***process.c***

```c
/* split the file_name into argc and argv */
void args_split(char* file_name, char* argv[]) {...}

/* get argc before splitting arguments for memory allocation */
int count_args(const char* file_name) {...}

/* helper function to push arguments onto stack and maintain stack pointer */
void push_stack(void** esp, void* src, size_t size) {...}

/* passed into thread_create() as a thread functinon, push the splited arguments onto stack. */
void args_load(const char* file_name, void** esp) {...} // esp stands for extended stack pointer

```

#### Algorithms

- **`args_load(const char* file_name, void** esp)`** in *process.c*
  - Get the number of arguments from `file_name` with `count_args()`
  - Call `args_split(char* file_name, char* argv[])` to get separated arguments
  - Push each argument onto **user stack** with `push_stack(void** esp, void* src, size_t size)` and make sure stack is 16-bytes aligned
  - Push elements onto aligned stack **in order** based on **calling convention**
    - A `NULL` pointer
    - **Pointers** to arguments saved on heap, i.e. `argv[0 : argc - 1]`
    - Address of `argv`, i.e. `argv[0]`
    - Number of arguments, i.e. `argc`

- **`args_split(char* file_name, char* argv[])`** in *process.c*
  - Split `file_name` by space and save in `argv`

- **`push_stack(void** esp, void* src, size_t size)`** in *process.c*

  - Reseve the memory space by decreasing `esp` by `size` (i.e. stack grows **downwards**, but `memcpy` write **upwards**)
  - Copy the memory of `size` from `src` to `esp`
  - After the push, `esp` will still be on the top of the stack

- **`load(const char* file_name, void (**eip)(void), void** esp)`** in *process.c*
  - Load the ELF executable from `file_name` into the current thread
  - Store the executable's entry into `eip` and its initial stack pointer into `esp`
  - Load program header, setup stack, and set start address
  - call **`args_load()`** to load arguments

#### Synchronization

- No resources are shared

### Process Control Syscalls

#### Data Structures and Functions

*userprog/process.h*

```c
typedef struct child {
  pid_t pid;
  struct semaphore exec_sema; // changed
  struct semaphore wait_sema; // changed
  char* file_name; // deleted
  int exit_status; // added
  bool is_exited; // added
  bool is_waiting; // added
  int ref_cnt;      // added
  struct lock ref_lock; // added
  struct list_elem elem; // change
} CHILD;

struct process {
  /* Owned by process.c. */
  uint32_t* pagedir;          /* Page directory. */
  char process_name[16];      /* Name of the main thread */
  struct thread* main_thread; /* Pointer to main thread */
  struct list children; // changed
  struct child* curr_as_child; // added
  int cur_fd;                 /* The fd number assigned to new file */
  struct list file_descriptor_table; /* All the files opened in current process */
};

// added
typedef struct start_proc_arg {
  struct child* new_c;
  char* file_name;
} SPA;
```

*userprog/process.c*

```c
CHILD* new_child(void);
void t_pcb_init(struct thread*, struct process*, CHILD*);
CHILD* find_child(pid_t);
void decrement_ref_cnt(CHILD*);
void decrement_children_ref_cnt(struct process*);
void exit_setup(struct process*);
void free_spa(SPA*);

static void start_process(void* spaptr_); // handle program loading faiture, call exit_setup on new_pcb

void process_exit(void); // also handle necessary synchronization
```

*userprog/syscall.c*

```c
bool is_valid_addr(uint32_t); // check if the address of args[i] is valid
bool is_valid_str(const char*); // check if the user input string is valid
void sys_practice(struct intr_frame*, int);
void sys_halt(void);
void sys_exec(struct intr_frame*, const char*);
void sys_wait(struct intr_frame*, pid_t);
void sys_exit(struct intr_frame*, int);

/* functions to be involked in if statement to handle actual syscall. */
int practice(int i);
void halt(void);
void exit(int status);
pid_t exec(const char* file);
int wait(pid_t pid);
```

#### Algorithms

- **`struct child`** in *process.h*
  - Processes execute and wait on their own semaphores(`exec_sema`, `wait_sema`) instead of sharing the same one(`sema`). This lower the complexity of handling the synchronization bewteen process.
  - `is_status` indicates whether the process exited
  - `exit_status` saves the exit status of the process
  - `is_waiting` indicates whether the parent process is waiting for the child process
  - `ref_cnt` tracks the number of processes that have access to the current struct
  - `ref_lock` prevents data racing when a process is trying to read/modify `ref_cnt`
- **`struct process`** in *process.h*
  - We added `children` for saving the child processes of the current process.
  - We added `curr_as_child`, so current process can access its corresponding struct child.
- **`struct start_proc_arg`** in *process.h*
  - An argument type for `start_process()` to create a new struct child passing to the new process
- **`process_exit(void)`** in *process.c*
  - Handle **synchoronization** issue
    - `wait`: call `sema_up` on `wait_sema`, such that parent process can continue executing.
    - `exec`: set `is_exited` to true, such that parent process knows the current process exited and is free to read the `exit_status`
  - Parent process doesn’t need to know whether the program is terminated by the kernal since the default value for `exit_status` is `-1`. Parent process can retrieve the exit status from a exited child process as needed.
  - Decrease the `ref_cnt` in the corresponding `struct child` of this process, and the `ref_cnt` in current process’s children.
  - Free the `struct child` when its `ref_cnt` is 0 to avoid memory leak.

- **Syscalls** in *userprog/syscall.c*
  - Determine which syscall the user want by `arg[0]`
    - **practice**: put `arg[1] + 1` to `f → eax`
    - **halt**: call `shut_down_poweroff()`
    - **exec**
      - Add `Children c` to `struct process`
      - Check if `CMD_LINE` is holding a valid address (is less than `PHYS_BASE`), if no, return `-1`
      - Initialize `semaphore` to `0`
      - Call `process_execute(CMD_LINE)`
      - Use `semaphore` to ensure `process_execute` try to load executable before current process continue
        - Parent process calls `sema_down()`
        - Child process calls `sema_up()` after load
      - If child process fails to load executable, return `-1`
      - Make new `Child`, save it to `c`
      - Return what `process_execute` return
    - **wait**
      - Add `Children wc` to `struct process`
      - Initialize `semaphore` to `0`
      - Check if current process is the parent of given `pid` process (in `c`), and returen `-1` if no
      - Check if `wait()` has already called on `pid` (in `wc`), and return `-1` if yes
      - Save `pid` process to `wc`
      - Wait for `pid` process to exit
      - Use `semaphore` to ensure `pid` process finish before current process continue
        - Parent process calls `sema_down()`
        - Child process calls `sema_up()` just before child process finish
      - If know `pid` process is terminated by kernel, return `-1`
      - Return what `pid` process returns

#### Synchronization

- use Child for sharing semaphore between parent and its child processes
- for exec, it needs to have a semaphore to ensure process_execute try to load executable before current process continue
- for wait, it needs a semaphore to ensure the child process exit before continue running current process
