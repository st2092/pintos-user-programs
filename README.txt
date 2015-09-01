			+--------------------------+
			|       CS 153             |
			| PROJECT 2: USER PROGRAMS |
			|      DESIGN DOCUMENT     |
			+--------------------------+

---- GROUP ----

>> Fill in the names and email addresses of your group members.
Natanael Ramirez
Steven To

---- PRELIMINARIES ----

>> If you have any preliminary comments on your submission, notes for the
>> TAs, or extra credit, please give them here.

The assignment is still incomplete. Argument passing does not work, and 
the system calls have not yet been implemented. Because of this, we can’t 
answer the majority of the questions yet.

>> Please cite any offline or online sources you consulted while
>> preparing your submission, other than the Pintos documentation, course
>> text, lecture notes, and course staff.

			   ARGUMENT PASSING
			   ================

---- DATA STRUCTURES ----

>> A1: Copy here the declaration of each new or changed `struct' or
>> `struct' member, global or static variable, `typedef', or
>> enumeration.  Identify the purpose of each in 25 words or less.

No new variables or changes to struct were declared for argument passing. We did change
the function declaration of static bool setup_stack and bool load to

static bool setup_stack (void **esp, char **saveptr, const char *filename) 

bool load (const char *file_name, void (**eip) (void), void **esp, char **saveptr)

The functions were redefined because in start_process, strtok_r placed a NULL pointer at the
 first space in the command line. This separates the file name from the arguments. The save 
pointer (saveptr) is a pointer to the start of the arguments, which is used in setup_stack.

---- ALGORITHMS ----

>> A2: Briefly describe how you implemented argument parsing.  How do
>> you arrange for the elements of argv[] to be in the right order?
>> How do you avoid overflowing the stack page?

Argument parsing is implemented in the setup_stack function. The algorithm to handle parsing of 
the arguments is as follow:
  1) Create two local variable char **cont and char **argv with a default size of 2.

  2) Parse the command line with strtok_r starting at save pointer. A pointer is kept to each 
  argument in char **cont throughout the parsing process. Since the number of arguments is 
  undefined, we will double the size of both cont and argv if we run out of space.

  3) Once the entire command line is parsed, we will copy cont to argv in reverse order (end to
   start). This is because we have to push onto the stack in reverse order. While copying over to
   argv we push each character string onto the stack as well.

  4) By the end of the copying process, the argv will have all the arguments in reverse order. Then 
  we can word align (multiple of 4 bytes) the stack pointer. 

  5) Push the array of pointers from argv onto the stack. 

  6) Push the argv, argc and fake return address onto the stack.

  7) Free argv and cont

---- RATIONALE ----

>> A3: Why does Pintos implement strtok_r() but not strtok()?

Pintos implement strtok_r() because in strtok() the point where the last
token was found is kept internally by the function to be use on the next
call to strtok(). The problem with this is that this will be prone to
race condition. Suppose two threads are calling strtok(), there is a possible
data race condition where one thread would use the last token held by another
thread. This would be incorrect and has potential to crash the kernel.

>> A4: In Pintos, the kernel separates commands into a executable name
>> and arguments.  In Unix-like systems, the shell does this
>> separation.  Identify at least two advantages of the Unix approach.

The kernel does not have to deal with parsing. Instead the shell will deal
with the parsing and error checking before passing the command line to the
kernel.

			     SYSTEM CALLS
			     ============

---- DATA STRUCTURES ----

>> B1: Copy here the declaration of each new or changed `struct' or
>> `struct' member, global or static variable, `typedef', or
>> enumeration.  Identify the purpose of each in 25 words or less.

Added the following to struct thread:

    struct list file_list; 
    -  The file_list is use to keep track of files.
    
    int fd; 
    -  The fd is the current file descriptor.
      
    struct list child_list;
    -  The child_list is a list of the child processes of the thread.
    
    tid_t parent;
    -  The thread id of the parent of this thread.
      
    struct child_process* cp;
    -  This is pointer the current running child process.
      
     
    struct file* executable; 
    -  Use for denying writes to executables.
    
    struct list lock_list;
    -  Use to keep track of locks the thread holds.
      
Added the following struct in syscall.h:

struct child_process {
  int pid;
  int load_status;
  int wait;
  int exit;
  int status;
  struct semaphore load_sema;
  struct semaphore exit_sema;
  struct list_elem elem;
};

  The struct child_process is meant to be a child process which will hold
  important information such as the pid, load status and semaphores.

struct process_file {
    struct file *file;
    int fd;
    struct list_elem elem;
};
  The process_file holds the file you're currently working with. It contains
  the file descriptor and the content of the file.

