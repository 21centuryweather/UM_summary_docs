# Introduction

The ACCESS modeling suite is reliant on the UK Met Office's **Unified Model** (UM) for atmospheric simulations.

This documentation summarises the UM Documentation Papers that are available (behind the Met Office Science Repository Service Paywall) at https://code.metoffice.gov.uk/doc/um/latest/umdp.html

It assumes the reader has some familiarity with:
- The equations of motion that underpin dynamical meteorology 
- The concepts that underpin numerical solutions of these equations (e.g. domain discretization, grid resolution, boundary conditions)
- The basics of the Linux operating system and how to use a command-line-interface (CLI).
- How resources for computational tasks are allocated on the ANU supercomputer (Gadi) according to projects, and what the PBS job scheduler is.

These documents focus on sections most relevant to staff in the 21stCenturyWeather Centre-of-Excellence:
- A quick review of the equations used to model the atmosphere
- Executables required to run a high-resolution regional UM simulation within a `rose/cylc` suite including:
    - `fcm_make`
    - `Reconfingure`
    - `CreateBC`
- Details of domain decomposition, halos and lateral boundary conditions (LBCs)
- Editing STASH tables to analyse UM outputs
