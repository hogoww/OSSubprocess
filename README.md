# OSSubprocess

Forking OS processes from Pharo language

##Table of Contents

  * [OSSubprocess Documentation](#ossubprocess-documentation)
    * [Table of Contents](#table-of-contents)
    * [Summary](#summary)
        * [Funding](#funding)
        * [Status](#status)
        * [OSProcess influence](#osprocess-influence)
    * [Installation](#installation)
    * [Introduction to the API](#introduction-to-the-api)
    * [Child exit status](#child-exit-status)
      * [OSSVMProcess and it's child watcher](#ossvmprocess-and-its-child-watcher)
      * [Accessing child status and interpreting it](#accessing-child-status-and-interpreting-it)
    * [Synchronism and  how to read streams](#synchronism-and--how-to-read-streams)
      * [Synchronism vs asynchronous runs](#synchronism-vs-asynchronous-runs)
      * [When to process streams](#when-to-process-streams)
      * [Facilities for synchronous calls and streams processing at the end](#facilities-for-synchronous-calls-and-streams-processing-at-the-end)


## Summary

OSSubprocess is a software project that allows the user to spawn Operatying System processes from within Pharo language. The main usage of forking external OS processes is to be able to execute OS commands (.e.g `cat`, `ls`, `ps`, `cp`, etc) as well as arbitrary shell scripts (.e.g `/etc/myShellScript.sh`) from Pharo. 

An important part of OSSubprocess is how to manage standard streams (`stdin`, `stdout` and `stderr`) and how to provide an API for reading and writing from them at the language level. 

#### Funding
This project is carried on by Mariano Martinez Peck and it is sponsored by Pharo Consortium.

#### Status
It was decided together with Pharo Consortium that as a first step, we should concentrate first on making it work on OSX and Unix. If the tool proves to be good and accepted, we could, at a second step, try to add Windows support. 
OSSubprocess is still in an exploring phase and so it might be unstable. Use this tool with that in mind. That being said, all tests are green in the tested platforms.

#### OSProcess influence   
OSSubprocess is highly influenced by a subset of the OSProcess project. There are parts which we even copied and adapted them (OSSPipe, OSSAttachableStream, OSSUnixProcessExitStatus). Other parts, we took them as inspiration (the idea of ThisOSProcess representing the VM process, the child watcher, and many others). In addition, OSSubprocess currently uses some of the OSProcess **plugin** (not OSProcess image side), such as the SIGCHLD handler or the creation of pipes. 

## Installation
**OSSubprocess currently only works in Pharo 5.0 with Spur VM**. Until Pharo 5.0 is released, for OSSubprocess, we recommend to always grab a latest image and VM. You can do that in via command line:

```bash
wget -O- get.pharo.org/alpha+vm | bash
```

Then, from within Pharo, execute the following to install OSSubprocess:

```Smalltalk
Gofer it
	package: 'OSSubprocess';
	package: 'OSSubprocessTests';
	url: 'http://smalltalkhub.com/mc/marianopeck/OSSubprocess/main';
load.
```

Important: Do not load OSProcess project in the same image of OSSubprocess because the latter won't work. 

## Introduction to the API
OSSubprocess is quite easy to use but depending on the user needs, there are different parts of the API that could be used. We start with a basic example and later we show more complicated scanarios.

```Smalltalk
OSSUnixSubprocess new	
	command: '/bin/ls';
	arguments: (Array with: '-la' with: '/Users');
	createAndSetStdoutStream;
	runAndWaitOnExitDo: [ :process :outString  |	
		outString inspect 
	]
```

Until we add support for Windows, the entry point will always be OSSUnixSubprocess, which should work in OSX, Linux and others Unix-like. You can read it's class comments for details. 

A subprocess consist of at least a command/binary/program to be executed (in this example `/bin/ls`) plus some optional array of arguments.

The `#command:` could be either the program name (.e.g `ls`) or the full path to the executable (.e.g `/bin/ls`). If the former, then the binary will be searched using `$PATH` variable and may not be found.  

For the `#arguments:` array, each argument must be a different element. In other words, passing `Array with: '-la /Users'` is not correct since those are 2 arguments and hence should be 2 elements of the array. It is also incorrect to not specify `#arguments:` and specify the command like this: `command: '/bin/ls -la /Users'`. OSSubprocess does *not* do any parsing to the command or arguments. If you want to execute a command with a full string like `/bin/ls -la /Users`, you may want to take a look to `#bashCommand:` which relies on shell to do that job.

With `#createAndSetStdoutStream` we are saying that we want to create a stream and that we want to map it to `stdout` of the child process. Since they are not specified, `stderr` and `stdin` will then be inherit from the parent process (Pharo VM process). If you comment the line of `#createAndSetStdoutStream` and run the example again, you can see how the output of `/bin/ls -la /Users` is printed in the terminal (where you launched your Pharo image).

Finally, we use the `#runAndWaitOnExitDo:` which is a high level API method that runs the process, waits for it until it finishes, reads and the closes `stdout` stream, and finally invokes the passed closure. In the closure we get as arguments the original `OSSUnixSubprocess` instance we created, and the contents of the read `stdout`. If you inspect `outString` you should see the output of `/bin/ls -la /Users` which should be exactly the same as if run from the command line.

## Child exit status
When you spawn a process in Unix, the new process becomes a "child" of the "parent" process that launched it. In our case, the parent process is the Pharo VM process and the child process would be the one executing the command (in above example, `/bin/ls`). It is a responsability of the parent process to collect the exit status of the child once it finishes. If the parent does not do this, the child becomes a "zombie" process. The exit status is an integer that represents how the child finished (if sucessful, if error, which error, if received a signal, etc etc.). Besides avoiding zombies, the exit status is also important for the user to take actions depending on its result. 

### OSSVMProcess and it's child watcher

In OSSubprocess, we have a class `OSSVMProcess` with a singleton instance accessed via a class side method `vmProcess` which represents the operating system process in which the Pharo VM is currently running. OSSVMProcess can answer some information about the OS process running the VM, such as running PID, children, etc etc. More can be added later. 

Another important task of this class is to keep track of all the launched children processes (instances of `OSSUnixSubprocess`). Whenever a process is started it's registered in OSSVMProcess and unregister in certain scenarios (see senders of ``#unregisterChildProcess:``). We keep a  list of all our children, and ocasionally prune all those that have already been exited. 

This class takes care of running what we call the "child watcher" which is basically a way to monitor children status and collect exit code when they finish. This also guarantees not to let zombies process. As for the implementation details, we use a SIGCHLD handler to capture a child death. For more details, see method `#initializeChildWatcher`.
 
*What is important here is that wether you wait for the process to finish or not (and no matter in which way you wait), the child exit code will be collected and stored in the `exitStatus` instVar of the instance of `OSSUnixSubprocess` representing the exited process, thanks to the `OSSVMProcess` child watcher.* 

### Accessing child status and interpreting it
No matter how you waited for the child process to finish, when it exited, the instVar `exitStatus` should have been set. `exitStatus` is an integer bit field answered by the `wait()` system call that contains information about exit status of the process. The meaning of the bit field varies according to the cause of process exit. Besides understanding `#exitStatus`, `OSSUnixSubprocess` also understands `#exitStatusInterpreter` which answers an instance of `OSSUnixProcessExitStatus`. The task of this class is to simply decode this integer bit field and provide meaningful information. Please read it's class comment for further details. 

In addition to `#exitStatus` and `#exitStatusInterpreter`, `OSSUnixSubprocess` provides testing methods such us `#isSuccess`, `#isComplete`, `isRunning`, etc. 

Let's see a possible usage of the exit status:

```Smalltalk
OSSUnixSubprocess new	
	command: '/bin/ls';
	arguments: (Array with: '-la' with: '/noneexisting');
	createAndSetStdoutStream;
	createAndSetStderrStream;
	runAndWaitOnExitDo: [ :process :outString :errString |	
		process isSuccess 
			ifTrue: [ Transcript show: 'Command exited correctly with output: ', outString. ]
			ifFalse: [ 
				"OSSUnixProcessExitStatus has a nice #printOn: "
				Transcript show: 'Command exit with error status: ', process exitStatusInterpreter printString; cr.
				Transcript show: 'Stderr contents: ', errString.
			] 
	]
```

In this example we are executing `/bin/ls` passing a none existing directory as argument. 

First, note that we added also a stream for `stderr` via `#createAndSetStderrStream`. Second, note how now in the `#runAndWaitOnExitDo:` we can also have access to `errString`. If you run this example with a `Transcript` opened, you should see something like:

```
Command exit with error status: normal termination with status 1
Stderr contents: ls: /noneexisting: No such file or directory
```

The `normal termination with status 1` is the information that `OSSUnixProcessExitStatus` (accessed via `#exitStatusInterpreter`) decoded for us. "Normal termination" means it was not signaled or terminated, but rather a normal exit. The exit status bigger than zero (in this case, 1), means error. What each error means depends on the program you run. 
 
The second line of the `Transcript` is simply the `stderr` conents.  


## Streams management
There are many parts of the API and features of OSSubprocess which are related to streams. In this section we will try to explain these topics.

### Handling pipes within Pharo
Besides regular files (as the files you are used to deal with), Unix provides **pipes**. Since Unix manages pipes quite polimorphically to regular files, you can use both for mapping standard streams (`stdin`, `stdout` and `stderr`). That means that both kind of files can be used to communicate a parent and a child process. 

For regular files, Pharo already provides `Stream` classes to write and read from them, such as `StandardFileStream` and subclasses. For pipes things are different becasue there is a write end and a read end (each having a different file descriptor at Unix level). For this purpose, we have implemented the class called `OSSPipe` which represents a OS pipe and it is a subclass of `Stream`. `OSSPipe` contains both, a `reader` and a `writer`, and it implements the `Stream` protocol by delegating messages to one or the other. For example `OSSPipe>>#nextPutAll:` is delegated to the writer while `#next` is delegated to the reader. That way, we have a Stream-like API for pipes. For more details, please read the class comment of `OSSPipe`.

The question is now what type of streams are the 'reader' and 'writer' instVars of a `OSSPipe`. As I said before, the write end and read end of a pipe have different file descriptor at Unix level, hence they are seem as two different file streams. When we create a pipe via a system call, we directly get both file descriptors. That means that in Pharo, once we created a OS pipe, we already have both, the read and the write. Ideally, we could use `StandardFileStream` for this case too. However, those file streams were *already opened* by the system call to make the pipes. To solve this issue, we created what we call `OSSAttachableFileStream` which is a special subclass of `StandardFileStream` and that can be *attached* to an already existing and opened file. 

To conclude, we use a system call to create an OS pipe. At Pharo level we represent that pipe as an instance of `OSSPipe`. And a `OSSPipe` has both, a reader and a writer which are both instances of `OSSAttachableFileStream` and that have been attached to the reader and writer end of the pipe. 

### Regular files vs pipes
As we said before, both regular files or pipes can be used for mapping standard streams (`stdin`, `stdout` and `stderr`) and OSSubprocess supports both and manages them quite polymorphically thanks to `OSSPipe` and `OSSAttachableFileStream`. But the user can decide use one or another. Pipes are normally faster since they run in memory. On the contrary, files may do I/0 operations even with caches (at least creating and deleting the file). With pipes you do not have to handle the deletion of the files as you do with regular files. You can read more about "regular files vs pipes" in the internet and come yourself to a conclusion. 

There is only one problem with pipes that you should be aware of and it's the fact that you may get a deadlock in certain cirtcunstances. See XXX for more details. 

### Customizing streams creation
Let's consider this example:

```Smalltalk
| process |
process := OSSUnixSubprocess new	
			command: '/bin/ls';
			arguments: (Array with: '-la' with: '/noneexisting');
			defaultWriteStreamCreationBlock: [OSSVMProcess vmProcess systemAccessor makeNonBlockingPipe];
			createAndSetStdoutStream;
			stderrStream: '/tmp/customStderr.txt' asFileReference writeStream;
			defaultReadStreamCreationBlock: [ process createTempFileToBeUsedAsReadStreamOn: '/tmp' ];
			createMissingStandardStreams: true.	
process runAndWait.  
process isSuccess 
	ifTrue: [ Transcript show: 'Command exited correctly with output: ', process stdoutStream upToEnd. ]
	ifFalse: [ 
		"OSSUnixProcessExitStatus has a nice #printOn: "
		Transcript show: 'Command exit with error status: ', process exitStatusInterpreter printString; cr.
		Transcript show: 'Stderr contents: ', process stderrStream upToEnd.
	].
process closeAndCleanStreams.
```

There are many things to explain in this example. First of all, we are not using the API `#runAndWaitOnExitDo:` and so certain things must be done manually (like retrieving the contents of the streams via `#upToEnd` or like closing streams with `#closeAndCleanStreams`). Now you get a better idea of what `#runAndWaitOnExitDo:` does automatically for you. 

With the methods `#stdinStream:`, `#stdoutStream:` and `#stdoutStream:` the user is able to set a custom stream for each standard stream. The received stream could be either a `StandardFileStream` subclass (as is the result of `'/tmp/customStderr.txt' asFileReference writeStream`) or a `OSSPipe` (as is the result of `OSSVMProcess vmProcess systemAccessor makeNonBlockingPipe`).

If you do not want to create a custom stream (like above example of `#stderrStream:`) but you still want to create a default stream that would be mapped to a standard stream, then you can use the methods `#createAndSetStdinStream:`, `#createAndSetStdoutStream` and `#createAndSetStderrStream`. Those methods will create *default* streams. The *default* streams are defined by the instVars `defaultWriteStreamCreationBlock` and `defaultReadStreamCreationBlock`. And these can be customized too as shown in above example. In this example, all write streams (to be used for `stdout` and `stderr`) will be pipes (forget for the moment about the none blocking part). And all read streams (only used by `stdin`) will be temp files automatically created in `/tmp`. This feature is useful if you want to change the way streams are created without having to specially create each stream. You can see how `OSSPipeBasedUnixSubprocessTest` and `OSSFileBasedUnixSubprocessTest` use these in the method `#newCommand`. 

*If you see `OSSUnixSubprocess>>#initialize` you will see that by default we create pipes for all type of streams.* 

Previously, we said that if the user does not specify a stream to be mapped to one of the standard streams (by any of the means explained above), the child will inherit the one of the parent process (VM Process). With the method `#createMissingStandardStreams:` one can change that behavior, and instead, create default streams for all none already defined streams. In above example, the user defined `stdout` via `#createAndSetStdoutStream` and `stderr` via `#stderrStream:`. So, by doing `#createMissingStandardStreams: true`, a default stream (pipe) will be created and mapped to the missing streams (only `stdin` in this example). Of course, if you are happy with default streams creation, you can avoid having to declare each stream and simple enable all. But keep in mind the costs of managing streams that you may not use at all. For this reason, the default value of `createMissingStandardStreams` is `false`.

Finally, note that since we are not using the API `#runAndWaitOnExitDo:` in this case, we must explicitly close streams via the message `#closeAndCleanStreams`. This method will also take care of deleting all those streams which were regular files (not pipes). 


## Synchronism and  how to read streams 


###  Synchronism vs asynchronous runs 
We call synchronous runs when you create a child process and you wait for it to finish before going to the next task of your code. This is by far the most common approach since many times you need to know which was the exit status (wether it was success or not) and depending on that, do something. 

Asynchronous runs are when you run a process and you do not care it's results for the next task of your code. In other words, your code can continue its execution no matter what happened with the spawned process. For this scenario, we currently do not support any special facility, althought it is planned. If you would be interested in this functionallity, please contact me.  

### When to process streams
If we have defined that we wanted to map a standard stream, it means that at some point we would like to read/write it and **do something with it**. 
There are basically two possibilities here. The most common approach is to simply wait until the process has finished, and once finished, depending on the exit status, **do something** with the stream contents. What to do exactly, depends on the needs the user has for that process.

The other possibility a user may do, is to define some code that loops while the process is still running. For every cycle of the loop, it retrieves what is available from the streams and do something with it. But in this case, **doing something** is not simply accumulating the stream contents in another stream so that it is processed at the end. By doing something we really mean that the user wants to do something with that intermediate result: print it in a `Transcript`, show a progress bar, or whatever.  

If what you need is the second possibility then you will have to write your custom loop code. You can take a look to `#waitForExitPollingEvery:retrievingStreams:` and `#runAndWaitPollingEvery:retrievingStreams:onExitDo:`


### Facilities for synchronous calls and streams processing at the end
For synchronous calls in which streams processing could be done once the process has exited, we provide some facilities methods that should ease the usage. Synchronous calls mean that we must wait for the child to exit. We provide two ways of doing this. 

#### Semaphore-based SIGCHLD waiting 
The first way is with the method `#waitForExit`. This method is used by the high level API methods `#runAndWait` and `#runAndWaitOnExitDo:`. The wait in this scenario does **not** use an image-based delay polling. Instead, it uses a semaphore. Just after the process starts, it waits on a semaphore of the instVar `mutexForSigchld`. When the `OSSVMProcess` child watcher receives a `SIGCHLD` singal, it will notify the child that die with the message `#processHasExitNotification`. That method, will do the `#signal` in the semaphore and hence the execution of `#waitForExit` will continue.

> **IMPORTANT** In this kind of waiting there is no polling and hence we do not read from the streams periodically. Instead, we read the whole contents of each stream at the end. Basically, there is a problem in general (deadlock!) with waiting for an external process to exit before reading its output. If the external process creates a large amount of output, and if the output is a pipe, it will block on writing to the pipe until someone (our VM process) reads some data from the pipe to make room for the writing. That leads to cases where we never read the output (because the external process did not exit) and the external process never exits (because we have not read the data from the pipe to make room for the writing).

> Another point to take into account is that this kind of wait also depends on the child watcher and `SIGCHLD` capture. While we do our best to make this as reliable as possible, it might be cases where we miss some `SIGCHLD` signals. There is a workaround for that (see method `#initializeChildWatcher`), but there may still be issues. If such problem happens, the child process (at our language level) never exits from the `#waitForExit` method. And at the OS level, the child would look like a zombie becasue the parent did not collect the exit status. 

#### Delay-based polling waiting
The other way we have for waiting a child process is by doing a delay-based in-image polling. That basically means a loop in which we wait some miliseconds, and then check the exit status of the process. For this, we provide the method `#waitForExitPollingEvery:retrievingStreams:`, and it's high level API method `#runAndWaitPollingEvery:retrievingStreams:onExitDo:`.

It is important to note that as part of the loop we send `#queryExitStatus`, which is the one that sends the `waitpid()` in Unix. That means that we are fully independent of the child watcher and the `SIGCHLD` and hence much more reliable than the previous approach.

In addition, we provide a `retrievingStreams:` boolean which if true, it reads from streams as part of the loop. This solves the problem mentioned before where there could be a deadlock if there is no room in the pipe for the writing. 

Of course, the main disadvantage of the polling is the cost of it at the language level. The waiting on a semaphore (as with the `SIGCHLD`) has no cost.  

#### Which waiting to use?
If you are using pipes and you are not sure about how much the child process could write in the streams, then we recommend the polling approach (with `retrievingStreams:` with `true`). 

If you are using pipes and you know the process will not write much, then you can use both approaches. If you want to be sure 100% to never ever get a zombie or related problem, then we think the polling approach is a bit more reliable. If you can live with such a possibility, then maybe the `SIGCHLD` approach could work too. 

If you are using files instead of pipes, then it would depend on how long does the process take. If it is normally short, then the polling approach would be fine. If it is a long process then you may want to do the `SIGCHLD` approach to avoid the polling costs. But you can also use the polling approach and specify a larger timeout. 

*As you can see, there is no silver bullet. It's up to the user to know which model would fit better for his use-case. If you are uncertain or you are not an expert, then go with the polling approach with `retrievingStreams:` on `true`.*


## Environment variables

## Shell commands