struct lock file_system_lock;
  Use for locking critical section that involves modifying files.
  
>> B2: Describe how file descriptors are associated with open files.
>> Are file descriptors unique within the entire OS or just within a
>> single process?

  The file descriptors are unique to each open file for every process. Each
  process has its only file descriptor counter, which will be incremented
  each time a file is opened. Therefore, the file descriptor is unique for each
  separate process.
  
---- ALGORITHMS ----

>> B3: Describe your code for reading and writing user data from the
>> kernel.

    For both write and read, we first check if the stack pointer is valid. 
  If the stack is valid then we can dereference the pointer and find out 
  which system call to run.
    Then we get the three arguments that will be use for the system call 
  “write” or “read” depending on the dereferenced pointer. Next we will 
  check if the buffer is valid. If it is then we can proceed with the 
  system call.
  
>> B4: Suppose a system call causes a full page (4,096 bytes) of data
>> to be copied from user space into the kernel.  What is the least
>> and the greatest possible number of inspections of the page table
>> (e.g. calls to pagedir_get_page()) that might result?  What about
>> for a system call that only copies 2 bytes of data?  Is there room
>> for improvement in these numbers, and how much?

>> B5: Briefly describe your implementation of the "wait" system call
>> and how it interacts with process termination.
 
  The system call "wait" calls process_wait which will wait for child
  processes to terminate using a while loop with sentinel being the child
  process exit member variable.

>> B6: Any access to user program memory at a user-specified address
>> can fail due to a bad pointer value.  Such accesses must cause the
>> process to be terminated.  System calls are fraught with such
>> accesses, e.g. a "write" system call requires reading the system
>> call number from the user stack, then each of the call's three
>> arguments, then an arbitrary amount of user memory, and any of
>> these can fail at any point.  This poses a design and
>> error-handling problem: how do you best avoid obscuring the primary
>> function of code in a morass of error-handling?  Furthermore, when
>> an error is detected, how do you ensure that all temporarily
>> allocated resources (locks, buffers, etc.) are freed?  In a few
>> paragraphs, describe the strategy or strategies you adopted for
>> managing these issues.  Give an example.


---- SYNCHRONIZATION ----

>> B7: The "exec" system call returns -1 if loading the new executable
>> fails, so it cannot return before the new executable has completed
>> loading.  How does your code ensure this?  How is the load
>> success/failure status passed back to the thread that calls "exec"?

I keep track of the status (loaded or not loaded) in a child_process struct.
So after I find the pid, check the status if it is loaded or not. If not
then exit with error (-1).

>> B8: Consider parent process P with child process C.  How do you
>> ensure proper synchronization and avoid race conditions when P
>> calls wait(C) before C exits?  After C exits?  How do you ensure
>> that all resources are freed in each case?  How about when P
>> terminates without waiting, before C exits?  After C exits?  Are
>> there any special cases?
When P waits for C, P will stop and wait for C to exit. When C exits its
lock are released. If P calls wait after C exits, P will have no child to wait
for so it won't wait.
 When P terminates before C exits, P kills all of its children including C.
All of the children will be terminated and release their locks.
When P terminates after C exits, C's locks should be released. 
---- RATIONALE ----

>> B9: Why did you choose to implement access to user memory from the
>> kernel in the way that you did?

  For access to user memory from the kernel, we chose to use function 
  decomposition for error catching. This reduces the difficulty and is
  more simple than to implement page fault memory handling.
  
  Whenever a pointer is invalid, it will be caught by the page fault interrupt
  handler. In the interrupt handler the syscall exit(-1) is called.

>> B10: What advantages or disadvantages can you see to your design
>> for file descriptors?

  Since every file descriptor is unique for each process, it eliminates
  the need to account for race conditions. 

>> B11: The default tid_t to pid_t mapping is the identity mapping.
>> If you changed it, what advantages are there to your approach?
  
  We used the default tid_t to pid_t mapping.

			   SURVEY QUESTIONS
			   ================

Answering these questions is optional, but it will help us improve the
course in future quarters.  Feel free to tell us anything you
want--these questions are just to spur your thoughts.  You may also
choose to respond anonymously in the course evaluations at the end of
the quarter.

>> In your opinion, was this assignment, or any one of the three problems
>> in it, too easy or too hard?  Did it take too long or too little time?

>> Did you find that working on a particular part of the assignment gave
>> you greater insight into some aspect of OS design?

>> Is there some particular fact or hint we should give students in
>> future quarters to help them solve the problems?  Conversely, did you
>> find any of our guidance to be misleading?

>> Do you have any suggestions for the TAs to more effectively assist
>> students, either for future quarters or the remaining projects?

>> Any other comments?


