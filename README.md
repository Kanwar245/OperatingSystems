OperatingSystems
================

CSCC69/369 U of T



<h2>Outline</h2>

<ol>
<li> <a href="c69_p1.shtml#intro">Introduction</a> </li>
<li> <a href="c69_p1.shtml#beg">Beginning the project</a></li>
<li> <a href="c69_p1.shtml#cr">Code Reading</a></li>
<li> <a href="c69_p1.shtml#t1">Task 1</a></li>
<li> <a href="c69_p1.shtml#t2">Task 2</a></li>
<li> <a href="c69_p1.shtml#t3">Task 3</a></li>

</ol>

<h2><a name="intro">Introduction</a></h2>

<p>In this project, you will gain experience with the standard Unix
process API and basic inter-process communication mechanisms.  You
will extend OS/161's thread (process) system, add the ability to pass
command line arguments to user programs on startup, and implement
several system calls to allow OS/161 to support more interesting user
programs. These extensions modify shared OS data structures, so you
will need to add appropriate synchronization to keep the OS's state
consistent.
</p>

<p>To complete this project, you will need to be familiar with the
OS/161 thread code. The thread system provides interrupts, control
functions, and synchronization (semaphores, locks, and condition
variables). You will be extending the thread system to provide
additional, higher-level functions found in most thread libraries and
will use that functionality to implement the process management system
calls <a href="https://mathlab.utsc.utoronto.ca/courses/cscc69s13/OS161_man/syscall/getpid.html">getpid</a>,
   <a href="https://mathlab.utsc.utoronto.ca/courses/cscc69s13/OS161_man/syscall/waitpid.html">waitpid</a>, and <a href="https://mathlab.utsc.utoronto.ca/courses/cscc69s13/assignments/a1/kill.html">kill</a>.</p>

<p>Of course, these system calls are not very interesting without
a way to create new processes! To help with that, we have given you the
implementation of <a href="https://mathlab.utsc.utoronto.ca/courses/cscc69s13/OS161_man/syscall/fork.html">fork</a>.
</p>

<h2><a name="beg">Beginning the project</a></h2>

<!--STG-->
<h3>Project Management</h3>

<p>A big part of these projects is working effectively in a team.
We recommend discussing and documenting how you will manage the
project -- how you will identify jobs which have been completed (or
which remain), specific problems you have encountered, design
decisions you have made, and discoveries/insights you've
encountered. A simple text file in SVN may suffice, but we encourage
you to use DrProject's wiki and ticketing system. We strongly
recommend that you collect your notes in your wiki (or a text file) as
you go. </p>

<p>You are not required to submit your working notes, and they will not be
graded if you do.  In the interview, you will be asked to describe your design, explain design and implementation decisions, and to consider alternate implementations. You will have access to your notes, so document anything that you or your partner might need to know as you discuss the project.  You may be asked to describe work that your partner completed.

</p><h3>Speeding up Testing</h3>

<p>The simulator allows you to pass arguments to the kernel from the
command line.  These arguments are treated as commands entered at the
menu prompt. You may find it convenient to use this feature to
automate some tests. To pass arguments to the kernel from the command
line, pass a string of commands (with a semicolon separating each
command) as the second parameter to sys161. For example, to have the
kernel run the array test and then quit, type:</p>

<pre>    sys161 kernel "at;q"
</pre>

<h2><a name="cr">Code Reading</a></h2>

<p>To learn a bit about where various parts of the thread and
synchronization code lives, complete the following code reading
questions. You don't need to read every line of code in the system to
answer these questions, but note where files and functionality live:
you will be using and modifying code in those locations soon. Treating
these questions seriously will save you time on the rest of the
project, so don't skimp on time -- and feel free to ask for help on
the discussion board.
</p>

