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
- `IOS_RelayToSlaves` : Selects how *R_0* sends command and control, together with metadata, to the members of a parallel IO server. The 