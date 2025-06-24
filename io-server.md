# IO server

## Introduction

UM data flow is unidirectional. Data is written to an IO channel as an integer represented data stream. To bypass bottlenecks created by MPI rank zero (*R_0*, the first process in a parallel application), handling all preprocessing an IO, a sub-model is implemented using IO server processes (IOS) designed to reduce disk latency. This sub-model responds to requests from the atmosphere sub-model.

## Architecture

Scaling of parallel applications is limited by several components: load imbalance, serial computing time, communications time and synchronisation frequency.

Operations such as the gather in STASH handling contributes to each of these, the imbalance arising from the larger communications workload of the gathering PE, serial time from the packing
and actual IO, and the large frequencies of these operations.  The serial components are offloaded to a dedicated IO server, deferring synchronisation for as long as possible. A limit of 'in flight operations' is created to best compromised between resource usage and performance. This limit is sufficiently high so the first operation completes before the limit is reached.

The IO server implements a coupling protocol that is buffered at the sending and receiving (gathering) sides of the operation. The receive side buffering on the IO sever exploits its queue-like nature.

The sending side has two dispatch queue structures. The first is a simple ring buffer, with strict in-order semantics used solely by *R_0*. The second queue is used by all tasks *R_i* for cases where output data is domain decomposed - model fields intended for STASH or dump output channels. This queue is dispatched in-order only with regards to a particular output channel, and each queue element is designed to aggregate (cache) operations destined for the same file. Each queue state can be 'part-filled'. Queue items are dispatched only when needed and request for a new queue resource will attempt to find any completed queue item that can be reused. Hence the use pattern of the second queue is non-deterministic b/w identical runs, and between ranks in the same run.

The goal is to separate at both send and receive sides the time between initiating block message transmission and completion.

## IO Server

The IOS comprises multiple tasks decomposing the logical data stream space (similar to Fortran unit numbers), where each task implements a FIFO operational queue. A client API enables a sub-model to insert operations into the queue. A typical client request contains a control message describing the operation (metadata) and, if required, a payload message containing data associated with the operation. A STASH/dump operational has extra parts, as the 'payload' is an extended metadata block describing the operations required to describe the STASH request containing similar data. Since the STASH mechanism can aggregate field levels into a single request, the metadata comprises a list of records of arbitrary number and length.

Other than a global exception, the IOS cannot provide information back to the client sub-model. This ensures asynchroneity, but raises issues with error-handling.

## Lookup tables and buffering fieldsfiles

UM Fieldsfiles contain a fixed length header (*H*), fixed length index/lookup records (*I*) and variable length data fields (*D*).
*<p style="text-align:center;">H,I_1, I_2,... I_N,D_1,D_2,..,D_N</p>*
Data is written in numeric order over the course of execution. *H* is written at the start, the *I_1, D_1* is generated followed by *I_2, D_2* etc. To minimise file pointer movement the model writes *D_i* immediately but buffers *I_i* in *R_0* internal memory until the file is closed or explicitly synchronised. The bufferring is done within the `model_file` object which is flushed to disk as needed. This caching model creates problems, as the IO server has minimal knowledge of STASH and other parameters.

To resolve this, the `model_file`'s data structures are replicated b/w *R_0* and the IOS task responsible for that Fortan unit. *R_0* never sees a complete record. When *R_0* is instructed to flush the *I_1..I_N* table, the *R_0* copy is sent to the appropriate IOS which merges it with its own, ensuring a complete record prior to data being committed to disk.

Dump outputs are deterministic as the final disk sizes and location are know.

## Design

### Components

The main components of the IO subsystem are:
- `file_manager.F90` : module providing dynamic management of unit numbers
- `io.F90` : interface providing a single entry point to IO for all UM executables.
- `model_file.F90` : module which wraps some of the above commands for common operations such as opening and closing UM fieldsfiles.
- A c-layer providing low-level access to reading/writing from/to disk.
- `ios.F90` : module providing client access to remote IO services. 
- `ios_stash.F90` : module providing client access to pattern specific remote IO services, for fast handling of STASH of Dump requests.
- `io_server_writer.F90`, `io_server_lister.F90`, `ios_queue.F90` : modules comprising the IO server itself.

## Data flow

