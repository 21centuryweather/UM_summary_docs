# Overview

The Unified Model is a suite of Fortran-based executables built to simulate the future state of the atmosphere. It is designed to interface with land/surface and sea-surface data.

The model is so named (i.e. "Unified") because it contains a single set of code that can be applied across a diverse range of length and timescales; form urban weather forecasts at grid resolutions of a few hundred metres, to global climate model projections extending into next century at grid resolutions of hundreds of kilometres.

It's modular nature allows its use across a vast array of applications, e.g.
- detailed atmospheric chemistry and cloud experiments
- cean-atmosphere experiments
- coupled fire-weather experiements
- operational Numerical Weather Prediction forecasts (NWP)

A brief overview of the UM is available here : https://www.metoffice.gov.uk/research/approach/modelling-systems/unified-model

All this power and complexity does come at the cost of usability. The UM's code extends back to 1990 and its configuration and settings are controlled using a dizzying array of plain-text input files called `namelists`.

Thankfully in recent years, the UK Met Office, in conjunction with some of its overseas science partners, has created various graphics-based applications which sit on top of the `rose/cylc` scheduling system which shield the use from much of the underlying complexity. Simple UM experiments can now be configured and run using a few clicks of the `rose edit` or `cylc gui`.

Other aspects about the UM:
- All modern UM version use the ENDGame dynamical core to solve the equations of motion - https://www.metoffice.gov.uk/research/news/2014/endgame-a-new-dynamical-core
- Uses Arakwa 'C'-type grid in horizontal planes, with data ordered south to north. 
- Uses a Charney-Phillips grid in the vertical which is terrain-following near the surface but evolves to constant height surfaces higher up. **This is important to remember as its the reason why some level data is NaN (Not-a-Number) along 850 hPa pressure-levels near high mountains (e.g. Papua New Guinea)
- Reconfiguration is required for limited area models (i.e. non-spherical grids) which are usually "nested" inside a lower-resolution global model. Reconfiguration is a fancy way to describe what is essentially a regridding process; taking information from an lower resolution global grid provide boundary conditions for a higher resolution regional grid. Sometimes you may use Reconfiguration to take information from one regional configuration to provide information for an even  higher resolution configuration. This is known as "double-nesting."
- The UM requires a lot of external information to run a simulation:
    - Orography
    - soil moisture
    - locations of rivers and lakes
    - vegetation and urban geography
    - land/sea masks
    - Snow cover
    - Sea-ice
    - Sea-Surface-Temperatures (SSTs) 
    - etc. 
This information is provided using *ancillariy files*. The data may be constant or incorporated as the simulation proceeds to allow temporal variation of surface vegetation, snow cover, wind stress etc. Much of the difficulty in configuring regional models is dealing with the ancilliary data at the correct resolution, e.g. matching the orography data between the outer resolution to the inner, higher resolution required by the regional configuration.
- For fully coupled simulations (e.g. where the SSTs are computed using a seperate ocean model) the UM uses the OASIS3-MCT (Model Coupling Toolkit) coupler.