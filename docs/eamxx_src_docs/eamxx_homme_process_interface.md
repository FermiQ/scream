# eamxx_homme_process_interface.cpp

## Overview

This file implements the EAMxx interface to the HOMME (High-Order Methods Modeling Environment) dynamical core. It is responsible for managing the grids, state variables (like velocity, temperature, pressure, tracers), and forcing terms related to atmospheric dynamics. It orchestrates the execution of the HOMME model for advancing the atmospheric state in time and handles the necessary data transformations (remapping) between the physics grid and the dynamics (spectral element) grid used by HOMME.

## Key Components

### Classes

*   **`HommeDynamics`:** Inherits from `AtmosphereProcess` and manages the HOMME dynamical core execution.
    *   **`HommeDynamics(const ekat::Comm& comm, const ekat::ParameterList& params)`:** Constructor. Initializes the HOMME interface, registers as a `HommeContextUser`, and sets up basic parameters.
    *   **`~HommeDynamics()`:** Destructor. Unregisters as a `HommeContextUser`.
    *   **`set_grids(const std::shared_ptr<const GridsManager> grids_manager)`:** Sets up grid information (dynamics, physics, physicsGLL), initializes HOMME data structures if not already done, and declares required/computed/internal fields for dynamics. This includes prognostic state variables on the dynamics grid (e.g., `v_dyn`, `vtheta_dp_dyn`, `dp3d_dyn`, `Qdp_dyn`) and fields on the physics grid that are updated by or are inputs to dynamics (e.g., `horiz_winds`, `T_mid`, `ps`, `qv`). It also sets up remappers (`m_p2d_remapper`, `m_d2p_remapper`, `m_ic_remapper`) for transferring data between grids.
    *   **`requested_buffer_size_in_bytes() const`:** Returns the size of the memory buffer required by HOMME's internal functors.
    *   **`init_buffers(const ATMBufferManager& buffer_manager)`:** Initializes HOMME's internal functor buffers using memory provided by the `ATMBufferManager`.
    *   **`initialize_impl(const RunType run_type)`:** Performs one-time initializations for the HOMME dynamics process. This includes:
        *   Aliasing tracer group names.
        *   Completing HOMME's `prim_init1_xyz` sequence.
        *   Creating helper fields for forcings (`FQ_dyn`, `FT_dyn`, `FM_dyn`) and intermediate states (`Q_dyn`).
        *   Setting up and finalizing remappers.
        *   Initializing HOMME views to point to EAMxx field data.
        *   Calling `initialize_homme_state()` or `restart_homme_state()` based on `run_type`.
        *   Completing HOMME model initialization (`prim_init_model_f90`).
        *   Setting up field property checks (e.g., bounds for tracers, temperature).
        *   Initializing Rayleigh friction variables.
    *   **`run_impl(const double dt)`:** Executes the HOMME dynamical core for a given atmospheric timestep `dt`. This involves:
        *   Calculating `nsplit` (number of HOMME sub-steps).
        *   Calling `homme_pre_process(dt)` to compute tendencies and remap them to the dynamics grid.
        *   Looping `nsplit` times, calling `prim_run_f90()` for each HOMME sub-step.
        *   Updating HOMME's internal step counter.
        *   Calling `homme_post_process(dt)` to remap updated states back to the physics grid and perform necessary conversions (e.g., virtual potential temperature to temperature).
    *   **`finalize_impl()`:** Finalizes the HOMME process by calling `prim_finalize_f90()` and cleaning up the HOMME context.
    *   **`set_computed_group_impl(const FieldGroup& group)`:** Special handling for the "tracers" group to set `qsize` in HOMME.
    *   **`homme_pre_process(const double dt)`:** Computes temperature and momentum tendencies on the physics grid, remaps them (along with tracers) to the dynamics grid, and prepares HOMME's forcing terms.
    *   **`homme_post_process(const double dt)`:** Remaps updated HOMME state variables (velocity, temperature, pressure, tracers) from the dynamics grid back to the physics grid. Converts virtual potential temperature to temperature and updates pressure fields. Stores previous temperature and winds for next step's tendency calculation.
    *   **`create_helper_field(...)`:** Utility to create and allocate internal helper `Field` objects.
    *   **`init_homme_views()`:** Sets HOMME's internal Fortran data pointers (e.g., `state%v`, `tracers%Qdp`) to point to the memory allocated for EAMxx `Field` objects.
    *   **`restart_homme_state()`:** Handles the logic for restarting HOMME from previously saved state variables on the dynamics grid.
    *   **`initialize_homme_state()`:** Initializes HOMME state variables for an initial run, typically by remapping initial conditions from the physics GLL grid.
    *   **`copy_dyn_states_to_all_timelevels()`:** Copies the current HOMME state (n0 time level) to other time levels (nm1, np1) as required by HOMME's time stepping algorithms.
    *   **`update_pressure(const std::shared_ptr<const AbstractGrid>& grid)`:** Computes hydrostatic pressure on interfaces and midpoints for both wet and dry air.
    *   **`rayleigh_friction_init()` & `rayleigh_friction_apply(const double dt)`:** Initialize and apply Rayleigh friction for damping waves near the model top.