The Fortran IO module acts as a switch-box. The IO server tasks maintain a thread (OpenMP thread 0, or *T_0*) that maximises its availability for listening for inbound protocol messages. Any received messages and associated data payload is appended to an FIFO queue. The queue is a linked list of a Fortran derived data type `IOS_node_type`. The maximum queue length is the sum of all payloads attached to nodes, and any buffers set aside for receiving asynchronous STASH/dump data. *T_0* will exit its listen loop if it receives a termination message from *R_0* after placing it in the FIFO. The second of the IO sever *T_1* monitors the FIFO queue for objects to process. *T_1* exits its execution loop when the head of the queue contains the termination command.

Communication between *T_0* and *T_1* is via shared memory using a small number of scalar variables:
- The queue length
- The number of items in the queue
- Pointers to the head and tail of the queue.

Correct flushing of such objects is critical to avoid deadlock situations. Extra threads, *T_n*, *n* > 1 wait at the end of the main parallel section of `IOS_init()` until *T_0* and *T_1* have completed.

## Configuration

### General requirements

To use IO servers:
- The application should be compiled with OpenMP.
- The IO server should be configured to use at least two OpenMP threads. It should be possible to run the atmosphere model on any number of OpenMP threads.
- The environment variable `FLUME_IOS_NPROCS` should be set to the total number of processes designated to be used as IO Servers. This value is computed as the number of servers required, multiplied by the size of each server (`IOS_tasks_per_server`). The number of IO servers that provide benefit will range between one and ten, after which adding more uses more resources without any additional benefit.

### `IOSCNTL` namelist

