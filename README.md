# SGK (Space Generative Kernel)

SGK is a personal open-source project for propulsion analysis and design. The intended workflow is to define an engine case, generate its geometry, evaluate it and compare design changes through a straightforward interface. Cases and results should remain easy to inspect and reuse.

The central objective is to bypass expensive simulations during most design iterations through fast model inference. Reference analyses (CFD, FEM or plasma simulations, depending on the problem) provide training data and checks on selected configurations. The software connects preliminary physics, geometry generation, learned models and design search.

This document defines the pre-design. The repository currently contains the specifications and license; the SGK application is still to be implemented. **Adopted** below means a design choice for that implementation. **Open** identifies a decision that requires a defined physical case or an experiment.

## Scope

SGK targets all propulsion families listed below. Liquid chemical propulsion is the starting point for implementation; the architecture and overall scope cover chemical, electric and nuclear propulsion.

| Stage | Scope |
|---|---|
| First application | Liquid chemical propulsion, with parameterized geometry and one bounded physical analysis |
| Extension of the liquid baseline | Cooling, structural response, coupled fields and changing operating conditions |
| Overall scope | All listed propulsion families, each with its own physical case and evaluation path |
| Product target | A usable design workflow with inspectable geometry, physical results, comparisons and exportable cases |

The first liquid case is open. A nozzle-flow problem or cooling-channel study is a candidate. Its parameters, equations, reference solver and target outputs must be defined before producing a propulsion dataset.

Manufacturing constraints are part of the intended geometry design. For metal additive manufacturing (L-PBF), this includes wall and channel dimensions, build orientation and access for support or powder removal. Process-specific limits and export requirements remain to be selected.

## Architecture

| Layer | Responsibility | Input | Output |
|---|---|---|---|
| 1. Preliminary physics and geometry | Resolve sizing targets and generate a parameterized geometry | Physical case, design parameters and operating conditions | Geometry, named domains and boundaries, derived physical quantities |
| 2. Fast physical evaluation | Predict the response of a supported case | Geometry representation, physical conditions and trained model | Fields or engineering quantities, model identity and applicability information |
| 3. Design search | Propose parameter changes and evaluate objectives and constraints | Parameter bounds, objective definitions and evaluation results | Candidate history, comparisons and reference-checked designs |
| 4. User interface | Configure, inspect and run the numerical workflow | User edits or structured requests | Case changes, geometry views and result comparisons |

Reference solvers generate training data and verify selected designs; trained surrogates provide the fast evaluations used during design search.

### Tools and selection status

