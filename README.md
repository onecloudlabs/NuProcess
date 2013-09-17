[![Build Status](https://www.travis-ci.org/brettwooldridge/NuProcess.png?branch=master)](https://www.travis-ci.org/brettwooldridge/NuProcess)

NuProcess
=========

A low-overhead, non-blocking I/O, external Process execution implementation for Java.  Think of it as ``java.lang.ProcessBuilder``
and ``java.lang.Process`` on steroids.

Have you ever been annoyed by the fact that whenever you spawn a process in Java you have to create two or three "pumper"
threads (for every process) to pull data out of the ``stdout`` and ``stderr`` pipes and pump data into ``stdin``?  If your
code starts a lot of processes you can have dozens or hundreds of threads doing nothing but pumping data!

NuProcess used the JNA library to use platform-specific native APIs to achive non-blocking I/O on the pipes between your
Java process and the spawned processes:

 * Linux: uses epoll
 * MacOS X: uses kqueue/kevent
 * Windows: uses IO Completion Ports

#### It's mostly about the memory ####
Speed-wise, there is not a significant difference between NuProcess and the standard Java Process class, even when running
500 concurrent processes.  On some platforms such as MacOS X or Linux, NuProcess is 20% faster than ``java.lang.Process``
for large numbers of processes, while on Windows NuProcess is about 20% slower.

However, when it comes to memory there is a significant difference.  The overhead of 500 threads, for example, is quite
large compared to the one or few threads employed by NuProcess.

Additionally, on unix-based platforms such as Linux, when creating a new process ``java.lang.Process`` uses a fork()/exec()
operation.  This requires a temporary copy of the Java process (the fork), before the exec is performed.  When running
tests on Linux, in order to spawn 500 processes required setting the JVM max. memory to 3Gb (-Xmx3g).  NuProcess uses a
variant of fork() called vfork(), which does not impose this overhead.  NuProcess can comfortably spawn 500 processes
even when running the JVM with only 120Mb (-Xmx128m).

#### Settings ####

##### ``org.nuprocess.threads`` #####
This setting controls how many threads are used to handle the STDIN, STDOUT, STDERR streams of spawned processes.  No
matter how many processes are spawned, this setting will be the *maximum* number of threads used.  Possible values are:

 * ``auto`` (default) - this sets the *maximum* number of threads to the number of CPU cores divided by 2.
 * ``cores`` - this sets the *maximum* number of threads to the number of CPU cores.
 * ``<number>`` - the sets the *maximum* number of threads to a specific number.  Often ``1`` will provide good performance even for dozens of processes.

The default is ``auto``, but in reality if your child processes are "bursty" in their output, rather than producing a
constant stream of data, a single thread may provide equivalent performance even with hundreds of processes.

##### ``org.nuprocess.softExitDetection`` #####
On Linux and Windows there is no method by which you can be notified in an asynchronous manner that a child process has
exited.  Rather than polling all child processes constantly NuProcess uses what we call "Soft Exit Detection".  When a
child process exits, the OS automatically closes all of it's open file handles; which *is* something about we can be
notified.  So, on Linux and Windows when NuProcess determines that both the STDOUT and STDERR streams have been closed
in the child process, that child process is put into a "dead pool".  The processes in the dead pool are polled to 
determine when they have truly exited and what their exit status was.  See ``org.nuprocess.deadPoolPollMs``.  The default
value for this property is ``true``.  Setting this value to ``false`` will completely disable process exit detection,
and the ``NuProcess.waitFor()" API __MUST__ be used.  Failure to invoke this API on Linux will result in an ever-growing
accumulation of "zombie" processes and eventually an inability to create new processes.

##### ``org.nuprocess.deadPoolPollMs`` #####
On Linux and Windows, when Soft Exit Detection is enabled, this property controls how often the processes in the dead
pool are polled for their exit status.  The default value is 250ms, and the minimum value is 100ms.

