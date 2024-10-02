# Code management and structure

The UM Source code can be viewed at https://code.metoffice.gov.uk/trac/um using your UK Met Office (UKMO) Science Repository Service (MOSRS) login. This is the same service that you will use to access ACCESS `rose/cylc` suites for 21stCentury Weather.

It you want to make UM code changes, the user is advised to complete the following tutorials: http://metomi.github.io/fcm/doc/user_guide/getting_started.html

There are many sources of information regarding [`rose`](https://www.metoffice.gov.uk/research/approach/modelling-systems/rose) and `cylc` - the scheduling software used to execute the UM to simulate the atmosphere. `Cylc` has an active [discourse group] (https://cylc.discourse.group).  The homepage for `Rose` documentation is [here](https://metomi.github.io/rose/doc/html/index.html) which contains a short tutorial.

A good place to start is to read through the material available here:

https://code.metoffice.gov.uk/doc/um/latest/um-training/getting-started.html

We will run through the UM tutorial in the next section.

As mentioned before, the UM is a very complex piece of software whose operation is governed by a vast array of plain-text (ASCII) files including `namelist` files which provide information to every component of the UM. The `rose/cylc` framework provides a graphical-user interface (GUI) and a simple set of master control files which (hopefully) shield the user from the complex management tasks of the entire UM configuration, and allow them to focus on their particular field of interest.

## Compilation 

The UM code is compiled using `fcm make`. The `fcm` executable is the UKMO wrapper around the version control software `svn` which is like `git` but older and harder to use. 

- Rose suites can often trigger UM compilations. The rose-app.conf file contains an `[env]` section which defines variables that control the compilation settings. These can be viewed through the `'rose config-edit'` GUI.
- Building from MOSRS requires authentication.Wwhen you wish to access an `fcm` or `svn` URL on the MOSRS in an `fcm make` app, it is necessary instead to access the local mirror of that URL, e.g. `fcm:um.xm/trunk` instead of `fcm:um.x/trunk`. This is true both when providing a path to a central configuration file, and when providing a path to any locations from which a task will extract source code. 
- When modifying a `rose-app.conf` file : 
    a. `rose-app.conf` variables may not refer to each other. 
    b. The position of a statement relative to the `include =` line is important. 
- Global, file, Path and Couple overrides for flags exist.

It is unlikely you will have to alter any parameters that control UM compilation while running your suite, but it's useful to know what is happening "under the hood."

## Parallelisation.

Atmospheric simulations must be run across multiple, separate processors as there doesn't exist a single processor with sufficient speed and memory availability to run any useful simulation by itself.

The UM uses horizontal domain decomposition, i.e. the latitude/longitude grid is parceled up into rectangular subdomains, each containing a complete set of vertical levels.  Typically, a single domain will run on a separate processor (which typically will have multiple processing cores). Each processor has access to its own memory space.

Regular inter-processor communication is required so that information can be transferred between neighbouring subdomains. This is achieved by extending the local subdomain by a number of rows or columns in each dimension, to form a *halo* around the data. This halo region holds data which is communicated from adjacent processors, so allowing each processor to have access to neighbouring processors’ edge conditions. The UM uses different halo sizes for different variables. Variables that have no horizontal dependencies in calculations will have no halos while those that have will have single point halos. Some variables (primarily advected quantities) have extended halos, the size of which may be specified in the GUI/input namelists.

Further information regarding how halos are used between each processor can be found at : https://code.metoffice.gov.uk/doc/um/latest/papers/umdp_C71.pdf

(These notes will summarise the relevant sections of that document in due course)

## User-definable 2D decomposition ## 

If you are generating a new subdomain, or creating an ACCESS suite to run at a new resolution, it's likely that you will have to pay attention to some of these issues.

- The processors are numbered from the bottom left corner, along the rows. For example for a 4 x 2 decomposition the `pe` (processing-element) numbers would be:

                4 5 6 7 

                0 1 2 3 

- As far as possible each processor has the same local row length and number of rows. However, if this is not possible, the extra points in a row are distributed symmetrically. This means that in a global model each processor has the same number of points per row as the processor on the opposite side of the globe. Extra rows are distributed to the southern-most processor for the ENDGame UM core. The remaining extra rows are distributed symmetrically around the equator, starting at the processor(s) closest to the equator. 
- Additionally, a shared memory parallelisation is available based on OpenMP compiler directive technology. This works underneath the parallel decomposition layer with each processor being able to utilise a number of threads to share the work available. This is done by subdivision of independent iterations of the most costly loops. 
- Running the UM on a parallel computer requires attention to the following inputs:
    - The ”job submission” method and resource ”directives” required. This is currently set up in the Rose suite.rc files. 
    - The number of processors in the North-South / East-West directions. Some experimentation may be re- quired to find the arrangement giving optimum performance. It is typical (although not universally the case) for a decomposition with smaller numbers of processors in the East-West direction than in the North-South direction to give best performance. The atmosphere model is restricted in the possible decompositions in that (for efficient communications over the poles) the number of processors in the East-West direction has to be either one, or an even number. 
    - The number of OpenMP threads. Most model configurations work best with 1 or 2 threads currently, but this will depend on your computer architecture and more may be useful in some circumstances. 
- The following tweaks can improve runtime:
    - Disabling timer facility.
    - The number of OpenMP threads (mentioned above) can be tuned for best performance also. There is little experience with this yet but 1 or sometimes 2 threads tend to be the best currently. 
    - Altering the number of segments or segment size in the Short/Long Wave radiation and convection scheme. Precipitation, Boundary Layer and Gravity Wave Drag allow tuning of segment size. Recommended values can be found in the UM metadata file. 
    - I/O servers may be utilised to speed up model output. The default (synchronous) mode has been well tested. The asynchronous mode for STASH and dumps is newer and has had less testing at present, but may give better performance. 
    - Section C96 describes GCOM and MPI methods.

STASH is the name given the files and formats that control UM outputs (i.e. which variables are output at what frequency)

## Namelist control ##

Below is section describing the main `namelists` used to control the UM executable. These will be configured in your `rose/cylc` suite and should not require any alteration. But if something goes wrong, it's good to know where to start to investigate the problem.

- The UM uses serval Control Files which provide namelists:
    - ATMOSCNTL — atmosphere-only runtime namelist items, especially science options. 
    - IOSCNTL — settings for the IO server
    - RECONA — control of the atmosphere reconfiguration
    - SHARED — namelists common to both the atmosphere sub-model and reconfiguration 
    - SIZES — sizes defining model domain, number of levels etc. 
    - STASHC — control of STASH diagnostic output system
- For 21stCentury users, the most important sections are the RECONA (reconfiguration control) and STASHC (controlling user outputs). As mentioned before, every time you run an experiment which moves from a lower resolution domain into a higher one, you will be using RECONA.
- The top level script that runs the Unified Model atmos is um-atmos and may be found at, for example: `$UMDIR/cylc-run/vn9.1_prebuilds/share/[configuration]/build-atmos/bin/um-atmos` or http://fcm2/projects/UM/browser/UM/trunk/bin 
- A dedicated IO server can be specified with namelist `io_control`
- `UM_SHELL` is the Top Level routine. The history file contains only control variables required for restarts. 
- Running `NRUN` (normal run) as `CRUN` (continuation run) - required to support long-running climate suites where `CRUN` data isn't otherwise available as its scattered across many files.
- Domain decomposition, even for a single processor job (1x1), is done by the `DECOMPOSE ATMOS` and `CHANGE DECOMPOSITION` routines, followed by the routine `DERVSIZE` to derive the sizes needed for dynamic allocation of memory for all the main model arrays. 
- Routine `U_MODEL` houses `ATM_STEP` (atmospheric step) which does the time integration. ENDGame specific versions are `EG_ATM_STEP`, with filename `ATM_STEP_4A`.
- Time updateding of NetCDF climatology fields is possible