<blockquote> 
  <ol>
    <li>What happens to a thread when it exits (i.e., calls
    <tt>thread_exit()</tt> )? What about when it sleeps?</li>

    <li>What function(s) handle(s) a context switch?</li>

    <li>How many thread states are there? What are they?</li>

    <li>What does it mean to turn interrupts off? How is this accomplished?
    Why is it important to turn off interrupts in the thread subsystem
    code?</li>

    <li>What happens when a thread wakes up another thread? How does a
    sleeping thread get to run again?</li>
    
    <li>Semaphores are implemented using <tt>spinlocks</tt>. Where are spinlocks implemented? And what does it mean to get the data for a spinlock?</li>

    <li>Are OS/161 semaphores "strong" or "weak"?</li>

    <li>Why does the lock API in OS/161 provide <tt>lock_do_i_hold()</tt>, but
    not <tt>lock_get_holder()</tt> ?</li>

    <li>What are the ELF magic numbers?</li>

    <li>What is the difference between UIO_USERISPACE and UIO_USERSPACE? When
    should one use UIO_SYSSPACE instead?</li>

    <li>Why can the <tt>struct uio</tt> that is used to read in a segment be
    allocated on the stack in <tt>load_segment()</tt> (i.e., where does the
    memory read actually go)?</li>

    <li>In <tt>runprogram()</tt>, why is it important to call
    <tt>vfs_close()</tt> before going to usermode?</li>

    <li>What function forces the processor to switch into usermode? Is this
    function machine dependent?</li>

    <li>The <tt>mips_trap</tt> function handles all exceptions. Is it possible to determine whether the exception being handled was triggered during user execution or kernel execution?</li>	                
  </ol> 
</blockquote>

<h2><a name="t1">Task 1: Passing arguments to user-level programs on startup.</a></h2>

