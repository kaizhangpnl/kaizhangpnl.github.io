E3SM-SCM runscript template 
===========================

ARM95 case
----------

Below is an example for the ARM95 case:: 


   #!/bin/csh
   
   set echo 
   
   ##################################################################
   ##  Script to configure, build, and run E3SM SCM
   ## 
   ##  ARM95 case 
   ##  
   ##  Based on script provided by Peter Bogenschutz 
   ##################################################################
   
   
     set ACMEROOT=~/model/ACME
     set PROJECT=CMDV
   
     # Set the name of your case here
     setenv casename scm_ARM95
   
     # Set the case directory here
     setenv casedirectory ${ACMEROOT}/scmrun
    
     # Name of machine you are running on (i.e. edison, anvil, etc)                                                    
     setenv machine constance
   
     # Want to submit run to the queue?
     #   Setting to false will submit run directly 
     #   onto the login nodes rather than using
     #   the batch queue
     setenv submit_to_queue false
     
     # Aerosol specification
     # Options include:
     #  1) cons_droplet (sets cloud liquid and ice concentration
     #                   to a constant)
     #  2) prescribed (uses climatologically prescribed aerosol 
     #                 concentration)
     setenv init_aero_type cons_droplet ###prescribed 
   
     # User enter any needed modules to load or use below
     # for some ACME/E3SM versions, have to set the following: 
     module load python/2.7.8
     module load intel/15.0.1
     module load mvapich2/2.1
     module load netcdf/4.3.2
     module load mkl/15.0.1
     setenv MKL_PATH $MLIB_LIB
     setenv NETCDF_HOME /share/apps/netcdf/4.3.2/intel/15.0.1
   
     # Case specific information kept here
     ###change 
     set lat = 36.6 # latitude  
     set lon = 262.5 # longitude
     set do_iop_srf_prop = .true. # Use surface fluxes in IOP file?
     set do_scm_relaxation = .false. # Relax case to observations?
     set do_turnoff_swrad = .false. # Turn off SW calculation
     set do_turnoff_lwrad = .false. # Turn off LW calculation
     set micro_nccons_val = 100.0D6 # cons_droplet value for liquid
     set micro_nicons_val = 0.0001D6 # cons_droplet value for ice
     set startdate = 1995-07-18 # Start date in IOP file
     set start_in_sec = 19800 # start time in seconds in IOP file
     set stop_option = ndays 
     set stop_n = 17
     set iop_file = ARM95_iopfile_4scam.nc #IOP file name
   
     # Location of IOP file
     set iop_path = atm/cam/scam/iop
   
     # Prescribed aerosol file path and name
     set presc_aero_path = /pic/projects/climate/zhan524/share/
     set presc_aero_file = mam4_0.9x1.2_L72_2000clim_c170323.nc
   
     cd $ACMEROOT/cime/scripts
     set compset=F_SCAM5
     set grid=T42_T42
   
     set CASEID=$casename   
   
     set CASEDIR=${casedirectory}/$CASEID
     
     set run_root_dir = $CASEDIR
     set temp_case_scripts_dir = $run_root_dir/case_scripts   
   
     set case_scripts_dir = $run_root_dir/case_scripts
     set case_build_dir   = $run_root_dir/build
     set case_run_dir     = $run_root_dir/run 
   
     set walltime = '00:10:00'
   
     # COSP, set to false unless user really wants it
     setenv do_cosp  false
   
     # Create new case
     ./create_newcase -case $temp_case_scripts_dir -mach $machine -project $PROJECT -compset $compset -res $grid --walltime $walltime
     cd $temp_case_scripts_dir
   
     # SCM must run in serial mode
     ./xmlchange --id MPILIB --val mpi-serial
     
     # Define executable and run directories
     ./xmlchange --id EXEROOT --val "${case_build_dir}"
     ./xmlchange --id RUNDIR --val "${case_run_dir}" 
   
     # Set to debug, only on certain machines  
     if ($machine == edison) then 
       ./xmlchange --id JOB_QUEUE --val 'debug'
     endif
   
     if ($submit_to_queue == false) then
       ./xmlchange --id RUN_WITH_SUBMIT --val 'TRUE'
       ./xmlchange --id SAVE_TIMING --val 'FALSE'
     endif   
   
     # Get local input data directory path
     set input_data_dir = `./xmlquery DIN_LOC_ROOT -value`
   
     # need to use single thread
     set npes = 1
     foreach component ( ATM LND ICE OCN CPL GLC ROF WAV )
       ./xmlchange  NTASKS_$component=$npes,NTHRDS_$component=1
     end
   
     # CAM configure options.  By default set up with settings the same as ACMEv1
     set CAM_CONFIG_OPTS="-phys cam5 -scam -nospmd -nosmp -nlev 72 -clubb_sgs"
     if ( $do_cosp == true ) then
       set  CAM_CONFIG_OPTS="$CAM_CONFIG_OPTS -cosp -verbose" 
     endif
   
     # CAM configure options dependant on what aerosol specification is used
     if ($init_aero_type == cons_droplet) then 
       set CAM_CONFIG_OPTS="$CAM_CONFIG_OPTS -chem linoz_mam4_resus_mom_soag -rain_evap_to_coarse_aero -bc_dep_to_snow_updates" 
     endif
   
     if ($init_aero_type == prescribed || $init_aero_type == observed) then
       set CAM_CONFIG_OPTS="$CAM_CONFIG_OPTS -chem none"
     endif
   
     ./xmlchange CAM_CONFIG_OPTS="$CAM_CONFIG_OPTS" 
   
   # User enter CAM namelist options
   #  Add additional output here for example
   cat <<EOF >> user_nl_cam
    cld_macmic_num_steps = 8
    cosp_lite = .true.
    use_gw_front = .true.
    iopfile = '$input_data_dir/$iop_path/$iop_file'
    mfilt = 5000
    nhtfrq = 1
    scm_iop_srf_prop = $do_iop_srf_prop 
    scm_relaxation = $do_scm_relaxation
    iradlw = 1
    iradsw = 1
    swrad_off = $do_turnoff_swrad 
    lwrad_off = $do_turnoff_lwrad
    scmlat = $lat 
    scmlon = $lon
   EOF
   
   # CAM namelist options to match ACMEv1 settings
   #  Future implementations this block will not be needed
   #  Match settings in compset 2000_cam5_av1c-04p2
   cat <<EOF >> user_nl_cam
    use_hetfrz_classnuc = .true.
    micro_mg_dcs_tdep = .true.
    microp_aero_wsub_scheme = 1
    sscav_tuning = .true.
    convproc_do_aer = .true.
    demott_ice_nuc = .true.
    liqcf_fix = .true.
    regen_fix = .true.
    resus_fix = .false.
    mam_amicphys_optaa = 1
    fix_g1_err_ndrop = .true.
    ssalt_tuning = .true.
    use_rad_dt_cosz = .true.
    ice_sed_ai = 500.0
    cldfrc_dp1 = 0.045D0
    clubb_ice_deep = 16.e-6
    clubb_ice_sh = 50.e-6
    clubb_liq_deep = 8.e-6
    clubb_liq_sh = 10.e-6
    clubb_C2rt = 1.75D0
    zmconv_c0_lnd = 0.007
    zmconv_c0_ocn = 0.007
    zmconv_dmpdz = -0.7e-3
    zmconv_ke = 1.5E-6
    effgw_oro = 0.25
    seasalt_emis_scale = 0.85
    dust_emis_fact = 2.05D0
    clubb_gamma_coef = 0.32
    clubb_C8 = 4.3
    cldfrc2m_rhmaxi = 1.05D0
    clubb_c_K10 = 0.3 
    effgw_beres = 0.4
    do_tms = .false.
    so4_sz_thresh_icenuc = 0.075e-6
    n_so4_monolayers_pcage = 8.0D0
    micro_mg_accre_enhan_fac = 1.5D0
    zmconv_tiedke_add = 0.8D0
    zmconv_cape_cin = 1
    zmconv_mx_bot_lyr_adj = 2
    taubgnd = 2.5D-3
    clubb_C1 = 1.335
    raytau0 = 5.0D0
    prc_coef1 = 30500.0D0
    prc_exp = 3.19D0 
    prc_exp1 = -1.2D0
    se_ftype = 2
    clubb_C14 = 1.3D0
    relvar_fix = .true. 
    mg_prc_coeff_fix = .true.
    rrtmg_temp_fix = .true.
   EOF
   
   # if constant droplet was selected then modify name list to reflect this
   if ($init_aero_type == cons_droplet) then
   
   cat <<EOF >> user_nl_cam
     micro_do_nccons = .true.
     micro_do_nicons = .true.
     micro_nccons = $micro_nccons_val 
     micro_nicons = $micro_nicons_val
   EOF
   
   endif
   
   # if prescribed or observed aerosols set then need to put in settings for prescribed aerosol model
   if ($init_aero_type == prescribed ||$init_aero_type == observed) then
   
   cat <<EOF >> user_nl_cam
     use_hetfrz_classnuc = .false.
     aerodep_flx_type = 'CYCLICAL'
     aerodep_flx_datapath = '$presc_aero_path' 
     aerodep_flx_file = '$presc_aero_file'
     aerodep_flx_cycle_yr = 01
     prescribed_aero_type = 'CYCLICAL'
     prescribed_aero_datapath='$presc_aero_path'
     prescribed_aero_file='$presc_aero_file'
     prescribed_aero_cycle_yr = 01
   EOF
     
   endif
   
   # if observed aerosols then set flag
   if ($init_aero_type == observed) then
   
   cat <<EOF >> user_nl_cam
     scm_observed_aero = .true.
   EOF
   
   endif
   
   # avoid the monthly cice file from writing as this 
   #   appears to be currently broken for SCM
   cat <<EOF >> user_nl_cice
     histfreq='y','x','x','x','x'
   EOF
   
   # Use CLM4.5.  Currently need to point to the correct file for Eulerian 
   #  dy-core (this will be fixed in upcoming PR)
   set CLM_CONFIG_OPTS="-phys clm4_5"
   ./xmlchange CLM_CONFIG_OPTS="$CLM_CONFIG_OPTS"
   cat <<EOF >> user_nl_clm
     fsurdat = '$input_data_dir/lnd/clm2/surfdata_map/surfdata_64x128_simyr2000_c170111.nc'
   EOF
   
   # Modify the run start and duration parameters for the desired case
     ./xmlchange RUN_STARTDATE="$startdate",START_TOD="$start_in_sec",STOP_OPTION="$stop_option",STOP_N="$stop_n"
   
   # Modify the latitude and longitude for the particular case
     ./xmlchange PTS_MODE="TRUE",PTS_LAT="$lat",PTS_LON="$lon"
     ./xmlchange MASK_GRID="USGS"
   
     ./case.setup 
   
   # Don't want to write restarts as this appears to be broken for 
   #  CICE model in SCM.  For now set this to a high value to avoid
     ./xmlchange PIO_TYPENAME="netcdf"
     ./xmlchange REST_N=30000
   
   # Modify some parameters for CICE to make it SCM compatible 
     ./xmlchange CICE_AUTO_DECOMP="FALSE"
     ./xmlchange CICE_DECOMPTYPE="blkrobin"
     ./xmlchange --id CICE_BLCKX --val 1
     ./xmlchange --id CICE_BLCKY --val 1
     ./xmlchange --id CICE_MXBLCKS --val 1
     ./xmlchange CICE_CONFIG_OPTS="-nodecomp -maxblocks 1 -nx 1 -ny 1"
   
   # Build the case 
     ./case.build
   
   # Submit case to queue if set, else submit
   #   via the case.run script
     if ($submit_to_queue == true) then 
       ./case.submit
     else 
       ./case.submit --no-batch
     endif
     
     exit