## Important Variables/Constants

*   **`m_dyn_grid`, `m_phys_grid`, `m_cgll_grid` (std::shared_ptr<const AbstractGrid>):** Pointers to the dynamics, physics (output), and CGLL (initial condition input) grids.
*   **`m_p2d_remapper`, `m_d2p_remapper`, `m_ic_remapper` (std::shared_ptr<AbstractRemapper>):** Remappers for transferring data between physics and dynamics grids, and for initial conditions.
*   **`m_helper_fields` (std::map<std::string, Field>):** A map storing internal `Field` objects used by the HOMME interface, such as prognostic variables on the dynamics grid (e.g., `v_dyn`, `vtheta_dp_dyn`, `dp3d_dyn`, `Qdp_dyn`) and forcing terms.
*   **HOMMEXX_NP, HOMMEXX_NUM_TIME_LEVELS, HOMMEXX_Q_NUM_TIME_LEVELS, HOMMEXX_PACK_SIZE, HOMMEXX_QSIZE_D, HOMMEXX_NUM_LEV, HOMMEXX_NUM_LEV_P:** Compile-time constants imported from HOMME defining grid and packing parameters.
*   **`m_bfb_hash_nstep` (int):** Frequency for printing Bit-for-Bit hash of HOMME state for debugging.

## Usage Examples

*(This class is an `AtmosphereProcess` and is primarily used internally by the `AtmosphereDriver`. Developers working on dynamical core aspects might modify its parameter handling, field definitions, or the logic within `run_impl`, `homme_pre_process`, or `homme_post_process` if, for example, new physics-dynamics coupling strategies are explored or HOMME's internal algorithms change.)*

## Dependencies and Interactions

*   **Internal Dependencies:**
    *   `dynamics/homme/physics_dynamics_remapper.hpp`: For remapping fields between grids.
    *   `dynamics/homme/homme_dimensions.hpp`: Provides HOMME-specific dimension constants.
    *   `dynamics/homme/homme_dynamics_helpers.hpp`: Helper functions for HOMME.
    *   `dynamics/homme/interface/scream_homme_interface.hpp`: C++ interface to HOMME's Fortran routines (e.g., `prim_init_data_structures_f90`, `prim_run_f90`, `prim_finalize_f90`).
    *   `physics/share/physics_constants.hpp`: For physical constants.
    *   `share/util/scream_common_physics_functions.hpp`: For common physics calculations (e.g., temperature conversions).
    *   `share/util/scream_column_ops.hpp`: For column-wise operations.
    *   `ekat/ekat_assert.hpp`, `ekat/ekat_parameter_list.hpp`, `ekat/kokkos/ekat_subview_utils.hpp`, `ekat/ekat_pack.hpp`: For assertions, parameter handling, Kokkos subviews, and data packing.
*   **External Libraries:**
    *   **HOMME (Fortran library):** The core dynamical core being interfaced. This includes its context (`Homme::Context`), simulation parameters (`Homme::SimulationParams`), state variables (`Homme::ElementsState`, `Homme::Tracers`), forcing structures (`Homme::ElementsForcing`), and various functors.
    *   **MPI:** For parallel communication (via `ekat::Comm` and HOMME's internal MPI setup).
*   **Interactions with other Components:**
    *   **`AtmosphereDriver`:** Manages the lifecycle of `HommeDynamics`.
    *   **FieldManager:** `HommeDynamics` registers its required/updated fields on the physics grid and internal fields on the dynamics grid.
    *   **GridsManager:** Obtains grid information and creates remappers.
    *   **Physics Parameterizations:** `HommeDynamics` receives updated temperature, horizontal winds, and tracer concentrations from physics parameterizations. It computes tendencies from these changes and provides them as forcing to HOMME. After HOMME runs, it provides the updated atmospheric state back to the physics grid.
    *   **IOP (Intensive Observation Period):** If IOP is active, `HommeDynamics` may apply IOP-specific forcing.