| Component | Choice | Status and purpose |
|---|---|---|
| Geometry core | C#/.NET and [PicoGK](https://github.com/leap71/PicoGK) | Adopted for parameterized voxel geometry |
| Model environment | Python and PyTorch | Adopted for training and inference |
| Fluid reference solver | OpenFOAM | Candidate for the first fluid case; solver and version depend on the selected equations |
| Operator architecture | GNO/FNO composition, with [GINO](https://arxiv.org/html/2309.00583v1) as a reference | Candidate for variable geometry and field prediction |
| Physical training constraints | Equation and boundary residuals, informed by [PINO](https://arxiv.org/abs/2111.03794) | Experiment to compare with supervised training |
| Model framework | [NVIDIA PhysicsNeMo](https://github.com/NVIDIA/physicsnemo) | Candidate source of operator implementations and training components |
| Sparse voxel processing | [fVDB](https://github.com/openvdb/fvdb-core) | Optional, subject to the geometry-to-model conversion requirements |
| Geometry helpers | LEAP71 ShapeKernel and LatticeLibrary | Optional for operations that justify their use |
| Search algorithm | Parameter sweep first; optimization method open | Establish a baseline before selecting Bayesian or population-based search |
| Interface | File-based/command-line access, then graphical and conversational controls | Adopted order; concrete interface technologies remain open |

## Pipelines

### Reference data and model training

```text
physical case + parameters + operating conditions
    -> preliminary sizing
    -> geometry + named domains and boundaries
    -> solver mesh + numerical configuration
    -> reference analysis
    -> checked fields and engineering quantities
    -> geometry/field sampling + normalization
    -> training dataset
    -> trained model + preprocessing + evaluation report
```

The reference-analysis path handles mesh generation, boundary-condition assignment, solver execution and result extraction. Only accepted numerical results enter the training targets; failed or unconverged runs retain their diagnostics.

Each training sample links a geometry representation and operating conditions to reference outputs at identified locations. For a field target, the sample retains the query coordinates and corresponding values; for an integrated quantity, it retains the integration definition and units.

### Design iteration

```text
candidate parameters
    -> sizing and geometry generation
    -> applicability checks
    -> fast inference
    -> objectives and constraints
    -> candidate comparison
    -> next parameter proposal
```

Selected candidates return to the reference solver for verification. Cases outside the supported domain are rejected or scheduled for reference analysis. New reference data can support a later training cycle; switching models requires re-evaluating candidates used in a common comparison.

The initial search is **parametric**: it explores shapes expressible by the geometry generator and its permitted parameters. Free-form topology generation is outside the current pre-design.

## Data exchanged between components

The following names identify logical records. Exact field names, types and file formats remain open.

| Record | Contents | Role |
|---|---|---|
| Case (`EngineConfig`) | Geometry parameters, sizing targets, operating conditions, equations/closures, properties, boundary conditions and assumptions | Complete definition of the physical problem |
| Geometry | Case identity, voxel/surface representation, dimensions, coordinate frame, resolution, solid/fluid domains and boundary labels | Shared geometric description for meshing, sampling and viewing |
| Reference run | Case and geometry identity, solver configuration, fields, integrated quantities, diagnostics and numerical checks | Evidence produced by a reference analysis |
| Trained model | Architecture/configuration, weights, preprocessing, normalization, input/output definitions, data revision and evaluation report | Reusable predictor with a declared domain of use |
| Evaluation | Link to a reference run or model inference, engineering quantities, applicability and constraint results | Common summary consumed by the search and comparison interface |

Large fields remain in the referenced run or inference artifacts. The evaluation record contains the quantities needed for comparison and links to those artifacts, avoiding a second copy of the physical result.

**Exchange conventions**

- SI units for physical exchange; explicit conversion at geometry-library boundaries, including millimetres in PicoGK.
- Defined coordinate frames, vector components and surface-normal orientation.
- A new case revision when geometry, conditions, properties or the physical formulation changes.
- Separate identities for cases, reference runs, datasets and trained models. Storage and revision mechanisms remain open.

## Layer 1: preliminary physics and geometry

### Input modes

| Mode | Definition | Status |
|---|---|---|
| Engine-driven | Independent physical/design parameters plus a sizing target; remaining quantities derived from the selected model | Initial mode |
| Mission-driven | Mission requirements translated into candidate engine cases through assumptions, objectives and constraints | Later extension |

For a liquid case, inputs may include propellant combination, chamber pressure, mixture ratio and expansion ratio. Requested thrust or specific impulse must be distinguished from the performance calculated by the sizing model.

Mission-driven sizing must resolve additional choices such as vehicle mass, available power and operating environment. It produces explicit engine cases for the same downstream pipeline.

### Geometry processing

Voxel geometry is intended to support repeated boolean operations and complex internal passages (e.g. a regenerative cooling network formed by subtracting parameterized channel volumes from the chamber wall). Voxel resolution determines which channel and wall features remain resolved, as well as the memory and computation required.

1. Resolve the independent inputs and derived quantities using the case's preliminary physical models.
2. Generate voxel geometry at a specified resolution.
3. Identify solid regions, fluid passages and their interfaces.
4. Check dimensions, connectivity and the applicable manufacturing constraints.
5. Export geometry and domain labels for meshing, model preparation and inspection.

Preliminary models may use analytical relations, numerical integration, property tables or equilibrium calculations. Their assumptions and valid ranges belong to the selected case.

Voxel storage, surface extraction and solver meshing are distinct representations. The conversion path must preserve units, domain identity and relevant geometric features. VDB is the intended voxel storage format; surface and solver-specific formats remain open.

Geometry checks cover known-shape dimensions and volumes, channel connectivity, minimum wall thickness and sensitivity to voxel resolution. Manufacturing-oriented exports additionally require the selected process limits and build constraints.

## Layer 2: fast physical evaluation

### Prediction target

The first case must choose between a scalar/vector engineering output (such as pressure drop) and a spatial field (such as pressure or velocity). This choice determines the required representation and architecture.

| Target | Data required | Model consideration |
|---|---|---|
| Engineering quantities | Case parameters and reference quantities | Compare a small parameter-to-output predictor with operator methods |
| Surface fields | Geometry samples, conditions and fields at identified surface locations | GNO/FNO composition is a candidate for variable geometry |
| Volume fields | Interior sampling/domain representation, conditions and reference fields | Requires an explicit volume-output formulation |
| Coupled or transient response | Coupled variables, interface conditions, or time and initial conditions | Later extension of the first steady problem |

Geometry and operating conditions are both inputs when conditions vary. Preprocessing and normalization are saved with the trained model and reused during inference.

### GNO/FNO path

The [GINO reference architecture](https://arxiv.org/html/2309.00583v1) connects irregular geometry samples to a regular latent grid and decodes the result at query locations. Encoding and decoding refer to learned feature transformations; the decoder predicts a physical field at those locations.

```text
PicoGK geometry
    -> surface samples + signed distance function (SDF)
    -> GNO encoder: irregular samples -> latent grid features
    -> concatenate SDF values on the latent grid
    -> FNO blocks: global processing on the regular grid
    -> GNO decoder: latent grid -> query-point features
    -> output projection -> predicted physical field
```

| Stage | Operation |
|---|---|
| Geometry encoding | For each latent-grid location, find geometry samples within a specified radius. A learned kernel aggregates their local information into feature channels on the grid. |
| Latent processing | Combine encoded geometry with SDF values. FNO blocks use Fourier transforms and learned spectral operations to exchange information across the grid. |
| Field decoding | For each query point, aggregate neighboring latent-grid features through a graph operator, then project the resulting features to the requested physical variables. |

The published examples predict surface pressure. SGK's output variables, operating-condition inputs and any volume-field extension remain case-specific choices.

**SGK data preparation (proposed):**

| Data | Representation and use |
|---|---|
| Geometry samples | Coordinates of surface points in a common frame; sampling must preserve relevant boundaries and small features |
| Latent grid | Regular spatial coordinates with chosen extent and resolution; stores encoded feature channels |
| Distance values | Signed distances evaluated at grid locations, with an explicit sign convention and scale |
| Operating conditions | Case variables and boundary-condition information; their injection into the network is still to be defined |
| Query locations | Coordinates where the field is requested, paired with reference values during supervised training |
| Outputs | Named physical variables at the query locations, converted back to physical units |

Encoder, latent blocks and decoder are trained together against the reference outputs. Inference uses the saved weights and preprocessing with new geometry and conditions, without requiring reference values at the query points.

The implementation sequence is to check geometry/SDF extraction, construct the encoder and decoder neighborhoods, verify tensor shapes and coordinate mappings, then train a small supervised case before adding physical residuals. Open choices include sampling density, grid resolution, neighborhood radii, feature widths, conditioning and loss normalization. Whether extraction needs fVDB remains open.

### Physics-informed training

The proposed experiment adds governing-equation and boundary-condition residuals to a supervised predictor. It requires:

- the equations, variables and boundary conditions of the physical case;
- a derivative/residual evaluation method compatible with the representation;
- scaling and weighting of data and physical losses;
- checks of residual evaluation against known fields.

Compare supervised training with the added physical losses on the same data splits. Measure prediction errors, conservation behavior, data requirements and training cost. Loss formulation for discontinuities or coupled physics remains a separate decision.

### Dataset and evaluation

| Item | Definition |
|---|---|
| Parameter space | Geometry and operating ranges, fixed assumptions and admissibility conditions |
| Initial sampling | A small controlled design of experiments; Latin hypercube sampling is one candidate |
| Dataset growth | Guided by pilot results, learning curves and errors relevant to design decisions |
| Splits | Group by physical case; keep points, time steps and resolutions from a case together when evaluating new-case generalization |
| Model selection | Validation split for tuning; final test split reserved for the fixed evaluation protocol |
| Metrics | Field errors and engineering-output errors, including local extrema or constraint-critical regions |

Reference quality is assessed through discretization sensitivity, conservation and analytical or independent comparisons where available. Numerical verification and comparison with experimental measurements must be reported separately.

Dataset size, acceptance tolerances and the available compute budget are open. Quantitative results from unrelated geometry classes do not set those values for SGK.

### Applicability and inference

An inference request identifies the case, model revision and requested outputs. Its result includes predictions and the outcome of the declared applicability checks.

Input compatibility, applicability and prediction accuracy are separate properties. Initial checks can cover schema, parameter bounds and supported geometry classes. Recognition of unfamiliar topology or physical regimes, uncertainty estimation and the fallback policy remain open design decisions.

Measure inference cost together with input preparation and data transfer. Report the model's test errors and applicable domain alongside timing results.

## Layer 3: design search

**Search definition:** adjustable parameters, bounds, objective quantities, constraints, initial candidates and an evaluation budget.

1. Propose a parameter change and regenerate the geometry.
2. Evaluate through the surrogate when the applicability policy permits it.
3. Calculate objectives and constraints separately.
4. Select candidates for reference checks or further exploration.
5. Compare reference results with predictions and retain the candidate history.

A small sweep or random search provides the first baseline. The optimization method is selected from the dimension and structure of the parameter space and the evaluation cost.

**Examples of possible design studies (e.g., for liquid propulsion):**

| Study | Adjustable geometry | Fixed conditions and comparison |
|---|---|---|
| Regenerative cooling passage | Channel width and depth along the axial coordinate, rib width and hot-wall thickness | Specify gas-side thermal loading, coolant inlet state and total flow. Compare coolant pressure drop and wall mass subject to peak wall-temperature and stress limits. |
| Nozzle contour | Contour control points between a fixed throat and exit | Hold throat area, exit area, chamber state and ambient pressure fixed. Compare axial thrust and nozzle length, with flow and thermal constraints defined for the operating case. |

A multi-objective search can return a set of trade-offs (e.g. cooling designs for which reducing pressure drop requires greater wall mass), with feasibility checked by reference analysis. The selected physical case determines which of these quantities can actually be evaluated.

Reference-check selection and stopping criteria remain open. A search may return no feasible design or no improvement; these are valid outcomes to distinguish from numerical failure. Report feasibility and improvement using reference results for the selected candidates.

Computational benefit is measured at comparable design quality. Separate initial dataset/training cost from subsequent searches using an existing model, and estimate the number of evaluations needed to recover that initial cost.

## Layer 4: interface

The interface exposes the same operations used by the numerical workflow:

| Operation | Information shown |
|---|---|
| Define/edit a case | Inputs, units, assumptions and derived quantities |
| Generate/inspect geometry | Shape, named domains, dimensions and geometric checks |
| Run an evaluation | Selected solver or model, applicability and results |
| Compare candidates | Changed parameters, fields, objectives, constraints and reference/model provenance |
| Run a search | Parameter space, budget, progress and selected candidates |

Initial access is through documented files and commands. A graphical interface adds geometry and field inspection. Conversational control is a later input method: an LLM proposes structured case edits through the same execution path, with assumed values visible before a run.

Concrete UI technology, command syntax and visualization/export formats remain open.

## Propulsion families

The intended scope includes the following applications. This list defines coverage; it does not prescribe a class hierarchy or a common physical model.

| Category | Application |
|---|---|
| Chemical | Liquid bipropellant (pressure-fed or pump-fed) |
| Chemical | Liquid monopropellant |
| Chemical | Solid |
| Chemical | Hybrid |
| Electric | Hall-effect thruster |
| Electric | Gridded ion thruster |
| Electric | Magnetoplasmadynamic thruster (MPD) |
| Electric | Microwave electrothermal thruster |
| Nuclear | Nuclear thermal propulsion (NTP) |
| Nuclear | Nuclear electric propulsion (NEP) |
| Nuclear | Fusion propulsion |

Case handling, geometry operations and result comparison are intended to be reusable across applications; physical formulations, reference analyses and trained models are defined per case.

## Development sequence

| Step                            | Deliverable                                                                                       |
| ------------------------------- | ------------------------------------------------------------------------------------------------- |
| 1. Controlled numerical example | Parameterized geometry, explicit domains, analytical reference and reproducible solver comparison |
| 2. Data and inference path      | Small dataset, baseline predictor and comparison with the proposed operator approach              |
| 3. Search loop                  | Parametric search with selected reference checks and recorded cost/accuracy                       |
| 4. First propulsion application | One liquid-propulsion problem with defined inputs, outputs and acceptance criteria                |
| 5. Broader capability           | Coupled analyses, interface improvements and additional physical applications                     |

The controlled example establishes the data and execution path. The propulsion application must demonstrate whether the workflow is useful on a problem with expensive reference analysis.

## Open decisions

| Decision | Needed before |
|---|---|
| First propulsion case: parameters, ranges, equations, properties and boundary conditions | Propulsion geometry and dataset implementation |
| Target output: engineering quantities, surface fields or volume fields | Model architecture selection |
| Geometry conversion: sampling, distance representation and solver formats | Dataset generation |
| Exchange schemas and C#/Python integration | Connecting the geometry and model components |
| Reference solver configuration and numerical checks | Accepting reference data |
| Error metrics, tolerances and constraints | Model selection and design comparison |
| Applicability, uncertainty and fallback policy | Automated surrogate-driven search |
| Search algorithm, reference-check selection and stopping rule | Optimization experiments |
| Dataset size, supported hardware and runtime budget | Scaling beyond pilot experiments |
| UI commands, visualization and manufacturing exports | Usable application release |

## License

Apache 2.0. See [LICENSE](LICENSE).

[SGK Mini](https://github.com/matteomafrici/sgk-mini) is a companion experiment for testing the proposed toolchain on a small controlled case.
