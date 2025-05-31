# atmosphere_driver.cpp

## Overview

This file implements the main driver for the EAMxx atmospheric model. It is responsible for orchestrating the overall simulation lifecycle, including initialization, time stepping, and finalization. It manages the interactions between various atmospheric components such as dynamics, physics parameterizations, and I/O.

## Key Components

### Classes

*   **`AtmosphereDriver`:** The central class that manages the atmospheric simulation.
    *   **`AtmosphereDriver(const ekat::Comm& atm_comm, const ekat::ParameterList& params)`:** Constructor. Initializes the driver with the communicator and parameters.
    *   **`~AtmosphereDriver()`:** Destructor. Handles cleanup and finalization.
    *   **`set_comm(const ekat::Comm& atm_comm)`:** Sets the MPI communicator for the driver.
    *   **`set_params(const ekat::ParameterList& atm_params)`:** Sets the configuration parameters for the simulation.
    *   **`init_scorpio(const int atm_id)`:** Initializes the Scorpio I/O library.
    *   **`init_time_stamps (const util::TimeStamp& run_t0, const util::TimeStamp& case_t0, int run_type)`:** Initializes the run and case start timestamps.
    *   **`setup_iop()`:** Sets up the Intensive Observation Period (IOP) if enabled.
    *   **`create_atm_processes()`:** Creates all registered atmospheric processes.
    *   **`create_grids()`:** Creates the simulation grids based on requirements from atmospheric processes.
    *   **`setup_surface_coupling_data_manager(...)`:** Sets up the data manager for surface coupling.
    *   **`setup_surface_coupling_processes()`:** Configures surface coupling importer/exporter processes.
    *   **`reset_accumulated_fields()`:** Resets fields that accumulate values over time.
    *   **`setup_column_conservation_checks()`:** Sets up checks for mass and energy conservation.
    *   **`add_additional_column_data_to_property_checks()`:** Adds specified fields to property checks for debugging.
    *   **`create_fields()`:** Creates and allocates all necessary simulation fields.
    *   **`create_output_managers()`:** Creates managers for handling simulation output.
    *   **`initialize_output_managers()`:** Initializes the output managers with fields and grids.
    *   **`set_provenance_data(...)`:** Sets provenance information (case name, user, version, etc.).
    *   **`initialize_fields()`:** Initializes field values, either from initial conditions or a restart file.
    *   **`restart_model()`:** Reads all necessary data from a restart file.
    *   **`set_initial_conditions()`:** Sets initial conditions for fields based on various mechanisms (constants, file input, copy).
    *   **`initialize_atm_procs()`:** Initializes all atmospheric processes.
    *   **`initialize(...)`:** High-level method to orchestrate the entire initialization sequence.
    *   **`run(const int dt)`:** Executes a single atmospheric time step of duration `dt`.
    *   **`finalize()`:** Finalizes the simulation, cleans up resources, and closes I/O.
    *   **`get_field_mgr(const std::string& grid_name) const`:** Retrieves the field manager for a given grid.

## Important Variables/Constants

*   **`m_atm_comm` (ekat::Comm):** The MPI communicator used by the atmosphere driver.
*   **`m_atm_params` (ekat::ParameterList):** Holds the configuration parameters for the current simulation.
*   **`m_run_t0` (util::TimeStamp):** Timestamp for the start of the current run.
*   **`m_current_ts` (util::TimeStamp):** The current timestamp of the simulation.
*   **`m_case_t0` (util::TimeStamp):** Timestamp for the absolute start of the case (could be different from `m_run_t0` in restarted runs).
*   **`m_run_type` (RunType):** Enum indicating if the run is Initial or Restart.
*   **`m_atm_process_group` (std::shared_ptr<AtmosphereProcessGroup>):** Manages the collection of all atmospheric processes.
*   **`m_grids_manager` (std::shared_ptr<GridsManager>):** Manages all simulation grids.
*   **`m_field_mgrs` (std::map<std::string, field_mgr_ptr>):** A map storing field managers for each grid.
*   **`m_output_managers` (std::vector<OutputManager>):** A list of managers for handling various output streams.
*   **`m_restart_output_manager` (std::shared_ptr<OutputManager>):** Specific output manager for restart files.
*   **`m_iop` (std::shared_ptr<IntensiveObservationPeriod>):** Handles data and logic for Intensive Observation Periods.
*   **`m_memory_buffer` (std::shared_ptr<ATMBufferManager>):** Manages memory buffers for atmospheric processes.
*   **`s_comm_set`, `s_params_set`, etc. (int constants):** Bit flags to track the initialization status of different driver parts.

## Usage Examples

*(This file defines the main driver, so direct usage examples are less about calling its individual functions and more about how it's invoked by the E3SM framework. For a developer extending EAMxx, understanding the initialization sequence and the `run` loop is key.)*

## Dependencies and Interactions

*   **Internal Dependencies:**
    *   `control/atmosphere_surface_coupling_importer.hpp`: For importing surface coupling data.
    *   `control/atmosphere_surface_coupling_exporter.hpp`: For exporting surface coupling data.
    *   `physics/share/physics_constants.hpp`: For physical constants.
    *   `share/atm_process/atmosphere_process_group.hpp`: Core dependency for managing collections of atmospheric model components.
    *   `share/atm_process/atmosphere_process_dag.hpp`: For managing the Directed Acyclic Graph of processes.
    *   `share/field/field_utils.hpp`: Utilities for working with fields.
    *   `share/util/scream_time_stamp.hpp`: For handling timestamps.
    *   `share/util/scream_timing.hpp`: For timing simulation parts.
    *   `share/io/scream_io_utils.hpp`: Utilities for I/O operations.
    *   `share/property_checks/mass_and_energy_column_conservation_check.hpp`: For setting up conservation checks.
    *   `ekat/ekat_parameter_list.hpp`: Used extensively for managing configuration parameters.
    *   `ekat/ekat_parse_yaml_file.hpp`: For parsing YAML configuration files.
    *   `ekat/ekat_comm.hpp`: For MPI communication.
    *   `scorpio_input.hpp` / `scorpio_output.hpp` (via `scream_io_utils.hpp` or `OutputManager`): For file I/O using the Scorpio library.
*   **External Libraries:**
    *   **MPI:** For parallel communication (via `ekat::Comm`).
    *   **NetCDF/PIO (Scorpio):** For reading initial conditions, restart files, and writing output.
    *   **GPTL:** For performance timing (conditionally compiled).
*   **Interactions with other Components:**
    *   **Atmospheric Processes (various types):** The driver creates, initializes, runs, and finalizes all registered atmospheric processes (dynamics, physics parameterizations, diagnostics, etc.). It manages data flow between them via FieldManagers.
    *   **GridsManager:** The driver instructs the GridsManager to build necessary grids and then provides these grids to the atmospheric processes.
    *   **FieldManager:** For each grid, a FieldManager is created to manage the allocation and access of all simulation variables (fields) on that grid.
    *   **OutputManager:** The driver initializes and calls OutputManagers to write data to files at specified frequencies.
    *   **Surface Coupler:** Interacts with surface coupling components to exchange data between the atmosphere and other earth system model components (e.g., land, ocean, ice).

*(Remove any subsection if not applicable)*
