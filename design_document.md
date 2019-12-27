# Design Document for Project 1: User Programs   
  
## Design Overview  
Task 1: Argument Passing
1.Data structures and functions

In pintos/src/userprog/process.c
I may change this function:
static void start_process (void *file_name_);

2.Algorithms
I may use tokenize to parse the input

3.Synchronization
I think no synchronization is needed. This is inside a thread.

4.Rationale
Maybe copy the command string to the stack will help. This might make me finish the work faster.

Task 2: Process Control Syscalls
1.Data structures and functions

I plan to add the following structures in pintos/src/userprog/process.h:
struct pnode {
  pid_t pid;
  bool loaded;
  struct list_elem elem;
  struct semaphore sema;
  int exit_status; //default is -1
}

and in pintos/src/threads/thread.h
struct thread {
  ...
#ifdef USERPROG
    /* Owned by userprog/process.c. */
    struct pnode* pnode;
    struct list children;
    ...
#endif
  ...
}

2.Algorithms
All syscalls will be implemented in syscall_handler ()(at pintos/src/userprog).
Practice
Take args[1], increment it, and store it in the eax register of the interrupt frame.

Halt
Call shutdown_power_off ()(at pintos/src/devices/shutdown.c).

Exec
Call process_execute ()(at pintos/src/userprog/process.c) on the input string, and obtain the tid of the launching process.

In init_thread ()(at pintos/src/threads/thread.c), we will allocate a pnode struct for the new thread, add it to children of the current thread, and initialize pnode->sema to zero. When the thread finishes loading the program in start_process ()(at pintos/src/userprog/process.c), it will increment pnode->sema.

Meanwhile, back in the exec system call, we will call sema_down ()(at pintos/src/threads/synch.c) on the pnode associated with the child whose pid equals the returned tid. Then, if the pnode has loaded, we will return the pid of the child. Otherwise, we will return -1.

Wait
Look for a pnode in the current thread's children list whose pid equals the given pid. If one does not exist, return -1. Otherwise, call sema_down () on that pnode's semaphore. Then, return its exit_status, remove its list_elem from children, and free the pnode.

A thread will call sema_up ()(at pintos/src/threads/synch.c) on its pnode semaphore at the end of process_exit ()(at pintos/src/userprog/process.c). The default value of exit_status will be -1, but if the exit syscall was used, this value will be overwritten with the provided value.

3.Synchronization
I think no synchronization is needed. This is inside a thread.

4.Rationale
Initially, we wanted to use a pair of locks instead of a semaphore to take advantage of priority donations, but we found it was too difficult to coordinate. The child process would need to acquire them before the parent, but there is no guarantee that the child will run before the parent returns from thread_create.

We decided to create a new struct to hold information about child processes that need to persist even after the child is terminated. Since it's possible for a child to load and terminate before the parent process returns from thread_create (), this includes the semaphore used to signal the parent that the child has finished loading, as well as the boolean indicating success. We also found that pnodes were a better way for parent and child processes to keep track of each other, since a parent can easily maintain a linked list of pnodes, and it also made more sense for a child to possess a pointer to its pnode than directly to its parent (which was our original idea).

Task 3: File Operation Syscalls
1.Data structures and functions
In syscall.c(at pintos/src/lib/user/syscall.c)
struct lock file_lock; 

In syscall.h(at pintos/src/lib/user/syscall.h)
struct files{
	int fd;
	struct file *file_instance;
	const char *file_name;
}

In thread.h(at pintos/src/threads/thread.h)
struct list file_list; 
int cur_fd; 

In syscall.c
struct files* get_file_instance_from_fd(int fd); 
struct files* get_file_instance_from_name(int file_name_); 
int get_cur_fd();

2.Algorithms
Initialize the file_lock inside syscall_init.

Create: Acquire the file_lock to ensure that no process can intrupt the creation. Then use filesys_create to create the file. Release the file_lock after creation.

Remove: Get the FILES object and remove it from the current process's file_list. Then use filesys_remove to remove the file_instance of the FILES object.

Open: Use filesys_open to open the file and its return value is the file_instance. Get the next fd which is get_cur_fd() + 1. Create a FILES instance and put it into the current process's file_list.

Filesize: Get the FILES oject and call file_length on its file_instance.

Read: Get the FILES object and call file_read on its file_instance.

Write: Check the fd. If fd is 1, call lib/kernel/console.c putbuf(buffer, size). Otherwise, get the FILES object with the corresponding fd and call file_write on its file_instance.

Seek: Get the FILES object with the corresponding fd and call file_seek on its file_instance.

Tell: Get the FILES object with the corresponding fd and call file_tell on its file_instance.

Close: Get the FILES object with the corresponding fd and call file_close on its file_instance.

3.Synchronization
In order to prevent intruption during any of the file syscalls, all of the above filesystem syscalls have to acquire the file_lock in the very beginning of the function and release the file_lock right before it returns. Hence, there should not have any intruption in the middle of the file modification. There should not be any synchronization issue.

4.Rationale
There is another approach to store the files in an array, but the array has a fix size and we cannot modify the array size or remove the element after we initialize it, meanwhile we do not know the maximum amount of files a process can hold, we choose to use a linked-list structure instead. Obviousely, we access time of array is faster than the linked-list's, since it cannot be modify and the linked-list structure is provided already, we choose to maintain the files in the linked-list.

1. Additional Questions  
Sc-bad-sp.c: Invokes a system call with the stack pointer (%esp) set to a bad address. The process must be terminated with -1 exit code.

Sc-bad-arg.c: Sticks a system call number (SYS_EXIT) at the very top of the stack, then invokes a system call with the stack pointer (%esp) set to its address. The process must be terminated with -1 exit code because the argument to the system call would be above the top of the user address space.

If a process is holding a lock and die because of the syscall all locks held by the process should be released. Create 2 processes A and B. A acquires lock 1, and then B tries to acquire lock 1, and is made to wait. Process A tries to perform a syscall with a null stack pointer, process A dies. If locks are properly released after a process dies, then process B will be able to finish. Otherwise, the program will be stuck in an infinite loop.
