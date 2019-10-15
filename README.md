# TAM\_ORCA2
## Overview
This NEMO configuration incorporates all bug fixes, modifications and feature additions to NEMOTAM used in my work. 


NEMOTAM is a tangent-linear and adjoint model (TAM) counterpart to the nonlinear NEMO primitive equation solving ocean circulation model (c.f. *Vidard et al.*, 2011). As such, it details the evolution of small perturbations to a known background state (the "trajectory") obtained by running the nonlinear model (tangent-linear mode). In adjoint mode, "cost functions" of the ocean state are run backwards along the trajectory to determine their sensitivity to earlier perturbations.


In addition to the fixing bugs in the default NEMOTAM source code, the configuration adds:

- Ability to initialise the tangent-linear (/adjoint) model with a netCDF file containing an ocean-state perturbation (/cost function)
- forward and reverse passive-tracer tracking, with optional surface ventilation
- Long-format output file names allowing the model to be run for up to 100 million time-steps.
- Ability to apply time-averaged cost-functions
- trajectory "offsetting" allowing the tangent-linear to begin at any point in an existing trajectory, or for longer trajectories to be built using model restarts
- New advection schemes (upwind, TVD, weighted-mean scheme and manually-weighted scheme)
- Ability to apply scalar-valued adjoint cost functions to the nonlinear forward model and produce an ASCII time series
- Option to include eddy-induced velocities (EIV) in the TAM trajectory velocity fields
- Ability to apply perturbations to the ocean state of the non-linear model (asminc)

## Installation
### Installing NEMO and PT_TAM configuration
This repository requires existing NEMO v3.4 (and NEMOTAM) installs. These can be installed using:

`svn co https://forge.ipsl.jussieu.fr/nemo/svn/NEMO/releases/release-3.4`

This provides the source code for the model and reference configurations, on which the configuration should be based.

For a first-time NEMO install, the user will need to set up their machine's architecture. This will either already exist somewhere inside `NEMOGCM/ARCH` (e.g. `ARCH/<DIR>/arch-<ARCHITECTURE>.fcm`) or will have to be created. Instructions for creating/modifying this file are found [here](http://forge.ipsl.jussieu.fr/nemo/wiki/Users/ModelInstall#Setupyourarchitectureconfigurationfile).

At this point, the configuration can be created. To do this, run the following command in the `CONFIG` directory:

`./makenemo -d "OPATAM_SRC LIM_SRC_2 OPA_SRC" -n TAM_ORCA2_MASTER -m <ARCHITECTURE> add_key "key_netcdf4 key_mpp_mpi key_mpp_rep key_nosignedzero key_tam key_diainstant" del_key "key_iomput key_zdfddm"`

This creates a new configuration with ocean (`OPA_SRC`) and sea-ice (`LIM_SRC_2`) dynamics, and TAM (`OPATAM_SRC`, `key_tam`) compatibility. 
(NOTE: customise line as required, e.g. regarding parallel processing - `key_mpp_mpi` and `key_mpp_rep`. Replace <ARCHITECTURE> with the relevant substring from the arch. filename)
 
This configuration can now be modified. To do so, move the contents of `MY_SRC` from this repository into the `NEMOGCM/CONFIG/ORCA2_TAM/MY_SRC` directory and recompile using

`./makenemo -n ORCA2_TAM add_key "key_asminc"`

