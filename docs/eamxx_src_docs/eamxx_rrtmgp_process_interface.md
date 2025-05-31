# eamxx_rrtmgp_process_interface.cpp

## Overview

This file implements the EAMxx interface to the RRTMGP (Rapid Radiative Transfer Model for General Circulation Models) radiation parameterization. It is responsible for setting up required inputs, calling the RRTMGP library to compute radiative fluxes and heating rates, and providing these outputs back to the EAMxx framework. It handles both shortwave and longwave radiation calculations.

## Key Components

### Classes

*   **`RRTMGPRadiation`:** Inherits from `AtmosphereProcess` and manages the RRTMGP radiation calculations.
    *   **`RRTMGPRadiation(const ekat::Comm& comm, const ekat::ParameterList& params)`:** Constructor. Initializes the RRTMGP interface with communicator and parameters, including active gas names.
    *   **`set_grids(const std::shared_ptr<const GridsManager> grids_manager)`:** Sets up the grid information and declares required/computed fields for RRTMGP. This includes atmospheric state variables, surface properties, cloud properties, aerosol optics, and gas concentrations. It also defines output radiation fields (fluxes, heating rates).
    *   **`requested_buffer_size_in_bytes() const`:** Returns the size of the memory buffer required by the RRTMGP interface for its calculations, accounting for column chunking.
    *   **`init_buffers(const ATMBufferManager& buffer_manager)`:** Initializes internal YAKL/Kokkos buffer views using memory provided by the `ATMBufferManager`. These buffers are used to pass data to/from the RRTMGP Fortran kernels.
    *   **`initialize_impl(const RunType run_type)`:** Performs one-time initializations for the RRTMGP process. This includes setting radiation timestep frequency, orbital parameters, fixed solar irradiance/zenith angle (if any), prescribed gas concentrations, initializing YAKL, and loading RRTMGP coefficients and cloud optics data from files.
    *   **`run_impl(const double dt)`:** Executes the RRTMGP radiation calculation for a given timestep `dt`. This involves:
        *   Determining if radiation needs to be updated based on frequency.
        *   Calculating orbital parameters (solar declination, eccentricity factor).
        *   Preparing gas concentrations (converting MMR to VMR for H2O, using prescribed or computed values for others).
        *   Looping through column chunks:
            *   Copying input field data for the current chunk into YAKL/Kokkos buffers.
            *   Calculating layer thickness (`dz`) and interface temperatures (`T_int`).
            *   Populating the `GasConcs` object for RRTMGP.
            *   Setting cloud fraction based on subcolumn sampling options.
            *   Converting cloud mixing ratios to cloud mass per unit area (LWP, IWP).
            *   Computing band-by-band surface albedos.
            *   Calling the `rrtmgp_main` driver (Fortran kernel) to perform SW and LW calculations.
            *   Computing heating rates from flux divergence.
            *   Calculating diagnostic surface fluxes and cloud area.
            *   Copying output flux and heating rate data from YAKL/Kokkos buffers back to EAMxx fields.
        *   Applying the computed temperature tendency to the atmospheric state.
        *   Setting boundary fluxes for energy/mass conservation checks if enabled.
    *   **`finalize_impl()`:** Finalizes the RRTMGP process, including YAKL finalization.

### Functions

*   **`ConvertToRrtmgpSubview` (struct):** A helper struct with methods to create subviews of Kokkos views for RRTMGP, handling potential layout differences (LayoutLeft for YAKL vs. LayoutRight for EAMxx fields) and trimming excess elements due to SIMD packing.

## Important Variables/Constants

