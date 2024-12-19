# Using the UM

Let's through a simple UM tutorial. 

https://code.metoffice.gov.uk/doc/um/latest/um-training/getting-started.html

(To do - insert user screenshots)

- Use `rose bush` (web viewer) to look at logs, stderr and stdout.
- `rose stem` is a CI tool for the UM trunk.
- `rosie` handles suite storage and discovery, based on SVN and a web-accessed database (MOSRS)

## Reconfiguration ##

As previously mentioned, a lot of the work to configure the UM to run over a new domain, at a new resolution, is performed by the *Reconfigure* executable. 

The reconfiguration is part of the suite of UM supporting executables and is a standalone program. The re-configuration allows the user to modify UM dump files which contain the necessary data to define the initial conditions for a model run. The format of UM dump files is described in [UMDP-F03](https://code.metoffice.gov.uk/doc/um/latest/papers/umdp_F03.pdf) . The reconfiguration program only processes atmospheric data.

The reconfiguration is run as a task within a Rose suite. The initial conditions are input in the UM app namelist and configurable through the UM Rose GUI. Not all UM suites require a reconfiguration task and thus this may or may not be included in a suite dependency graph as required.

- Reconfiguration allows the user to modify UM dump files which contain information to define the initial conditions. A file needs reconfiguring if:
    - Upgrade to new model release
    - Change of resolution or area
    - Configure from an ancillary file to overwrite existing data or form a new field
    - New prognostics or tracers
    - Starting from a GRIB file
- Reconfigure can also initialise from NetCDF, initialise the VAR LS state dumps.
- Any data from external sources to the model must be at the same resolution and on the same domain as the output dump.
- Reconfiguration can be defined using a GUI
- Reconfiguration is compiled using `fcm_make`
- Ancillary files can updated standard fields via reconfiguration.
- You can 'transplant' or replace input data.
- It is strongly recommended to use dedicated ancillary files at the new resolution (i.e. don't rely on the UM to interpolate for you).
- Make sure your orography, land-sea mask and sea-ice are consistent. Don't request orography or land-sea mask from an ancillary as it will trigger interpolation.
- Reconfiguration as a coastal adjustment step, adjusting land specific fields at coastal points where horizontal interpolation uses a mixture of land and sea points. The spiral coastal adjustment scheme is recommended here.
- Changing vertical resolution must be done with great care!
    - Specify `ETA_THETA` and `ERA_RHO` values to ensure smooth variation b/w levels
    - Use predefined vertical level sets when available.
- Changing horizontal or vertical levels may require changing dynamics and diffusion parameters. Seek advice from dynamics code experts
- Changing horizontal resolution:
    - `UM -> namelist -> Reconfiguration and Ancillary control -> Output dump grid sizes and levels`: Specific number of rows and `row_length`. Number of land points is required if you are reconfiguring to a different horizontal resolution without a land sea mask
    - `UM -> namelist -> Top Level Model control -> Model domain and timestep`: Increased resolution needs a reduction in timestep. If 'period' is kept = 1 day, you need to increase number of timesteps per 'period'. Check STASH diagnostic requests and frequency of radiation calls are still integer multiples of the timestep.
    - Increased resolution requires more memory – you may need to increase 'number of segments' for long/short-wave radiation and convection. Or, increase number of processors.
    - `UM -> env -> Runtime Controls -> Atmosphere Only`: Choose number of processors in North-South and East-West directions. Try to aim for a 'squarer' decomposition. If running on more than one node, ensure there are no unused processors (unless allocated to I/O server or coupled model).
    - To reconfigure ancillaries : `UM -> namelist -> Reconfiguration and Ancillary control -> Configure ancils and initialise dump fields` : The reconfiguration program can interpolate dumps to other resolutions, but ancillary files can only be used at their native resolution. The UM will fail if you try to "Update" ancillary fields from files at the wrong resolution. Reconfiguration will fail if you try to "Configure" from ancillary files at the wrong resolution. 
    - In addition to changing processors, you may need to alter job resources (memory and time). Memory is proportional to grid points (unless you have more processors), time is proportional to grid points and timesteps required. These are set in suite files rather than apps.
    - Changes to dynamics or diffusion parameters will be required when changing horizontal resolution.
- Changing vertical resolution:
    - `UM -> namelist -> Reconfiguration and Ancillary control -> Output dump grid sizes and levels` : Choose number of levels, 'wet' levels (which include moisture calculations) and ozone levels. `ETA_THETA`, `ETA_RHO` levels and the top height of the model (`Z_TOP_OF_MODEL`) are specified in the vertical levels namelist. The 'number of levels+1' `ETA_THETA` values must start at 0.0 and end at 1.0. `FIRST CONSTANT R RHO LEVEL` is where terrain-following surfaces change to constant height. This should be at least 2x maximum orography height in the model domain. The vertical levels namelist file is `vert_lev` which contains number of boundary layer levels (which may now cover the whole depth of the troposphere to allow better boundary layer/convection coupling), deep soil levels (usually four), and cloud levels used in radiation (which should be the minimum of number of wet levels and total number of levels).
    - Cloud info : `UM -> namelist -> UM Science Settings -> General Physics Options -> Large Scale Cloud` : Number of `rhcrit` values must = number of model levels. Regional models often use different sets of low level critical humidity ratio values compared with global models. Diffusion settings may need to be reviewed as they contain level information.
    - `Ancillaries : UM -> namelist -> Reconfiguration and Ancillary control -> Configure ancils and initialise dump fields`: Changes to vertical levels and spacing will impact the ozone ancillary.
    - STASH will need to be updated for new vertical levels (including boundary layer levels), and maybe timestep changes via: `UM -> namelist -> Model Input and Output -> STASH request and profiles`. Avoid the use of timesteps in preference to hours, days etc.
- UM can support up to 150 'tracers' advected using semi-Lagrangian advection scheme. See `UM -> namelist -> Sections 10 11 12 Dynamic settings -> Advection` : 
    - Tracers are initialised  with ITEMS namelist. Ancillary file for tracer initialisation can be generated from `pp` files, ensure STASH, `pp` and grid codes are correct and set `space code = 2` so item is included in the dump.
    - The reconfiguration step checks verification time of the data matches that of the dump being reconfigured into. It is difficult to include a 'test blob' without recreating the ancillary field. An FCM changeset can get generated to solve this.
    - Level dependent constants must match the dump.
- `MURK` is advected using the tracer scheme and can assimilate visibility data. See `UM -> namelist -> UM Science settings -> Section 17 Aerosol (CLASSIC, dust and murk)`
- `CLASSIC`, dust and `MURK` lateral boundary conditions (LBCs) can in included in the LBC files. 
- UKCA model tracers may be selected depending on chemical or aerosol scheme.
- Free tracers is for testing new schemes. See `UM -> namelist -> UM Science settings -> Section 33 Free Tracers`.


## Setting up a new Limited Area Model (LAM) ## 

Setting up the Unified Model for a new limited area domain is relatively straightforward, and will enable you to run higher resolution suites concentrating on your area of interest. However you should at the outset consider the time and effort involved.
- The time will be 2 days to a week or so of dedicated work, to get a smoothly running Limited Area Model (LAM) on a new domain, depending on experience. A similar length of time may be needed to create new ancillary files for the new domain. 
- It is recommended, where possible, that users base their LAMs on one of the Met Office app configurations. This will greatly reduce the work involved in setting up a LAM over your domain of interest, leaving you to consider:
    - the size of domain, horizontal and vertical resolution
    - the availability of initial data and boundary conditions
    - the length of forecast
    - whether ancillary files are available or will need to be created
    - what STASH diagnostic output you will require
    - what computer resources you have available.
- If the user chooses to build a LAM from scratch, or with different settings (resolution, dynamics and/or physics) to those used by the Met Office one will also need to consider:
    - horizontal and vertical resolution (the spacing of grid points/model levels)
    - the model timestep
    - the choice of science parameterizations
    - the order and coefficients of diffusion; these will depend on grid length and timestep.

### Setting up a new LAM domain
- The definition of a UM domains request 8 items:
    - Lat/lon coords of the rotated pole
    - X and Y direction gridlengths – labelled as col and row spacing
    - X and Y grid dimensions in gridlengths – labelled as No. Cols/rows
    - Lat/lon of bottom left corner (relative to rotated pole), labelled as First lat/lon
- Working in the Southern Hemisphere, take extra care in defining the domain coords! The UM can work with imaginary latitude > 90N for the rotated poll but data in the output files could break plotting programs**.
- An alternative and the recommended approach, which gives an identical forecast and a correct map background, is to ”mirror” back to a rotated latitude between 0 and 90N by subtracting from 180, e.g. 110N => 70N. Then swing the rotated longitude through 180 degrees, and also add 180 degrees to the longitude of the bottom left corner. 
- When selecting the new domain, it will save time on tuning the model if you choose a tried and tested resolution for which diffusion coefficients etc. are known. e.g. 0.11 degrees for the current UK NAE model, 0.036 degrees for the UK 4km model, and 0.0189 for AUS2200. 

### Creating new ancillary files ###

- You will want new land/sea masks, orography, soil, vegetation and ozone ancillary files for your new LAM domain.
- Atmosphere ancillary files are created with a program running on UKMO IBM. If you require one outside UKMO, contact Ported UM support at UKMO.
- Examine the land/sea masks around single grid point islands or sea inlets which may create unrealistic effects for surface and boundary fields. After finalising the land/sea mask, create the other ancillaries.
- Enter values to exactly the same precision as you will use in the UM namelists/GUI when setting up suites. Otherwise your runs may fail internal consistency checks in the model. 
- Also make sure longitudes are in the range 0 - 360 degrees as this is preferred for the UM namelists. 

### Obtaining new lateral boundary data ###
- Use CreateBC to create LBCs.
- Setting up RCF (Reconfiguration) and UM. You can use RCF to transplant data from any new ancillary files into the LAM start dump. 
- The orography at edge of the LAM must be the same as used in the driving model. LBC data is defined on the driving model's levels, which need to be the same heights as the LAM levels in the rimwidth where they are defined. Since model level height depends on the height of the orography beneath it, both models must have the same orography within the LBC rimwidth. In the model area immediately inside the LBC rimwidth the orography is slowly relaxed from the driving model to the full resolution orography for the LAM using linear interpolation. This area is called the `blending zone' and it includes the LBC rimwidth. Blending across the rimwidth is achieved by running RCF with the input dump from the driving model to create an output dump for the LAM model to be run. Gridpoints in the blending zone have weights from 1 (orography = driving model) to 0 (orography = LAM). 
- Settings for the width of the blending zone and their weights will depend on the orographic details at the LAM boundary. Rimwidth blending weights should be 1.
- Whenever orography in the driving model changes, the blended LAM orography must be reconfigured!
- To check the model domain is set to LAM use `UM -> namelist -> Top Level Model Control -> Model Domain and Timestep`:
- Use `UM -> namelist -> Top Level Model Control -> LAM Configuration` to set delta_lat/lon, frstlata/firstlona, polelata, polelona as set in LAMPOS.
- Specify blending zone and LBC file using `UM -> namelist -> Top Level Model Control -> LBC Related Options`.
- Set forecast run start-time and forecast length using `UM -> namelist -> Top Level Model Control -> Run control and Time Settings`. Do not exceed the timespan covered by your LBCs!
- Enter your vertical level definitions which must be identical to the file specified for generating your LBCs using `UM -> namelist -> Reconfiguration and Ancillary Control -> Output dump grid sizes and levels`.
- If your land/sea mask was not initialised from an ancillary file you will need to enter the number of land points. You can find this number using RCF. 
- You can configure new ancillaries using `UM -> namelist -> Reconfiguration and Ancillary Control ->configure ancils` and initialise data
- Many suites will be set up (via env var) to immediately pick up the start dump created by reconfiguration. Set this explicitly using `UM -> namelist -> Model Input and Output -> Filenames for model input and output`. 
- If you have a non-standard horizontal resolution, you can revise your choice of diffusion coefficients using `UM -> namelist -> UM Science settings -> Section 13: Diffusion, Filtering`
- Once set up, execute a run short enough to test STASH output or ancillary updates. 
- The UM supports multiple scientific model configurations. Within each sub-model (e.g. atmosphere) there are internal models (e.g. radiation) that can be further sub-divided in the sections sub-menu which control routines for calculating diagnostics, I/O operations etc.
- Science schemes should be readily interchangeable with those from other models. The glue routine separates control and science layers.

## Ancillary and Boundary files 

An ancillary file can contain an ancillary field (temporal boundary conditions for a model forecast ) or a prognostic field (predicted by the model's output). 
There are 3 ways to include an ancillary file in a UM run:
- In the initial dump already. 
- Replaced in the initial dump by configuring from an ancillary file on the correct domain and resolution
- Added to the initial dump as a new ancillary file by configuring from an ancillary file on the correct domain and resolution

The user should note:
- There are advantages to using ancillary files. The user has more control of the data, the interpolation / regridding performed within the ancillary creation program is consistent across fields (e.g. vegetation and soil parameters).
- For testing new suites, interpolation performed using the Reconfiguration executable itself (RCF) is sufficient and ancillary files only become necessary when the new domain is finalised.

Locations of ancillary files are specified in 'Ancil version files'.
- Standard ancillary filenames included 'parm' for constant files (e.g. orography, land-sea mask) or 'clim' for climatologies which have time-dependence (e.g. soil moisture, SSTs).
- New atmospheric ancillary files can be created using UKMO programs (ANTS and Central Ancillary Program) - see https://code.metoffice.gov.uk/doc/ancil/ants/latest/index.html
- Ancillary fields can be 'configured' (replaced or added to start dump by RCF) or 'updated' (modified at specific time intervals). Updating is part of the UM itself.
- Control of ancillary fields in the UM is done via the rose-app.conf file. Look for namelists &ANCILCTA and optionally &ITEMS if fields are to be updated. The ancillary fie header will determine the update frequency.
- Other controls including SST and sea-ice, temporal interpolation of fields, choice of updating land or sea points are coded in subroutine REPLANCA (one in RCF, one in UM).
- LBCs (created with the CreateBC utility) can be termed as an ancillary file.
- New prognostic and ancillary fields can be set up using any STASH item number.
- Setting up user ancillaries is more restrictive and must use a list of reserved STASH item numbers for single and multi-level fields.

## Coupling to Ocean and Sea Ice 

UKMO support coupling with NEMO ocean and CICE sea ice. 21st Century Weather uses the MOM ocean model.

CICE is not dynamically allocated and CPU arrangements must be provided during compilation. This is done by supplying dimensions and PE configuration details in the form of numeric FPP keys which are incorporated in the code during compilation.

Before you can run a coupled model, you need
- Directories containing input files
- NEMO and CICE restart files - usually netCDF files containing the grid definition.
- NEMO and CICE configuration and FCM options. See Rose configuration items : `fcm_make_ocean -> env -> Sources`
- FPP Keys to set all necessary options (including CICE PE config) also need to be set. See `fcm_make_ocean -> env -> Pre-processing`
- NEMO and CICE components are built into a single executable combining both sets of code with an internal direct coupling interface. Hence the OASIS3-MCT coupler plays no part in this internal coupling.

NEMO and CICE have poor error handling (e.g. some NEMO versions terminate with Fortran 'STOP', leaving any processes belonging to other components deadlocked). The model will exceed CPU wall time, while the real reason for failure is an internal NEMO-CICE internal error.

Attempt to ensure all components employ an MPI_ABORT (with non-zero return code to allow rose environment to detect the error) on all processes (possibly via OASIS_ABORT) if it becomes necessary for any of them to prematurely halt execution.
- CICE standard output ice_diag.d is not included in UM output.
- For any failed coupled model run, examine not only the UM outputs but any files created in the job directory produced by the coupler, NEMO or CICE (and there are many!)

## Coupled Models

- UM uses OASIS3-MCT to support multi-way coupling b/w the UM, NEMO-CICE and a wave model (Wavewatch III). Coupler libraries have to be pre-built. The coupler interfaces routines (PRISM System Model Interface Library - PSMILe) are linked during compilation to the UM and other components.
- 21st Century Weather will use NUOPC coupler in line with ACCESS-NRI : https://earthsystemmodeling.org/nuopc/
- To achieve coupling - model states are regridded and passed periodically from one sub-model to another via the coupler. 
- The time-stepping convention is time-averaged atmospheric fields force the ocean, while instantaneous ocean-sea-ice components force the atmosphere (due to the timesteps of the atmosphere being shorter). One or three-hour coupling periods are common practise.
- OASIS3-MCT will handle the regridding between the sub-models.
- For high-resolution configurations the use of ESMF regirdding weights generation tool is recommended.
- Coupled models require less ancillary fields (e.g. no more SST climatologies)

To set up coupling in Rose for NEMO and CICE, you need
- A copy of the coupled model drivers from the MOCI repository to provide the model setup and shutdown processes.
- A Rose "suite.rc" file to compile and run the coupled model, e.g. 
```
     suite conf
     coupled
     fcm_make_drivers
     fcm_make_ocean
     fcm_make_um
```
- Coupler settings, including coupling frequency, location of coupler and associated input and output files. Typical rose configuration settings:
`suite conf -> Machine Options -> Use Environment Modules`: set to `Centrally installed` or `Custom module files`
- `suite conf -> Machine Options-> Science Configuration Module Name:` The module to be employed. This makes the pre-built coupler libraries, NetCDF versions, etc. available to the compilation of all components and at run time.
- `um -> command -> command default` : set to `run_model`
- `um -> env -> Coupled Settings -> COUPLER: "none"` or `"OASIS3-MCT"`(or `"NUOPC")`?
- `um -> env -> RMP_DIR:` Location of OASIS remapping (rmp) weights files

```
    namelist -> Coupling
    -> l_oasis_ocean: Couple to ocean model
    -> l_oasis_wave: Couple to wave model
    -> oasis_couple_freq_ao: Coupling frequency for atmosphere to ocean 
    -> oasis_couple_freq_oa: Coupling frequency for ocean to atmosphere
    -> oasis_couple_freq_aw: Coupling frequency for atmosphere to wave 
    -> oasis_couple_freq_wa: Coupling frequency for wave to atmosphere
```
- Coupling frequency in `namcouple`  files must match the values in UM `oasis_couple_freq` values. `namcouple` frequencies are not automatically updated based on values specified in Rose. Inconsistent settings will lead to deadlock.

OASIS3-MCT performs its coupling transformations in parallel, directly in the PSMLe library layer. Running coupled model components concurrently presents opportunities to achieve optimum load balance, as each component can be independently assigned processors to match the speed of other components. The total speed is constrained by:
- The scalability of the slowest component, i.e. if the atmosphere scales well with lots of PEs but NEMO-CICE degrades with further PEs, adding more PEs won't give more speedup. Adding more PEs to the UM will help but its speed is always limited by exchanging data with NEMO-CICE.
- OASIS3-MCT has very little overhead. 
- Some platforms (gadi?) allow OASIS3-MCT to distribute atmosphere and ocean processes among the same nodes to achieve uniform memory usage.

## STASH

The UM outputs data using the Spatial and Temporal Averaging and Storage Handling (STASH) system. This is rather a byzantine method to determine which variables are sent by default to raw 'PP' binary Fortran output 'fieldsfiles'. (where 'PP' = 'Post Processing').

STASH can be confusing and archaic to begin with. It is very flexible and powerful, but that costs comes with associated complexity.

To access STASH in the Rose GUI:
- `UM -> namelist -> Model Input and Output -> STASH Requests and Profiles -> Stash requests`

The user selects the diagnostics (a.k.a. variables) required for their model run. Each diagnostic has three profiles:
1. dom(ain)_name
2. tim(e)_name
3. use_name

The best way to start is usually copy another app's STASHC namelists: `umstash_streq(:), umstash_domain(:), umstash_time(:)` and `umstash_use(:)` as a starting point.

The top of the STASH requests panel contains numerous macros for tidying and checking the requested stash namelists:
- stashindices.TidyStashTransform : Correct the index of the STASH related namelists
- stashindices.TidyStashTransformPruneDuplicated : Correct the index of the STASH related namelists and
prune duplicates
- stashindices.TidyStashValidate : Check STASH related namelists have the correct index
- stashtestmask.STASHTstmaskValidate : Check that the stash requests are available

It is advisable to work through these macros to check your STASHC namelist is correct before running a suite.

Inside a UM/ACCESS rose/cylc suite, the STASH information is provided via the `install_cold` app, which will link the centralised `STASHmaster` directory to your local `~/cylc-run` directory under `share/etc/ancils/STASHmaster`. Note the format of the `STASHmaster_A` file and be thankful there is now a GUI for editing this file.

New diagnostics are created using the "new" button on the top right of the GUI. A useful feature is to disable (rather than delete) various diagnostics which can be reactivated if required.

Each diagnostic must that a Time, Domain and Usage profile attached. Right-clicking the profile name in the stash requests panel and selecting view provides more detailed information. Essentially:
- Time profiles determine when a diagnostic will be output (e.g. hourly, daily) and any time processing (e.g. accumulation or mean)
- Domain profiles determine the spatial extent (horizontal and vertical) of the output
- Usage profiles specify the output unit number the fieldsfile will be written to. This is a relic of Fortran whereby data files are often written to files according to an integer 'unit number' (rather than a filename). Fieldsfiles unit numbers are reserved for range 60-69 and 151. So Output written to units 60, 61, ... , 69 and 151 is stored in files with extensions `.pp0, .pp1, ..., .pp9` and `.pp10`. As an example, output sent to unit 64 from run task ATMOS will go to the file atmos.pp4. These binary Fieldsfiles can also be periodically opened and closed during a model run, and can be used like a scratch disk. This is called reinitialisation and such fieldsfiles have extensions `.pa, .pb, ... , .pj` and include an automatically coded model date/time. To select this use `UM -> namelist -> Model Input and Output -> Model Output Streams`.

The STASHmaster file is centrally held and is unique for each UM version.

Altering your local STASHmaster copy is required to add new prognostics, diagnostics or ancillary fields. Usually adding new prognostics or diagnostics requires UM code changes, but a "user ancillary" system allows the user to add new fields as UM ancillary files without code changes and recompilation (provided these fields don't affect the model evolution itself). These "User Ancillaries: use reserved section 0 STASHcodes:
- 301–320 : For atmosphere single level ancillaries
- 321–340 : For atmosphere multi-level ancillaries

To initialise any **prognostic** fields for a run, use `UM -> namelist -> Reconfiguration and Ancillary Control -> Configure ancils and initialise dump fields`

STASH comes with pre-configured routined to take means at climate timescales. There are some restrictions:
- The lowest period mean (e.g. annual) must be a multiple of the model dumping period
- higher periods are multiples of the lower periods
- there are up to **four** periods available
- All climate-meaned diagnostics use a subset of the same periods
- the climate means system runs continuously throughout the whole run
- When using the Gregorian calendar, only monthly, seasonal and annual means are available.

A mean reference data can be supplied so that all mean periods are aligned with certain months (e.g. DJF for a seasonal mean).

Once climate-meaning is set up, STASH diagnostics have to be attached to a usage profile which sends them to the climate-means system.

## Troubleshooting

The complexity of the UM system allows many avenues for mistakes. A successful model runs tarts with the job definition using the UM GUI and input namelists. The job is launched by Rose and UM program scripts are executed. A standard pattern for a rose suite is:
1. Compile the model to generate a new executable (`fcm_make_task`)
2. Invoke the reconfiguration program (`recon task`)
3. Run the model (`atmos task`)
However a rose suite may only use one of these stages. 

All stages may work, but the model integration may fail due to numerical errors!

Each UM version has a set of release notes which contains lists of known problems and fubs.

First step to UM troubleshooting is to examine the relative output files using `rose suite-log` and `rose bush`.

When running in parallel note that an OUTPUT file is generated for each processor and held temporarily in separate files, optionally deleted at end of job. Only the file corresponding to PE0 is included in the job output listing, with title "PE0 OUTPUT". If there isn’t an error message in the PE0 output it is worth checking the output from other PEs as it is possible that only a subset of PEs encountered the error.

Runtime output will appear in the following order in job output files. The variables starting “%” can be used for searching but are also in the listing of SCRIPT if requested.
- Rose Script output:
- %PE0 : all output from PE0
    - "Atmosphere run-time constants" : details of science settings.
    - "READING UNIFIED MODEL DUMP" : details of the input starting dump
    - "Atm Step: Timestep 1" : initialisation completed, atmosphere time-stepping started
    - "END OF RUN - TIMER OUTPUT": timing information
Typical problems include:
1. User errors in model set-up.
• Incorrect filenames specified (including user-specific variables and typos) • Logic errors in user updates;
• Inappropriate parameter values chosen.
2. UM system errors in control. 
3. Science formulation errors.
4. Input data errors
5. OS/hardware problems