### Obtaining and linking forcing and other model input files
In order to run NEMO-ORCA2, additional files are required, which can be found [here](https://doi.org/10.5281/zenodo.1471702).

Unpack this archive into a directory named `ORCA2_INPUT`. This will be at the same level as your experiment directory and can be anywhere you choose.

### Creating an experiment directory
At the same level as `ORCA2_INPUT`, create an experiment directory (e.g. `RUN_DIR`). Copy the contents of our template experiment directory (`RUN_DIR`) into this directory. The bash script `link_runfiles.sh` creates softlinks to all files necessary to run NEMO and NEMOTAM. Edit it to provide the location of your `ORCA2_INPUT` and `BLD/bin`.

### Directory structure

You should now have a NEMO configuration with the following structure

- `/home/username/NEMO/dev_v3_4_STABLE_2012/NEMOGCM/CONFIG/`
  - `PT_TAM/`
    - `BLD/`
    - `MY_SRC/`
    - `WORK/`
    - `EXP00/`
    
and an experiment area with the structure    

- `/<PATH TO EXPERIMENT AREA>/`
  - `ORCA2_INPUT`
  - `RUN_DIR`

## Running NEMOTAM


### Spinning up
To set up, the model should first be spun up from rest (ideally for around a thousand years) to provide a start point for all experiments.
Inside `RUN_DIR/SPINUP_RUNS` is a bash script `make_spinup_namelists.sh`, which produces namelist files for consecutive runs of 50 years (273750 model time steps) each. They can be tuned by editing this script along with `namelist.SPINUP_template`.

The end result should be the files `SPINUP_????????_restart.nc` and `SPINUP_????????_restart_ice.nc`, used as start points for the trajectory.

### Running a trajectory
After spinning up, the non-linear model should be run for the desired length of the experiment. A script is provided (`RUN_DIR/TRAJECTORY_RUNS/make_trajectory_namelists.sh`) to produce ocean and ice namelists for consecutive 50 year trajectories adding up to 400 years. The namelist parameters themselves can be adjusted in `RUN_DIR/TRAJECTORY_RUNS/namelist.TRAJ_TEMPLATE` and `RUN_DIR/TRAJECTORY_RUNS/namelist_ice.TRAJ_TEMPLATE`.

namelist section **namrun**

Be sure to set `ln_rstart = .true.`, and set `nn_it000`, `nn_date0` and `cn_ocerst_in` in accordance with the `SPINUP_*.nc` files generated above.

namelist section **namtrj**:

Be sure TAM trajectory output is produced with `ln_trjhand = .true.`. The trajectory output location and filename prefix is set by `cn_dirtrj`. All other output files are not required by `NEMOTAM`. 

**Other parameters**:
Other options are detailled in [the NEMO 3.4 manual](http://forge.ipsl.jussieu.fr/little_nemo/export/44/vendor/nemo/current/DOC/NEMO_book.pdf).

### Running the TAM:
**Initial perturbation/cost function**:

NEMOTAM should be initialised using `31x149x182` arrays stored in a netCDF file, which the namelist will point to (`cn_tam_input`).
 Variable names are:

- `t0_tl` - tangent-linear temperature perturbation 
- `v0_tl` - tangent-linear meridional velocity perturbation
- `t0_ad` - adjoint cost function applied to temperature
- `v0_ad` - adjoint cost function applied to meridional velocity

**At the beginning of the namelist file**:

- namelist section **namrun**
  - Ensure `nn_it000 = 1` (even if beginning later in the trajectory). 
  - For the tangent-linear model, `nn_itend` should be set to the desired number of time steps in the run (up to the length of the trajectory). In the adjoint model, `nn_itend` determines the "start point" relative to the end of the trajectory, from which model will run backwards.

**At the end of the namelist file:**

- namelist section **namtst_tam**
  - `cn_dirtrj` is the location of the trajectory output. It should match that set in the trajectory namelist.
  - `nn_ittrjoffset` if set to "N", this treats trajectory step "N" as the first step in the trajectory
- namelist section **namtst\_tam**
  - `ln_swi_opatam` determines TAM mode:
    - 0,1: testing modes
    - 2: tangent-linear mode
    - 3: adjoint mode
    - 4: applies the adjoint cost function to a nonlinear model run, outputs a scalar value in the ASCII file `cost_fn_background.dlm` at every time step
    - 5: runs the tangent-linear model in passive mode
    - 6: runs the adjoint model in passive mode
  - `cn_tam_input` is the filename of the netCDF file which initialises the tangent-linear or adjoint model
  - `ln_tl_eiv` switch to include EIV in the trajectory seen by NEMOTAM or not
- namelist section **namtl_trj** 
  - `ln_trjwri_tan`: switch to write NEMOTAM output or not
  - `nn_ittrjfrq_tan`: output frequency 
  - `cn_tantrj`: location and filename format of output
- namelist section **nampttam**
  - `ln_pt_region` : switch to only read and write output on a subset of the global grid
  - `rn_NEptlat`   : specify the northeast and southwest corners (lat and lon) of the subset of interest
  - `rn_SWptlat`
  - `rn_NEptlon`
  - `rn_SWptlon` 

- namelist section **namtra\_adv\_tam** (Options for passive tracer advection scheme)
  - `ln_traadv_cen2` 2nd order centred
  - `ln_traadv_tvd` total variance diminishing (**NOTE: NONLINEAR**. This scheme is forward in the tangent-linear and reversed in the adjoint, but is recommended for passive-tracer propagation only.)
  - `rn_traadv_weight_h` weighting between upstream and centred scheme (lateral advection, 1 is upstream, 0 is centred, -1 invokes automatic calculation based on the weighted-mean scheme)
  - `rn_traadv_weight_v` weighting parameter between upstream and centred scheme (vertical advection)


### TAM output
If run in parallel, the model delegates parts of the grid to different CPUs. The output is thus returned in "tiles" as `PTTAM_output_<TIMESTEP>_????.nc`, which can be pieced together using `TOOLS/rebuild_nemo PTTAM_<TIMESTEP>_output <NO. OF TILES>`. Where the `TOOLS` directory is found at the same level as `CONFIG` (`../../TOOLS` from the install location of this repository). Each file contains the following variables:

- `nav_lon` and `nav_lat` (2D longitude and latitude arrays for the grid)
- `nav_lev` 1D depth array for the grid
- `time_counter` (in seconds)
- `un`, `vn`,`wn` snapshot of trajectory zonal, meridional and vertical velocity 
- `tn`, `sn` snapshot of trajectory temperature and salinity
- `sshn` snapshot of trajectory sea-surface height

As well as, in the tangent-linear:

- `t_tl`  snapshot of perturbed ocean-state temperature
- `s_tl` snapshot of perturbed ocean-state salinity 
- `u_tl` snapshot of perturbed ocean-state zonal velocity
- `v_tl` snapshot of perturbed ocean-state meridional velocity
- `wn_tl` snapshot of perturbed ocean-state vertical velocity
- `sshn_tl` snapshot of perturbed ocean-state sea-surface height

and in the adjoint:

- `t_ad` snapshot of cost function sensitivity to temperature 
- `s_ad` snapshot of cost function sensitivity to salinity 
- `u_ad` snapshot of cost function sensitivity to zonal velocity 
- `v_ad` snapshot of cost function sensitivity to meridional velocity
- `wn_ad` snapshot of cost function sensitivity to  vertical velocity 
- `sshn_ad` snapshot of cost function sensitivity to sea-surface height


### Running the TAM in passive mode to track a passive tracer
**Input**

The model should be run as above, with `ln_swi_opatam` in the namelist section `namtst_tam` set to either 5 (forward tracking) or 6 (backward tracking). Additionally, the input file variable name should be `pt0_tl` (forward) or `pt0_ad` (backward). 

**Surface ventilation**

In the absence of surface ventilation, tracer should be perfectly conserved, never leaving the system. To remove tracer at the surface, and produce a record of this removal:

- namelist section **namsbc\_ssr**
  - The namelist parameter `nn_sstr` determines whether the passive tracer is removed at the surface
  - The namelist parameter `rn_dqdt` sets the time scale of the surface restoring scheme. The default, -40Wm^-2K^-1, corresponds to 60 days over a 50 m mixed layer.

**Output**

In passive mode, the output filename structure is "PTTAM\_output\_<timestep>.nc". The variables are:

- `pt_conc_tl` (tangent linear) or `pt_vol_ad` (adjoint): a 4D (x,y,z,t) array describing the spatiotemporal distribution of passive tracer concentrations (tangent-linear) or volumes (adjoint).


**NOTE**: the tracer _concentration_ distribution is provided by the tangent-linear for mathematical consistency. The volume can be calculated by producing a `mesh_mask` file (`nn_msh = 1` in the namelist file) and multiplying the concentrations by `e1t`,`e2t` and `e3t` (the grid dimensions). The adjoint outputs volume automatically.

- `pt_vent_tl` (tangent-linear) or `pt_vent_ad` (adjoint) a 3D (x,y,t) array describing the _cumulative_ removal of tracer at the surface during the run (needs to be differenced to produce a time series).

## Modifications to model defaults

Each feature or model change has its own ID. In the source code, a modification associated with this change begins with
`!!!<ID>` and ends with `!!! /<ID>`. This allows modifications to be found, tweaked and removed without damaging other functionality. 
All changes and their purpose are listed here and were either added by S. Mueller or myself. Asterisks indicate modifications which are essential to run NEMOTAM.

- 20191004A : modifications permitting perturbations to the nonlinear model using "asminc"
- 20191004B : single TAM output variables (e.g. tn+tb) and specification of TAM output variables
- 20191004C : Add "sshn" to nonlinear trajectory
- 20191004D: Expanding trajectory and TAM filenames to allow at least 8-digit time-steps
- 20191004E*: correct transition of b->n for kdir==1 [see here](http://forge.ipsl.jussieu.fr/nemo/attachment/ticket/1443/trj_tam.F90.diff)
- 20191004F: corrected 'stpr2 - zstp -1' to 'stpr2 - zstp +1' in trj_tam.F90
- 20191004G: adjust output of interpolation coefficients 
- 20191004H: switch to allow adjoint output writing 
- 20191004I: essential modifications to dynzdf_imp_tam.F90 [see here](http://forge.ipsl.jussieu.fr/nemo/attachment/ticket/1362/dynzdf_imp_tam.F90.diff)
- 20191004J*: addition of adjoint time-stepping loop 
- 20191004K*: proper initialisation of TAM variables 
- 201910004L: ability to read cost function/perturbation from netCDF file
- 20191004M: use of cost function averaging window 
- 20191004N*: building of {t,u,v,f}msk_i variables using dom_uniq 
- 20191004O*: force flux SBC (http://forge.ipsl.jussieu.fr/nemo/attachment/ticket/1738/sbcmod_tam.F90.diff)[20160524]
- 20191004P: Passive tracer module + subroutines
- 20191004Q: Modifications to output ventilation record of passive tracer
- 20191004R: trajectory offsetting
- 20191004S: Introduction of weighted-mean avection scheme 
- 20191004T: Introduction of trajectory-upstream scheme and manual weightings
- 20191004U: Introduction of TVD scheme to tangent-linear and reversed fields to adjoint
- 20191004V: Option to output scalar time series of CF applied to trajectory
- 20191004W: Option to include EIV
- 20191013A: Switch and parameters to only write and read trajectory in a lat/lon defined region. If a CPU has no points in this region, it doesn't write trajectory tiles, except on the first time-step of the run. When reading, this CPU simply reads the first time step repeatedly. Any passive tracer concentration outside of this region is set to 0 to prevent instabilities. Dramatically reduces storage space required to run the trajectory for localised passive tracer studies.

## References
Fiadeiro, M.E. and Veronis, G., 1977. On weighted-mean schemes for the finite-difference approximation to the advection-diffusion equation. Tellus, 29(6), pp.512-522.

Vidard, A., Vigilant, F., Benshila, R. and Deltel, C., 2011. NEMO Tangent & Adjoint Models (NemoTam) Reference Manual & User's Guide (Doctoral dissertation, INRIA).

## Acknowledgements
The features of this configuration would be nonexistent without the work of S. Mueller, who is responsible for much of the core functionality here. My own additions would not have been possible without his inordinate patience in teaching me how to write them.
