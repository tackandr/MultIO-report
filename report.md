# MultIO Development within Destination Earth – Draft Report

## Background

The existing **Fortran I/O server** (hereafter *legacy I/O server*) has served as the
primary output mechanism for ECMWF's IFS model for many years.  However, its monolithic
design makes it difficult to support new output destinations or to extend its functionality
without significant re-engineering.  **MultIO** is a new, C++-based I/O multiplexing server
designed to replace the legacy Fortran I/O server while providing a cleaner, more extensible
foundation.  Several motivations drove this decision:

- **Broader output-source support.** The legacy Fortran I/O server is tightly coupled to a
  single output path. MultIO makes it straightforward to add further output sources, such as
  NEMO, without invasive changes to the core server code.
- **Potential extension to data input.** Beyond writing model output, MultIO is designed so
  that the same multiplexing framework could eventually be applied to reading input data,
  opening the door to a unified I/O layer – something the legacy Fortran I/O server was never
  designed to support.
- **Complementing and extending I/O-server development.** MultIO is intended to replace the
  legacy Fortran I/O server incrementally, sitting alongside it during transition periods,
  complementing its capabilities and providing an extension point for future work.

---

## Status of MultIO Technical Development

### Basic Design Principles

MultIO follows a pipeline-based architecture in which data passes through a configurable
chain of *actions* before reaching one or more output sinks. Key design goals are:

- **Modularity** – each action is a self-contained unit that can be enabled, disabled or
  reordered without touching the rest of the pipeline.
- **Configurability** – the pipeline is driven by a human-readable configuration file,
  keeping run-time behaviour separate from compiled code.
- **Extensibility** – new output formats or data-processing steps can be added by
  implementing a small, well-defined interface.

### Coverage

| Format / Feature | Status |
|---|---|
| GRIB2 | Supported |
| FA-files | Postponed to phase-3 |

#### GRIB2

Encoding and writing of GRIB2 fields is fully supported in the current phase. The
implementation relies on ecCodes for encoding, which ensures compatibility with the
standards used throughout ECMWF and DestinE workflows.

#### FA-files

Support for FA-files (Fichier d'Analyse, used by the ARPEGE/ALADIN model family) has been
scoped and partially designed but is deliberately deferred to phase-3 to keep the phase-2
scope manageable.

### Build Process

MultIO is built with CMake and integrates with the standard ecbuild toolchain used across
ECMWF software. Dependencies are managed through bundle builds or pre-installed modules.
Continuous integration runs the test suite on every commit to the main branch, covering unit
tests for individual actions as well as integration tests that exercise the full pipeline.

---

## Outlook for Phase-3

### Feeding Input Data through MultIO

Extending MultIO to handle model *input* as well as output is one of the main themes for
phase-3. Several concrete use cases have been identified:

- **Lateral Boundary Conditions (LBC).** LBC data are used to drive Limited-Area Model
  (LAM) runs. Channelling LBC provision through MultIO would allow the same pipeline
  machinery (logging, filtering, format conversion) to be applied consistently to input
  fields.
- **Speeding up initial input file reads.** By feeding complete 2-D horizontal slices
  ("levels") efficiently into the model at start-up, the wall-clock cost of reading initial
  conditions can be reduced significantly.
- **Restart files across different MPI topologies.** Present restart workflows require that
  the MPI decomposition used when writing a restart matches the decomposition used when
  reading it. A MultIO-based input layer would allow restart files written with one
  MPI-task layout to be read back under a different layout, removing a significant
  operational constraint.
- **Nudging with AI-model fields.** Periodic injection of fields from an AI-based model
  into a running numerical forecast (nudging) is an emerging use case in DestinE. MultIO
  can provide a clean interface for ingesting these asynchronous input streams.

### Implementing FA-file Support under the MultIO Framework

Full FA-file integration is planned for phase-3, covering both directions:

- **Output** – writing FA-format fields via a dedicated MultIO sink action, enabling
  ARPEGE/ALADIN-family models to use MultIO without any loss of native-format output.
- **Input** – reading FA-format files through a MultIO source action, consistent with the
  broader input-handling work described above.

### Implementing Restart-file Handling

Robust restart support is a prerequisite for production use of MultIO in ensemble and
high-resolution configurations:

- **Input** – reading binary restart files (one or more files per MPI task) and
  distributing the data correctly regardless of the MPI topology mismatch between write
  and read time.
- **Output** – writing restart files through MultIO, giving operators a single, unified
  mechanism for all model I/O and enabling features such as on-the-fly compression or
  routing of restarts to different storage tiers.
