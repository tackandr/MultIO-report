# Destination Earth MultIO Server Implementation — Phase 2 Report

## Table of Contents

1. [Background](#1-background)
2. [Status of MultIO Technical Development](#2-status-of-multio-technical-development)
   - 2.1 [Basic Design Principles](#21-basic-design-principles)
   - 2.2 [Coverage](#22-coverage)
   - 2.3 [Build Process](#23-build-process)
   - 2.4 [Code samples](#24-code-samples)
3. [Outlook for Phase 3](#3-outlook-for-phase-3)
   - 3.1 [Feeding Input Data through MultIO](#31-feeding-input-data-through-multio)
   - 3.2 [Implementing FA-Files under the MultIO Framework](#32-implementing-fa-files-under-the-multio-framework)
   - 3.3 [Implementing Custom Binary Formatted Restart Files](#33-implementing-custom-binary-formatted-restart-files)

---

## 1. Background

MultIO has been developed to serve as an I/O and in-memory post-processing layer for coupled Earth-system models. The primary motivation was the introduction of support for in-memory post-processing and GRIB2 output for ocean fields in the coupled IFS-NEMO model. This guarantees uniform GRIB2 output format across all components. However, this configuration requires each model to have their own dedicated I/O server, which introduces load-balancing challenges. The IFS Fortran I/O-server is also less flexible for supporting the computation of teomporal statistics. A goal of MultIO is to unify this under a single I/O-server layer and introduce that to limited-area model (LAM) configurations. The work described here represents progress towards it. Within the DEODE (Destination Earth On-Demand Extremes) framework, MultIO also provides the mechanism for integrating IAL model output with ECMWF data services, enabling direct writing of model results to the ECMWF Field DataBase (FDB). The primary drivers for MultIO are summarised below.

**Extensibility to additional output sources.**
The initial drive for MultIO was to support additional output sinks without duplicating or re-implementing I/O logic. The modular architecture of MultIO makes it straightforward to add new data sources and sinks.

**Single unified interface for multiple model components.**
The Earth-system modelling stack involves multiple components — atmosphere, ocean, land surface, sea ice, and others. MultIO provides a single, consistent I/O interface that all components can share, reducing integration complexity and maintenance burden.

**On-the-fly post-processing.**
MultIO supports in-situ post-processing operations including statistical aggregations over time horizons, horizontal interpolation, re-gridding, and re-projection through external libraries. This capability is particularly valuable within the DEODE framework, where the ability to produce derived products on the fly reduces downstream data volumes and latency.

**Integration with DEODE / IAL workflows.**
Within the DEODE framework, a key motivation for integrating MultIO into the Integrated Arome–ARPEGE–LAM (IAL) modelling workflow is to enable model output to be written directly to the ECMWF Field DataBase (FDB) through the MultIO pipeline. This capability supports the operational data distribution architecture used in the Destination Earth ecosystem and facilitates seamless access to simulation outputs through ECMWF data services.

---

## 2. Status of MultIO Technical Development

### 2.1 Basic Design Principles

The design of MultIO is guided by the need for modularity, portability, and asynchronicity, i.e., minimal coupling between the model code and the I/O backend. The key architectural concepts are described below and draw on the principles introduced in the MultIO paper <!-- TODO: insert full citation -->.

**Collaborative development.**
The development and integration work described here is carried out as a collaborative effort between the DEODE consortium and ECMWF teams, ensuring alignment with ECMWF operational infrastructure and with the evolving architecture of the Destination Earth platform.

**Communicator splitting.**
MultIO achieves parallel I/O by splitting the MPI communicator at model initialisation. Dedicated I/O server tasks are allocated from the global communicator and run concurrently with the compute tasks. Data is transferred asynchronously from compute ranks to I/O ranks, decoupling model integration time from I/O latency.

**Bidirectional workflow and transport.**
The recently updated architecture design supports both output (model → file/stream) and input (file/stream → model) data paths within the same framework. This symmetry ensures that the same configuration and pipeline machinery can be reused for read operations, avoiding code duplication and allowing a consistent treatment of I/O in the model lifecycle.

**Technical implementation in IFS.**

MultIO has been integrated into the NEMO component of IFS of ECMWF and is being intergrated into the atmosphere/wave components. The integration follows a client–server model in which the IFS(-NEMO) compute tasks act as MultIO clients, marshalling field data via the MultIO API, while dedicated server tasks handle additional post-processing tasks, GRIB2 encoding, and writing. The implementation replaces or wraps the legacy I/O calls, enabling a gradual migration path.

**Merge into the IAL repository.**
The MultIO codebase has been merged into the IAL repository, making it available to the wider community of NWP centres that build on the shared IAL model stack. This integration also enables the use of MultIO within the DEODE workflow, where writing output through MultIO directly to the ECMWF FDB is a key objective. This ensures that MultIO developments are available to all IAL users and that ongoing changes remain synchronised across the community.

### 2.2 Coverage

**GRIB2.**
The primary output format currently supported by MultIO is GRIB2, the WMO standard for gridded binary data exchange. Field encoding via ecCodes is integrated directly into the MultIO pipelines, allowing model output to be written in standards-compliant GRIB2 without additional post-processing steps.

### 2.3 Build Process

The previously CMake-based build process has been replaced by `ecbundle`, an ECMWF tool that provides a unified, bundle-based approach for managing and building the full software stack. `ecbundle` orchestrates the configuration and build of all required dependencies as a single bundle, superseding the manual CMake workflow. This aligns the IAL build process with that used for IFS at ECMWF, unifying the two workflows and thereby simplifying future co-developments between the IFS and IAL communities. Configuration details for building the IAL bundle — including dependency specifications and build instructions — are maintained in the [ial-bundle repository](https://github.com/destination-earth-digital-twins/ial-bundle) on GitHub.

### 2.4 Code samples

The MultIO must be initialized as follows in the file *arpifs/programs/master.F90* before calling the actual MPI initialization:
```fortran
CALL MULTIO_INITIALIZE()
```

Once the MPI initialization has taken place, MultIO communication split takes place:
```fortran
CALL INITIALIZE_LAYOUT( MERGE( MPLUSERCOMM, MPI_COMM_WORLD, LMPLUSERCOMM ) )
ICOMM_IFS_PLUS_IFS_IO_SERV = GET_COMM_BY_NAME( 'ifs-solver2ifs-io' )
LLIFS = (IS_IN_ROLE( 'ifs-solver' ) .OR. IS_IN_ROLE( 'ifs-io' ))
```

The MultIO client (on the compute tasks) gets started and stopped as follows:
```fortran
IF ( IS_IN_ROLE( 'ifs-solver' )) THEN
      CALL MULTIO_CLIENT_START( &
&      GET_COMM_BY_NAME( 'ifs-solver2multio-server' ), &
&      GET_COMM_BY_NAME( 'multio-server' ), &
&      GET_COMM_BY_NAME( 'ifs-solver' ) &
&     )
ENDIF

!! Forecasting code here
CALL CNT0(...)

IF ( IS_IN_ROLE( 'ifs-solver' ) ) THEN
      CALL MULTIO_CLIENT_STOP()
ENDIF
```

Similarly on the I/O-tasks MultIO server gets started and stopped as follows:
```fortran
CALL MULTIO_SERVER_START( &
     &    GET_COMM_BY_NAME( 'ifs-solver2multio-server' ), &
     &    GET_COMM_BY_NAME( 'multio-server' ), &
     &    GET_COMM_BY_NAME( 'ifs-solver' ) &
     &   )

CALL MULTIO_SERVER_STOP()
```

The MultIO epoch must be finalized at the end (preferably before finalizing MPI):
```fortran
CALL MULTIO_FINALIZE()
```

Writing data out using I/O-servers is implemented in the file *arpifs/control/allfpos.F90*.
It contains both Fortran I/O-server and MultIO server calls as MultIO currently only
supports GRIB2 files -- please observe that input parameters are kept the same for both implementations:
```fortran
IF (LLUSE_IOSERV) THEN
  CALL IOSERV_WRHFP(YDFPGEO,IFPDOM,YLRQGP%NFIELDG,CLWFP,YLFIODSC,ZFPBUF,YLFACTX,PTSTEP)
  IF ( LUSE_MULTIO ) THEN
    CALL MULTIO_WRHFP(YDFPGEO,IFPDOM,YLRQGP%NFIELDG,CLWFP,YLFIODSC,ZFPBUF,YLFACTX,PTSTEP)
  ENDIF
  ...
ENDIF
```

The Fortran I/O-server routine IOSERV_WRHFP in the file *arpifs/fullpos/ioserv_wrhfp.F90* does the following:
```fortran
ASSOCIATE(NFPRGPL=>YDFPGEO%NFPRGPL, NFPRGPG=>YDFPGEO%NFPRGPG)
  IO_SERV_C001%IDATEF = YDFACTX (1)%IDATEF
  ZTSTEP=0._JPRB
  IF (PRESENT (PTSTEP)) ZTSTEP = PTSTEP
  CALL IO_SERV_MAP_SEND_PART1(IO_SERV_C001, NFPRGPL, NFPRGPG, YDFLDSC, MTAG_MFIO_WRHFP+ICHAR(CDCONF), YLIOSMPP, &
  & KDOM_TYPE=NIO_SERV_HDR_IDOM_FPA,PIOPROC1=IO_SERV_C001%PIOPROC1_FLP,PIOPROC2=IO_SERV_C001%PIOPROC2_FLP,PTSTEP=ZTSTEP)
  CALL EXTFPSELECT (YDFPGEO,YDFLDSC,YLIOSMPP%YLBUFD, PFPBUF)
  CALL IO_SERV_MAP_SEND_PART2 (IO_SERV_C001, YLIOSMPP)
END ASSOCIATE
```

Whereas the MultIO server routine MULTIO_WRHFP in the file *arpifs/fullpos/multio_wrhfp.F90* is currently doing the following job:
```fortran
USE MULTIO_SERVER_MOD, ONLY: MIO_HANDLE_
USE MULTIO_API, ONLY: MULTIO_METADATA
USE MULTIO_SERVER_MOD, ONLY: CURRENT_STEP
USE MULTIO_SERVER_MOD, ONLY: CURRENT_COUNTER
TYPE(MULTIO_METADATA) :: MIOMD
CNT = CURRENT_COUNTER()
! Populate message with data                                                                                                                                                                                                                 DO I = 1, KNFG
  ! Populate the buffer                                                                                                                                                                                                                        FIELD_TMP = 0._JPRB
  CALL MESSAGE_FILL_DATA( YDFPGEO, PFPBUF, I, YDFPGEO%NFPROMA, YDFPGEO%NFPBLOCS, FIELD_TMP )
  IERR = MIOMD%NEW(MIO_HANDLE_)
  IERR = MIOMD%SET( 'misc-globalSize', YDFPGEO%NFPRGPG )
  IERR = MIOMD%SET( 'name', 'LAM_GP' )
  IERR = MIOMD%SET( 'domain', 'LAM_GP' )
  IERR = MIOMD%SET( 'representation', 'unstructured' )
  IERR = MIOMD%SET( 'grid', 'lambert_lam' )
  IERR = MIOMD%SET( 'param', 10000*CNT + I ) !YDFACTX(I)%YFAPARAM%INGRIB )
  IERR = MIOMD%SET( 'levelist', YDFLDSC(I)%ILEVG )
  IERR = MIOMD%SET( 'levtype', YDFLDSC(I)%CPREF )
  IERR = MIOMD%SET( 'suffix', YDFLDSC(I)%CSUFF )
  IERR = MIOMD%SET( 'is_spectral', YDFLDSC(I)%LSPEC )
  IERR = MIOMD%SET( 'timestep', PTSTEP )
  IERR = MIOMD%SET( 'step', CURRENT_STEP() )
  IERR = MIOMD%SET( 'toAllServers', .FALSE. )
  IERR = MIO_HANDLE_%WRITE_FIELD( MIOMD, FIELD_TMP )
  IERR = MIOMD%DELETE()
ENDDO
```

Please note that error handling code has been omitted for brevity.

---

## 3. Outlook for Phase 3

Phase 3 of the MultIO development will focus on three main areas: extending MultIO to handle model input in addition to output, adding support for FA-file formats, and enabling the handling of custom binary restart files.

### 3.1 Feeding Input Data through MultIO

A key objective for Phase 3 is to leverage the bidirectional architecture of MultIO for model *input*, enabling data to be fed into a running model through the same middleware layer used for output. Several high-value use cases have been identified.

**Lateral Boundary Conditions (LBC).**
Limited-Area Models (LAMs) require periodic injection of Lateral Boundary Conditions derived from a driving global model. Reading LBC data through MultIO would allow the same pipeline parallelism and buffering mechanisms used for output to be applied to input, potentially reducing the time spent waiting for boundary data and enabling more efficient coupling between global and regional models.

**Accelerating initial input file reads.**
The start-up phase of a model run can be a significant bottleneck due to the serial or poorly parallelised reading of large initial condition files. By feeding full two-dimensional fields ("levels") efficiently through MultIO's parallel I/O infrastructure, the start-up cost can be reduced, improving overall time-to-solution.

**Nudging forecasts with AI-model fields.**
An emerging use case is the periodic injection of fields from AI-based weather prediction models into a running physics-based forecast in order to nudge or constrain the solution. MultIO's input pathway provides a natural mechanism for this, allowing AI-model output to be ingested at runtime without bespoke coupling code.

### 3.2 Implementing FA-Files under the MultIO Framework

FA (Fichier ARPEGE) is the proprietary binary format used by the ARPEGE/ALADIN/AROME family of NWP models. Phase 3 plans to implement full FA-file support within MultIO, covering both output and input paths.

**Output.**

- *Wrapping the existing FA encoding:* The existing Fortran FA encoding library will be wrapped within the MultIO pipeline, allowing FA-format output to be produced through the standard MultIO API without modifying existing encoder logic.
- *GRIB2 output of FA fields:* In addition to the native FA format, it will be possible to produce GRIB2 output for the same fields, facilitating interoperability with data consumers that do not support FA. EcCodes is now ready to handle the encoding of FA formatted data. A partial set of translations from the FA name space already exists - for example, SFX.TG1P1 (temperature in the ground, level 1, patch 1) and further FA variables are defined in the ecCodes local-concepts definitions. Where the FA name is available, the translation logic used in facgrm.F90 can be applied together with the existing ecCodes definitions to map in-memory FA variables to their corresponding GRIB2 representations.
- *Non-standard WMO parameters:* FA fields may carry parameters that are not part of the standard WMO GRIB2 parameter database. These will be supported through the use of experimental parameter IDs defined in YAML configuration files. This mechanism is designed to be reusable across other model components and output formats.

**Input.**
The input path will allow FA-format files to be read back through MultIO, enabling restart and initialisation workflows that currently depend on direct FA file reads to benefit from the parallelism and abstraction provided by the MultIO layer.

### 3.3 Implementing Custom Binary Formatted Restart Files

Many model components write and read restart files in custom binary formats, often with a one-to-one correspondence between MPI tasks and restart file(s). Phase 3 will introduce support for handling such files within MultIO.

**MPI-topology-aware redistribution.**
A particular challenge with restart files is that the MPI decomposition used when writing a restart may differ from the decomposition used when reading it back (e.g., due to a change in the number of compute nodes). MultIO will implement the necessary redistribution logic to map data from one MPI topology to another, making restart files portable across different parallel configurations.

**Output (writing restart files).**
The MultIO output path will support writing custom binary restart files, with per-task or aggregated file layouts configurable through the MultIO YAML pipeline description.

**Input (reading restart files).**
Complementarily, the input path will support reading these restart files and distributing the data to the appropriate MPI ranks in the current run configuration, even when the topology differs from the one used at write time.