<p>
  The C programs that you are familiar with accept command line arguments which
  are presented as the <tt>argc</tt> and <tt>argv</tt> arguments to
  the <tt>main()</tt> function.  As you know, <tt>argc</tt> counts the number
  of arguments and <tt>argv</tt> is an array of pointers to the argument strings themselves. By convention, <tt>argv[0]</tt> is the name of the program that
  is running in the process.  Also, <tt>argv[argc]</tt> is
  <tt>NULL</tt> (that is, the <tt>argv</tt> array is NULL-terminated.  If you
  examine the address of <tt>argv</tt> and the strings that it points to, you
  will see that they are stored on the user stack. (On Linux, the stack starts
  at <tt>0xbfffffff</tt> and grows down to lower-numbered addresses.)
</p>

<p>
  Currently, OS/161 user-level processes are deficient because they are not
  provided with any command line arguments.  If you look at the last few lines
  of the <tt>runprogram()</tt> function in <tt>src/kern/syscall/runprogram.c</tt>, you will see that it always passes 0 for <tt>argc</tt> and <tt>NULL</tt>
  for <tt>argv</tt> to the function that transfers control to the newly loaded
  process in user mode.  In fact, the current <tt>runprogram()</tt> function is
  only passed the name of the program to load, and no other arguments. However,
  if you look at the caller of <tt>runprogram()</tt>, <tt>cmd_progthread()</tt>
  in <tt>src/kern/startup/menu.c</tt>, you can see the additional arguments are
  just being discarded.
</p>

<p>
  Your task is to modify the <tt>runprogram()</tt> function, and its caller,
  so that command-line arguments are passed to <tt>runprogram()</tt>,
  copied into the stack region of the new process, and the correct values for
  <tt>argc</tt> and <tt>argv</tt> are passed to the <tt>enter_new_process</tt>.
</p>

<p>
  You will need to use the <tt>copyout</tt> and/or <tt>copyoutstr</tt>
  functions to move data from the operating system to the user
  process's address space.  You must keep track of the virtual address
  where each argument string begins in the user address space, so that
  the <tt>argv</tt> array can be initialized correctly.  These
  pointers in the <tt>argv</tt> array must also be copied out to the
  user stack.  On the mips architecture, pointers must be aligned on a
  4-byte boundary, so you may need to add some padding.  Don't forget
  the NULL termination on the <tt>argv</tt> array, and the strings.
</p>
  
<h2><a name="t2">Task 2: Extending the Thread System</a></h2>

<p>For this part of the project, you will extend the kernel thread
subsystem to allow one thread to wait for another to exit. The starter
code includes a simple thread id management system (see <tt>pid.c</tt>
in <tt>src/kern/thread</tt> and <tt>pid.h</tt> in
<tt>src/kern/include</tt>). We have added code that uses these
functions to allocate an identifier when a new thread is created in
<tt>thread_fork</tt>.
  </p>

<!--TODO: Are these tests relevant?-->

We have also added a new kernel test that can be run from the OS/161
menu as "wt".  Read <tt>kern/test/waittest.c</tt> to see what this
test does.

<h3>Synchronization of the PID System</h3>

<p>The thread id management code requires synchronization. Without it,
if two threads were executing <tt>thread_fork</tt> concurrently, it
would be possible for both new threads to be assigned the same thread
id. This is obviously undesirable, so the pid system includes some
basic locking to ensure mutual exclusion in the pid functions.
</p>

<p>You should think of <tt>pid.c</tt> as implementing a
<em>monitor</em> that controls access to the shared pid data
structures. All access to the pid data structures (including querying
or modifying any fields) must be done through a function provided by
<tt>pid.c</tt>. The functions that start with <tt>pi_</tt> are
internal; they are declared "static" and should only be called from
within <tt>pid.c</tt>. The functions that start with <tt>pid_</tt> are
also declared in <tt>pid.h</tt> and can be called externally; these
are the <em>entry points</em> for the monitor.The existing locking
ensures that at most one thread can be active in the pid monitor at
any time.</p>

<p>Later, as you extend the thread system, you will need to add
additional external functions to <tt>pid.c</tt> to respect the monitor
requirement. Make sure that these functions are also appropriately
synchronized. (<b>Hint:</b> Think about what needs to be a logically
atomic operation, and implement those things as functions in
<tt>pid.c</tt> -- if you find yourself wanting to change an existing
static <tt>pi_</tt> function so that it can be called from outside
<tt>pid.c</tt>, then you are probably thinking about the problem in
the wrong way.)
</p>

<h3>Thread Co-ordination</h3>

<p>
  You must implement or modify the thread and pid functions below. The
  implementations of these four functions must all work together, so
  read and think about all of them before starting to work on any of
  them. Since you are extending the thread subsystem, place any new
  code in <tt>kern/thread/thread.c</tt> or <tt>kern/thread/pid.c</tt>
  and remember to add prototypes to <tt>kern/include/thread.h</tt> or
  <tt>kern/include/pid.h</tt>.
</p>

<ul>
  <li><b><tt>int pid_join(pid_t targetpid, int *status, int flags)</tt></b><br>

     <tt>pid_join</tt> suspends the execution of the calling thread
     until the thread identified by <tt>targetpid</tt> terminates by calling
     <tt>thread_exit</tt>.<br><br>
     If <tt>status</tt> is not NULL, the
     exit status of thread <tt>targetpid</tt> is stored in the location pointed to
     by <tt>status</tt>. The thread <tt>targetpid</tt> must be in the
     joinable state; it must not have been detached using
     <tt>pid_detach</tt>.<br><br>
     When a joinable thread terminates, its <tt>pidinfo</tt> struct
     (containing the PID and exit status, along with other fields) must be retained until another thread
     performs <tt>pid_join</tt> (or <tt>pid_detach</tt>) on
     it.<br><br>
     <b>Any thread in the kernel is allowed to call
     <tt>pid_join</tt> on any other thread, with a few exceptions as noted in the errors below. Multiple threads may call <tt>pid_join</tt> on the same <tt>targetpid</tt>.  All joining threads should be able to retrieve the exit status when the joined thread exits.</b> <br><br>
     <b>Return Value:</b> On success the exit status of thread <tt>targetpid</tt>
     is stored in the location pointed to by <tt>status</tt>, and the pid
     of the joined thread is returned.  On error, a negative error code
     is returned.<br><br>
     <b>Errors:</b>
	  <ul> 
	    <li><b>ESRCH</b>: No thread could be found corresponding
	      to that specified by <tt>targetpid</tt>.</li>
	    
	    <li><b>EINVAL</b>: The thread corresponding to <tt>targetpid</tt> has been detached.</li>
	    
	    <li><b>EINVAL</b>: <tt>targetpid</tt> is INVALID_PID or BOOTUP_PID.</li>
	    
	    <li><b>EDEADLK</b>: The <tt>targetpid</tt> argument refers to the calling thread. (You will need to add this error to <tt>errno.h</tt>, and a corresponding message to <tt>errmsg.h</tt>)</li>
	  </ul><br><br>
	  <b>Hint:</b> To decide whether it is necessary to wait or not,
and to retrieve the exit status of the specified thread, you will need to 
examine values that are internal to the <tt>pid</tt> monitor.
  </li>
	
  <li> <b><tt>int pid_detach(pid_t childpid)</tt></b><br>
            
    <tt>pid_detach</tt> puts the thread <tt>childpid</tt> in the
    detached state. This means that the thread cannot be joined, and the
    thread descriptor and exit
    status can be discarded immediately when <tt>childpid</tt> terminates.
    If <tt>childpid</tt> has already exited when <tt>pid_detach</tt> is
    called, the pidinfo struct should be reclaimed as part of
    handling <tt>pid_detach</tt>.<br><br>
    <b>Return Value:</b> On success, 0 is returned. On error, a
    non-zero error code is returned.<br><br> 
    <b>Errors:</b>
	  <ul>            
	    <li><b>ESRCH</b>: No thread could be found corresponding to that
	      specified by <tt>childpid</tt>.</li>
	  
	    <li><b>EINVAL</b>: The thread <tt>childpid</tt> is already in
              the detached state.</li>
	    
	    <li><b>EINVAL</b>: The caller is not the parent of
              <tt>childpid</tt>.</li>
	    
	    <li><b>EINVAL</b>: <tt>childpid</tt> is INVALID_PID or BOOTUP_PID.</li>
	  </ul>
  </li> 

  <li><b><tt>void thread_exit(int exitcode)</tt> and <tt>void pid_exit(int status, bool dodetach</tt></b><br>
  Modify <tt>thread_exit</tt> so that it calls <tt>pid_exit</tt> to store the
  new exit status argument in its pid struct, and to synchronize correctly
  with the joining thread (if any).  The <tt>dodetach</tt> argument to
  <tt>pid_exit</tt> should be <tt>true</tt> if the exiting thread was running
  a user-level process and <tt>false</tt> otherwise.
  </li>
        
  <li><b><tt>thread_fork</tt></b><br>
  We have changed the prototype and implementation of
  thread_fork so that it returns (via pass by reference) a thread id
  (pid) rather than a pointer to the thread structure itself. If the
  parent does not want to know the thread id (i.e., the last argument
  to thread_fork is NULL), you should place the new thread in the
  detached state immediately. (If the parent does not know the thread
  id, then it will not be able to call pid_join for the new thread).
  </li>
</ul>


<h3>Using <tt>pid_join</tt> and <tt>pid_detach</tt></h3>
      
<p>
  The OS/161 menu thread handles the "p" command by forking a new
  kernel thread to run a user-level program. Because we now have two
  threads running concurrently, and sharing resources (like the
  console device), output printed to the console can appear in any
  order.  Perhaps worse, we have no control over which thread will
  consume the characters when we type input at the console! This makes
  it very difficult to interact with user-level programs.
</p>

<p>
  Use <tt>pid_join</tt> to make the menu thread wait until the child
  thread has finished. In addition, add support for "&amp;"; any command
  that is followed by a "&amp;" should be detached, so that the menu
  thread does not wait for it.
</p>

<!--STG-->
<h3>Interview Planning</h3>

<p>
  Be prepared to describe your solution to the join/detach/exit
  synchronization problem. Describe what will happen if (a) the join
  happens before the exit, (b) the detach happens before exit, (c)
  exit happens before join, and (d) exit happens before detach. Are
  there other possibilities?
</p>

<h2><a name="t3">Task 3: System Calls</a></h2>

<p>In this section, you will implement <a href="https://mathlab.utsc.utoronto.ca/courses/cscc69s13/OS161_man/syscall/getpid.html">getpid</a>,
  <a href="https://mathlab.utsc.utoronto.ca/courses/cscc69s13/OS161_man/syscall/waitpid.html">waitpid</a>, and
  <a href="kill.html">kill</a>.</p>

<p>
  It's crucial that your syscalls handle all error conditions
  gracefully (i.e., without crashing OS/161) and return the correct
  value (in case of success) or error code (in case of failure). Check
  the man pages linked above to get a description of the system calls
  and the errors they need to handle. If there are conditions that can
  happen that are not listed in the man page, return the most
  appropriate error code from <tt>kern/errno.h</tt>. If none seem
  particularly appropriate, consider adding a new one.
</p>

<p>
  If you're adding an error code for a condition for which Unix has a
  standard error code symbol, use the same symbol if possible. If not,
  feel free to make up your own, but note that error codes should
  always begin with E, should not be EOF, etc. Consult Unix man pages
  to learn about Unix error codes; on Linux systems "man errno" will
  do the trick.
</p>

<p>
  Also, be aware that the OS/161 kernel is preemptive, meaning that it
  is possible for two different user-level processes to be executing a
  system call at the same time. This means that any system resources
  need to be protected by appropriate synchronization.
</p>

<p>
  Recall that the file <tt>usr/include/unistd.h</tt> contains the
  user-level interface definition of the system calls that you will be
  writing for OS/161 (including ones you will implement in later
  projects). This interface is different from that of the kernel
  functions that you will define to implement these calls. Note that
  the user-level interface is already defined for the system calls you
  will write in this project, but you might want to read this file
  to see the type and number of arguments that will be passed to the
  OS.
</p>

<h3>getpid(), waitpid()</h3>

<p>
  A pid, or process ID, is a unique number that identifies a process.
  The implementation of <tt>getpid()</tt> should be straightforward,
  since we have already taken care of allocating thread ids when a new
  thread is created (since every user-level process executes in the
  context of a single kernel thread). Similarly, your <tt>waitpid()</tt>
  system call should map nicely to the <tt>pid_join</tt> function in the
  kernel thread system.
</p>
<p>
  Note however that waitpid() has additional restrictions that are not
  enforced when kernel threads want to join with each other.  In particular,
  A process is only allowed to call waitpid() for one of its own children.
</p>

<h3>kill()</h3>

<p>
  The <tt>kill</tt> system call gives a process a way to send a signal
  to another process.  It is often used to terminate another process
  (hence the name), but it is also the system call that is used to
  send <b>any</b> signal. For this project, we allow any process to
  issue a <tt>kill</tt> system call for any other process, rather than
  restricting it to parent/child interactions. (For example, the
  parent of a runaway process may have already exited without waiting
  for it, but a user would still like a mechanism for stopping it!) In
  real systems, a process must have suitable permissions to kill
  another process (i.e., they belong to the same user, or the process
  issuing the kill belongs to a super-user). Since OS/161 does not
  support multiple users, it would not make sense to try to implement
  these restrictions.
</p>

<p>
  <tt>kill</tt> takes two parameters. The first is the pid of the
  process that is the target of the signal, and the second is the
  signal to send. 31 signals are defined, but only a few are
  implemented in OS/161. <b>See the table of signals in the <a href="kill.html">kill man page</a> to see which signals you must
  implement.</b> The signals to be implemented can terminate a process, cause it to stop executing (to be removed from the queue of
  runnable processes), or cause it to be eligible for execution
  again. (<b>Hint:</b> Essentially, you are making a thread
  "wait".) You are also implementing some signals that are handled by
  ignoring them (but error checking is still expected when an "ignored"
  signal number is specified).
</p>

<p>
  The main challenge in implementing this call is that you cannot
  simply stop or destroy the target thread immediately. To
  understand why, consider the possibility that the target process may
  itself be in the middle of a system call when the caller issues the
  <tt>kill()</tt>. This means that the target may be holding system resources,
  such as kernel locks. If we destroy the target's thread structure and
  never run it again, then those resources will never be
  released. Locating all resources held by the target is extremely
  difficult, and even if you could identify them all, just making them
  available again may leave things in an inconsistent state.
</p>

<p>
  The alternative used in real systems (and which you will implement
  for OS/161) is for the <tt>kill()</tt> system call to set a flag in the
  target thread indicating the signal (or signals) that it has been sent
  (i.e., SIGKILL, SIGSTOP, etc.). Every thread checks this flag before
  returning to userspace, and if the flag has been set, it exits
  instead of returning.
</p>

<p>
  <b>Hint:</b>You will find it useful to support the setting and
  checking of this flag through additional functions added to
  <tt>pid.c</tt>. Recall also that <em>all</em> exceptions go through
  <tt>mips_trap</tt> in <tt>kern/arch/mips/locore/trap.c</tt>, making
  this a good place to check things before returning to
  userspace. Recall also that an exception (like a timer interrupt)
  may occur in the middle of a system call causing a recursive call to
  <tt>mips_trap</tt>. Hence, not every return from <tt>mips_trap</tt>
  is a return to userspace.  You need to make sure that you <em>only</em>
  check and deal with signals if the thread is returning to userspace.
</p>

<p>
  We have supplied a new user-level test program named "killtest".
  Read the code in
  <tt>testbin/killtest/killtest.c</tt>
  to understand what
  this test does. You will need to use good DEBUG messages and to have
  those messages on to know that the tests are working correctly.
  You may also want to simplify the test during early development,
  or add additional cases that it does not currently include.
</p>

<h3>Interview Preparation</h3>

<p>Once again, be prepared to discuss how you handle each of these system calls in your code review.  You should have answers to the following questions about the kill system call:

</p><ol>
	<li> What happens if you kill a zombie?</li>
	<li> What exit code is used by a thread that exits because it was
	  killed?</li>
	<li> What happens if a process waits (using the waitpid() system call)
	  for a process that was the target of a kill() system call?</li>
	<li> Can a kernel thread use the same mechanism as the kill() system
	  call to terminate another kernel thread?</li>
	<li> The <tt>killtest</tt> program forks a user-level process that
	  executes an infinite loop (i.e the code contains "<tt>while (1) {}</tt>"), and its parent issues a kill() system call to end it.  Why does this
	  work? That is, why does the process return to userspace if it 
	  contains no code that enters the operating system?</li>
      </ol>

<p>Make sure you can identify where to find your code modifications in the OS/161 source. To make life a bit easier for the TAs, please put the system calls (<tt>sys_getpid, sys_waitpid,</tt> and <tt>sys_kill</tt>)in <tt>src/kern/syscall/proc_syscalls.c</tt>.</p>

<!--STG-->
<h2>Marking Guidelines</h2>

<p>Your submission will be marked based on correctness, style and
efficiency, and your ability to explain your design decisions:</p>

<ol>
  <li><b>Correctness</b> (70%): All of your thread system extensions and system calls should perform the specified work, return the correct return values and error codes, and handle error cases gracefully.

  </li><li><b>Style and Efficiency</b> (15%): Your code should be readable and well
  structured. Functionality should be placed in the correct module and should 
  not be duplicated.  Care should be taken to reduce memory overhead in the 
  design of your data structures, and memory leaks should be avoided.
  
  </li><li><b>Code Interview</b> (15%): You should be able to
  discuss all of the required topics in a clear, professional style.
</li></ol>

<hr>
  