The IO server namelist `IOSCNTL` contains options for activation and configuration.
- `IOS_Spacing` : Select the spacing of the IO servers. The servers are interleaved throughout the other processess with this periodicity.
- `IOS_tasks_per_server` : How many MPI tasks will be used for each IO server. The total number of tasks used by the IO Server subsystem is this multiplied by the number of servers. **The parallelism specified must be the same or less that the decomposition of the atmospheric model in the north south direction.**
- `IOS_Offset` : The number of tasks to skip before starting allocating IO Server tasks with the specified periodicity. If possible, **set this such that IO servers are not allocated to nodes which include polar rows, as these generally have high communication costs already.**
- `IOS_force_threading_mode` : Force the model to behave assuming the threading model you specify. For experts only, will cause errors if the MPI implementation doesn't conform to that specified. Useful for users confident their MPI implementation misreports its capabilities.
- `IOS_buffer_size` : The maximum size (in MB) of the IO Sever's FIFO queue. In general, bigger values will give improved performance, but take care not to oversubscribe the node's total memory.
- `IOS_concurrency_max_mem` : Maximum size (in MB) of the general purpose IO queue. Used only on *R_0*.
- `IOS_concurrency` : Number of IO operations queued on a client (*R_0*). Large values allow tolerance of delays in the IO servers responding to messages (usually caused by waiting for locks on the MPI library). Increasing this value increases memory usage on *R_0*. If accelerated STASH/Dumps are **not** enabled, the overhead will be roughly the size of a model level multiplied by the concurrency. 
- `IOS_backoff_interval` : The primary threads on the IO Servers use spin loops to wait for for work. Where threads share hardware resources (e.g. hyperthreading) the spin loop can slow down other threads. A larger polling interval makes checking the spin loops less aggressive, at the expense of latency (values is in microseconds)
- `IOS_timeout` : The value in seconds that the model will wait for a message to complete. 
- `IOS_local_ro_files` : Because IO servers treat all operations in strict order opening, a read-only file on the IO server can result in waiting for other pending operations to complete. The user can instead opt for all read-only only files to be managed by *R_0*, bypassing IO servers.
- `IOS_Unit_Alloc_Policy` : Static choices result in a fixed assignment for the run, while dynamic choices allows some units to be remapped by rotating the target server, by querying servers and selecting a lightly loaded one.
- `IOS_rank_client_lb` : The atmospheric rank used for the load balancing using client info.
- `IOS_queue_drain_rate` : The assumed rate of data drain (MiB/s) from an I/O server queue.
- `IOS_Verbosity` : Sets an initial value for the level of printing output.
- `IOS_num_threads` : Number of threads used by the I/O server. This may be different from the number used by the atmosphere. If so, extreme care must be taken to avoid thread binding.
- `IOS_acquire_model_prsts` : Whether the above setting should be overridden by the general output specified for the main model.
- `IOS_Interleave` : Whether the IO severs should be in sequential order or whether the tasks from different servers should interleave.
- `IOS_serialise_mpi_calls` : Ensures MPI calls are called simultaneously on different threads on the same task.
- `IOS_thread_0_calls_mpi` : Prohibits MPI calls on all threads except the first (*T_0*). *T_0* will proxy MPI calls needed by other threads.
- `IOS_RelayToSlaves` : Selects how *R_0* sends command and control, together with metadata, to the members of a parallel IO server. The default treats all IO servers as peers, which can generate high messaging rates from *R_0* and high memory requirements. Selecting this option will send messages only to the first task in the team.
- `IOS_Decomp_Model` : Selects how atmosphere tasks bind to members of a parallel IO Sever team.
- `IOS_async_stats` : Will report details about communication sizes used by accelerated STASH/dump protocols.
- `IOS_Lock_Meter`: Will report time waiting to access the MPI library and the shared memory FIFO queue on each thread. No time will be shown for MPI locks if the MPI library is multi-threaded.
- `IOS_debug_no_write` : Turns off actual disk output of accelerated STASH operations.
- `IOS_debug_no_packing` : Ignores the packing profile of accelerated STASH output and writes all data uncompressed. 
- `IOS_debug_no_domaining` : Ignores the spatial profile of accelerated STASH output and writes full fields in all cases.
- `IOS_together_end` : Resets the values of `IOS_Offset` and `IOS_Spacing` so the IO servers sit in a contiguous block at the end of the atmosphere MPI tasks. Useful when running setups which use rank reordering for the atmosphere.
- `IOS_use_async_stash` : Chooses accelerated STASH method. These offer overall best performance.
- `IOS_use_async_dump` :  Chooses accelerated Dump method. These offer overall best performance.
- `IOS_async_levs_per_pack` : Selects the number of model levels to coalesce into a single transition with an IO server. Asynchronous model output will accumulate model levels to output until this threshold is reached. This gives fewer, larger messages b/w the model and IO servers at the expense of higher memory usage.
- `IOS_as_concurrency` : Selects concurrency for accelerated STASH and dump methods. Works similarly to the general concurrency setting above, but for field data requests.
- `IOS_async_send_null` : This includes null data results giving consistent message sizes, at the expense of larger bandwidth.
- `IOS_use_helpers` : If the model has surplus OpenMP threads (i.e. more than two for an IO server) it allows them assist with low level I/O e.g. data fetch.
- `IOS_enable_mpiio` : Allows STASH and Dump record writes to bypass low-level C IO interface and instead be written in parallel with MPI-IO semantics.
- `IOS_no_barrier_fileops` : When enabled, the last IO server that reaches the `IOS_Action_FileOp` sync point performs the action while other IO Servers can continue their work. Otherwise, the `IOS_Action_FileOp` is always performed by a specific IO Server which has to wait for all the other IO Servers to reach the sync point.

In general, a server misconfiguration will result in the model executing without an IO servers active, and any MPI tasks assigned will idle for the duration of the program and IO will be performed by *R_0*.

Using an IO server perturbs the normal layout of the atmosphere processes, particularly output files from each process are named with its global rank, not the rank within the atmosphere model. Certain configurations may set the first task to be an IO server, will sends IO sever output to `*.leave` files.

Introducing IO servers changes the decomposition of atmospheric tasks, which will impact bit reproducibility.

### Output files

The stdout/Fortran unit 6 output files from IOS tasks are names consistently with the atmosphere files as `<experiment>.fort6.pe<num>` where `<num>` is the global MPI task rank. Changing IOS parameters will alter the file contents. **In bad configurations, the output of rank zero may be from an IOS task and copied into the "leave" file.**

Additional files produced by IOS server tasks:
- `ioserver_log.<num>` where `<num>` is again the global rank. This file tracks memory use of the IO Server in three columns; time, queue size in words and change in queue size in words. **The file can be loaded into gnuplot directly.**, e.g. `plot ioserver_log.0031`.