*   **`m_gas_names` (std::vector<std::string>):** Stores the names of active greenhouse gases considered in radiation calculations.
*   **`m_grid` (std::shared_ptr<const AbstractGrid>):** Pointer to the "Physics" grid.
*   **`m_ncol`, `m_nlay` (int):** Number of local columns and vertical layers on the physics grid.
*   **`m_nswbands`, `m_nlwbands` (int):** Number of shortwave and longwave bands (fixed for RRTMGP).
*   **`m_nswgpts`, `m_nlwgpts` (int):** Number of shortwave and longwave g-points used in RRTMGP.
*   **`m_col_chunk_size`, `m_num_col_chunks` (int):** Parameters for processing columns in chunks to manage memory.
*   **`m_buffer` (struct Buffer):** Contains YAKL/Kokkos views for all temporary arrays needed by RRTMGP kernels for a single column chunk.
*   **`m_gas_concs` (GasConcs):** YAKL object holding gas concentrations for RRTMGP.
*   **`m_gas_concs_k` (GasConcsKokkos):** Kokkos object holding gas concentrations for RRTMGP.
*   **`m_rad_freq_in_steps` (Int):** Frequency (in atmospheric timesteps) at which radiation is calculated.
*   **`m_orbital_year`, `m_orbital_eccen`, `m_orbital_obliq`, `m_orbital_mvelp`:** Parameters for calculating Earth's orbital position and its effect on solar insolation.
*   **`m_fixed_total_solar_irradiance`, `m_fixed_solar_zenith_angle` (double):** Options for idealized experiments.
*   **`m_do_aerosol_rad` (bool):** Flag to enable/disable aerosol radiative effects.
*   **`m_do_subcol_sampling` (bool):** Flag to enable/disable MCICA subcolumn sampling for cloud-radiation interaction.
*   **`coefficients_file_sw`, `coefficients_file_lw` (std::string):** Paths to RRTMGP gas optics coefficient files.
*   **`cloud_optics_file_sw`, `cloud_optics_file_lw` (std::string):** Paths to RRTMGP cloud optics data files.

## Usage Examples

*(This class is an `AtmosphereProcess` and is primarily used internally by the `AtmosphereDriver`. Developers working on radiation aspects might modify its parameter handling, input/output fields, or the logic within `run_impl` if, for example, new aerosol types or gas interactions are introduced.)*

## Dependencies and Interactions

*   **Internal Dependencies:**
    *   `physics/rrtmgp/scream_rrtmgp_interface.hpp`: Provides the C++ to Fortran bridge for RRTMGP routines.
    *   `physics/rrtmgp/rrtmgp_utils.hpp`: Utility functions for RRTMGP.
    *   `physics/rrtmgp/shr_orb_mod_c2f.hpp`: C++ interface to Fortran orbital mechanics routines.
    *   `physics/share/scream_trcmix.hpp`: For calculating trace gas mixing ratios.
    *   `share/io/scream_scorpio_interface.hpp`: For reading coefficient files.
    *   `share/util/scream_common_physics_functions.hpp`: For common physics calculations (e.g., VMR from MMR).
    *   `share/util/scream_column_ops.hpp`: For column-wise operations (e.g., interface temperature calculation).
    *   `ekat/ekat_assert.hpp`, `ekat/ekat_parameter_list.hpp`: For assertions and parameter handling.
*   **External Libraries:**
    *   **RRTMGP (Fortran library):** Core dependency for all radiative transfer calculations. This interface calls Fortran subroutines like `rrtmgp_initialize`, `rrtmgp_main`, `rrtmgp_finalize`.
    *   **YAKL/Kokkos:** Used for managing data arrays passed to/from the RRTMGP Fortran kernels, enabling performance portability on CPUs and GPUs.
*   **Interactions with other Components:**
    *   **`AtmosphereDriver`:** Manages the lifecycle of `RRTMGPRadiation` (initialization, run calls, finalization).
    *   **FieldManager:** `RRTMGPRadiation` registers its required input fields (temperature, pressure, water vapor, cloud properties, surface albedo, aerosol optics, gas concentrations) and computed output fields (fluxes, heating rates) with the FieldManager.
    *   **GridsManager:** Obtains the "Physics" grid information.
    *   **SPA (Simplified Plume Anlaysis):** If `m_do_aerosol_rad` is true, RRTMGP depends on aerosol optical properties (AOD, SSA, g) provided by an aerosol component like SPA.
    *   **Chemistry/Tracer Model:** May provide ozone (`o3_volume_mix_ratio`) concentrations. Other trace gases can be prescribed or calculated by `scream_trcmix`.
    *   **Cloud Microphysics/Macrophysics:** Provides cloud condensate amounts (qc, qi), effective radii (rel, rei), and cloud fraction (cldfrac_tot) which are crucial inputs for cloud-radiation interactions.
